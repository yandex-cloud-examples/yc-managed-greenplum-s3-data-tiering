# Охлаждение данных в Object Storage для Managed Greenplum

# Вводная часть

Многие современные СУБД и движки для аналитической обработки данных предлагают подход с дезагрегированными слоями хранения (storage) и вычислений (compute). Данный подход декларирует возможность даже в случае серьезных аварий вплоть до полной потери ресурсов оперативно восстановить работоспособность системы засчет реализации слоя хранения как геораспределенного отказоустойчивого файлового хранилища (S3) и compute-слоя как эластичное приложение в разных датацентрах (зонах доступности).

В Greenplum применяется классический подход с локальным размещением данных прямо на сегментных хостах, имеющий ряд недостатков - например, поломка одного диска может потребовать "переналивки" (пересоздания) сегмента или даже целого сегментного хоста. Также в Yandex Cloud дополнительным ограничением является лимитированный размер диска в виду специфики инфраструктурной составляющей облачной платформы, что в свою очередь в некоторых случаях может приводить к необходимости расширить кластер именно из-за недостатка дисковых ресурсов даже при удовлетворяющей производительности CPU и RAM.

В качестве возможного решения указанных выше проблем командой разработки Yandex Cloud было создано расширение под названием Yezzey[^1] для организации гибридного хранения в Greenplum. Расширение позволяет переносить AO- и AOCS-таблицы Greenplum в объектное хранилище и обратно целиком или отдельными партициями без существенной потери производительности чтения. С точки зрения конечного потребителя данных процесс переноса данных в объектное хранилище не влияет на способ чтения объектов и не требует создания дополнительных оберток в виде view и самописных процедур переноса, являющихся стандартной практикой при организации "температурного" хранения в Greenplum.

# Работа с модулем Yezzey в Managed Greenplum
Базовые операции описаны в документации Yandex Cloud[^2]. Ниже мы рассмотрим некоторые особенности функционирования модуля Yezzey, в том числе при работе с партицированными таблицами.

## Выгрузка данных в Object Storage

Для демонстрации механики работы расширения Yezzey создадим 3 таблицы - 2 с партициями (с other-партицией и без) и 1 без партиций:
```sql

/* создадим 3 таблицы */
create table parted_table_full_yezzey
(
    id int,
    abc varchar(10),
    dt date
)
with
    (
        appendoptimized=true,
        orientation=column
    )
distributed randomly
partition by range(dt) 
(
	start (date '2023-01-01') inclusive
	end (date '2023-04-01') exclusive
	every (interval '1 month')
    /* НЕ УКАЗЫВАЕМ OTHER-ПАРТИЦИЮ */
);

create table parted_table_fract_yezzey
(
    id int,
    abc varchar(10),
    dt date
)
with
    (
        appendoptimized=true,
        orientation=column
    )
distributed randomly
partition by range(dt) 
(
	start (date '2023-01-01') inclusive
	end (date '2023-04-01') exclusive
	every (interval '1 month'),
    /* УКАЗЫВАЕМ OTHER-ПАРТИЦИЮ */
	default partition other
);

create table non_parted_table_yezzey
(
    id int,
    abc varchar(10),
    dt date
)
with
    (
        appendoptimized=true,
        orientation=column
    )
distributed randomly;


/* и вставим в них по 1 млн строк */
insert into parted_table_full_yezzey (id, abc, dt)
select i as id, cast(i as varchar) as abc, '2023-01-01'::date + interval '1' day*round(89*random()) as dt
from generate_series(1,1000000) i;

insert into parted_table_fract_yezzey (id, abc, dt)
select i as id, cast(i as varchar) as abc, '2023-01-01'::date + interval '1' day*round(89*random()) as dt
from generate_series(1,1000000) i;

insert into non_parted_table_yezzey (id, abc, dt)
select i as id, cast(i as varchar) as abc, '2023-01-01'::date + interval '1' day*round(89*random()) as dt
from generate_series(1,1000000) i;

analyze parted_table_full_yezzey;
analyze parted_table_fract_yezzey;
analyze non_parted_table_yezzey;
```

Теперь перенесем данные таблиц в гибридное хранилище. Для `parted_table_fract_yezzey` перенесем партиции явно, для остальных - вызовем функцию переноса по имени самой таблицы.

```sql
SELECT yezzey_define_offload_policy('parted_table_full_yezzey');

SELECT yezzey_define_offload_policy('parted_table_fract_yezzey_1_prt_2');

SELECT yezzey_define_offload_policy('parted_table_fract_yezzey_1_prt_3');

SELECT yezzey_define_offload_policy('parted_table_fract_yezzey_1_prt_4');

SELECT yezzey_define_offload_policy('parted_table_fract_yezzey_1_prt_other');

SELECT yezzey_define_offload_policy('non_parted_table_yezzey');
```

Обратим внимание, что для партицированных таблиц parent-таблицы остаются в каталоге `base`:
```sql
select p.relname as parent_t
, pg_relation_filepath(p.oid) as parent_t_path
, c.relname as partition_t
, pg_relation_filepath(c.oid) as partition_t_path
from pg_class p 
left join pg_inherits i on p.oid=i.inhparent
left join pg_class c on i.inhrelid = c.oid
where p.relname in
('parted_table_full_yezzey', 'parted_table_fract_yezzey','non_parted_table_yezzey')
order by 1 desc,3 desc

parent_t                 |parent_t_path      |partition_t                          |partition_t_path   |
-------------------------+-------------------+-------------------------------------+-------------------+
parted_table_full_yezzey |base/16521/278700  |parted_table_full_yezzey_1_prt_3     |yezzey/16521/278712|
parted_table_full_yezzey |base/16521/278700  |parted_table_full_yezzey_1_prt_2     |yezzey/16521/278708|
parted_table_full_yezzey |base/16521/278700  |parted_table_full_yezzey_1_prt_1     |yezzey/16521/278704|
parted_table_fract_yezzey|base/16521/278610  |parted_table_fract_yezzey_1_prt_other|yezzey/16521/278614|
parted_table_fract_yezzey|base/16521/278610  |parted_table_fract_yezzey_1_prt_4    |yezzey/16521/278626|
parted_table_fract_yezzey|base/16521/278610  |parted_table_fract_yezzey_1_prt_3    |yezzey/16521/278622|
parted_table_fract_yezzey|base/16521/278610  |parted_table_fract_yezzey_1_prt_2    |yezzey/16521/278618|
non_parted_table_yezzey  |yezzey/16521/278650|                                     |                   |

```

Добавим партицию к таблице `parted_table_full_yezzey`. Обратите внимание, что необходимо явно задать для новой партиции тип AO/AOCS, в противном случае возникнут ошибки в функциях работы с гибридным хранилищем на parent-таблице.

```sql
ALTER TABLE parted_table_full_yezzey add PARTITION 
START (date '2023-04-01') INCLUSIVE
END (date '2023-05-01') EXCLUSIVE
WITH (appendoptimized=true, orientation=column);

select p.relname as parent_t
, pg_relation_filepath(p.oid) as parent_t_path
, c.relname as partition_t
, pg_relation_filepath(c.oid) as partition_t_path
from pg_class p 
left join pg_inherits i on p.oid=i.inhparent
left join pg_class c on i.inhrelid = c.oid
where p.relname in
('parted_table_full_yezzey')--, 'parted_table_fract_yezzey','non_parted_table_yezzey')
order by 1 desc,3 desc;

parent_t                |parent_t_path    |partition_t                              |partition_t_path   |
------------------------+-----------------+-----------------------------------------+-------------------+
parted_table_full_yezzey|base/16521/278700|parted_table_full_yezzey_1_prt_r419097287|base/16521/278716  |
parted_table_full_yezzey|base/16521/278700|parted_table_full_yezzey_1_prt_3         |yezzey/16521/278712|
parted_table_full_yezzey|base/16521/278700|parted_table_full_yezzey_1_prt_2         |yezzey/16521/278708|
parted_table_full_yezzey|base/16521/278700|parted_table_full_yezzey_1_prt_1         |yezzey/16521/278704|
```
Новая партиция создается в `base`. Для переноса в гибридное хранилище необходимо вызвать функцию `yezzey_define_offload_policy` для новой партиции или повторно для всей parent-таблицы.

Теперь попробуем выполнить разделение (split) партиции, выгруженной в гибридное хранилище:
```sql
ALTER TABLE parted_table_fract_yezzey split default PARTITION 
START (date '2023-04-01') INCLUSIVE
END (date '2023-05-01') exclusive
INTO (PARTITION apr2023, PARTITION other);
WITH (appendoptimized=true, orientation=column)
;

select p.relname as parent_t
, pg_relation_filepath(p.oid) as parent_t_path
, c.relname as partition_t
, pg_relation_filepath(c.oid) as partition_t_path
from pg_class p 
left join pg_inherits i on p.oid=i.inhparent
left join pg_class c on i.inhrelid = c.oid
where p.relname in
('parted_table_fract_yezzey')--, 'parted_table_fract_yezzey','non_parted_table_yezzey')
order by 1 desc,3 desc

parent_t                 |parent_t_path    |partition_t                            |partition_t_path   |
-------------------------+-----------------+---------------------------------------+-------------------+
parted_table_fract_yezzey|base/16521/278610|parted_table_fract_yezzey_1_prt_other  |base/16521/278734  |
parted_table_fract_yezzey|base/16521/278610|parted_table_fract_yezzey_1_prt_apr2023|base/16521/278730  |
parted_table_fract_yezzey|base/16521/278610|parted_table_fract_yezzey_1_prt_4      |yezzey/16521/278626|
parted_table_fract_yezzey|base/16521/278610|parted_table_fract_yezzey_1_prt_3      |yezzey/16521/278622|
parted_table_fract_yezzey|base/16521/278610|parted_table_fract_yezzey_1_prt_2      |yezzey/16521/278618|
```
\- новая и разделяемая партиция создались в `base`, автоматической выгрузки в гибридное хранилище не произошло.

# Тестирование производительности гибридного хранилища

Для оценки производительности расширения Yezzey было выполнено тестирование в инфраструктуре Yandex Cloud.
Сравнивалась производительность запросов поверх:
- объектов, находящихся во внутреннем локальном хранилище Greenplum
- объектов, перенесенных в гибридное хранилище при помощи расширения Yezzey
- объектов из Object Storage (S3), подключенных в Greenplum как внешние PXF-таблицы

## Инфраструктура тестирования
> [!NOTE]
> Все проведенные ниже замеры производительности запросов проводились на кластере следующей конфигурации:
> Master x2
> Класс хоста i3-c16-m128 (16 vCPU, 100% vCPU rate, 128 ГБ RAM)
> Хранилище 186 ГБ network-ssd-nonreplicated 
> **Segment** x8
> Класс хоста i3-c16-m128 (16 vCPU, 100% vCPU rate, 128 ГБ RAM)
> Хранилище 7.36 ТБ network-ssd-nonreplicated 
> Количество сегментов на хост 4
> Версия Managed Greenplum `6.25.3-mdb+yezzey+yagpcc-r+dev.11.ge3a7c26b10 build dev-oss`

## Подготовка данных

Тестирование проводилось на данных о поездках NY Yellow Taxi[^3] за период 2013-2022 гг.
Данные в формате parquet были предварительно выложены в S3 bucket и загружены в Greenplum через PXF-интерфейс.
Для каждого каталога были созданы отдельные таблицы, содержащие данные соответствующего календарного года, после чего они были загружены в партицированную по календарным годам таблицу `ny_taxi_src_gp`.
![s3-bucket-list](img/s3-bucket-list.png?raw=true "Структура каталогов в  Object Storage")

Таблицы фактов с поездками для всех движков были намеренно распределены по сегментам случайным образом (DISTRIBUTED RANDOMLY) с целью равномерного распределения данных и гарантированного вызова операции редистрибуции при агрегировании данных.

Все sql-запросы для создания объектов и тестирования выложены в каталоге [sql](/sql).

[!NOTE]
> DDL- и DML-запросы для скриптов создания и наполнения PXF-таблиц для разных календарных периодов могут несколько отличаться по типам данных из-за особенностей чтения parquet-файлов.
> Также отметим, что в запросе создания финального представления поверх PXF-таблиц было применено объединение через `UNION ALL` и выборка из каждой таблицы была дополена явным интервальным фильтром `WHERE` на соответствующий календарный год, чтобы при обращении к определенному календарному периоду запрос шел именно к таблице с этим периодом.

Размер таблицы фактов о поездках “желтого такси” с 2013 по 2022 год составил ~ 1 млрд строк.

![ny-taxi-row-count](img/ny-taxi-row-count.png?raw=true "Количество строк в датасете")

![ny-taxi-fact-table-row-count-per-year](img/ny-taxi-fact-table-row-count-per-year.png?raw=true "Количество строк в датасете по годам")


Перечень тестовых запросов:

- Выборка агрегата (без редистрибуции) - `select count(1) from <table>;`

- Выборка агрегата с одним разрезом `select vendorid, count(1) from <table> group by vendorid;`

- Выборка агрегата с 4 разрезами (для оценки снижения производительности при добавлении колонок в выборку) `select vendorid, pickup_date, passenger_count, payment_type, count(1) from <table> group by 1,2,3,4;`

- Выборка с аналитической функцией на 1й партиции `select distinct dolocationid, max(total_amount) over (partition by dolocationid)from <table> where pickup_date between '2014-01-01'::date and '2014-12-31'::date;`

- Выборка с аналитической функцией на 3х партициях `select distinct dolocationid, max(total_amount) over (partition by dolocationid)from <table> where pickup_date between '2014-01-01'::date and '2016-12-31'::date;`

- Выборка с соединением со справочником и агрегацией по 3м партициям `select z.locationid, count(r.vendorid), sum(r.total_amount) from ny_taxi_src_gp r left join ny_taxi_zones_src_gp z on r.pulocationid=z.locationid where r.pickup_date between '2014-01-01'::date and '2016-12-31'::date group by 1;`

- Выборка с соединением со справочником и агрегацией по всей таблице `select z.locationid, count(r.vendorid), sum(r.total_amount) from ny_taxi_src_gp r left join ny_taxi_zones_src_gp z on r.pulocationid=z.locationid group by 1;`

Каждый запрос для каждого из движков запускался по 5 раз, чтобы исключить случайные ошибки во время их исполнения.

Запросы выполнялись поодиночке без какой-либо фоновой нагрузки для обеспечения полной сопоставимости результатов.

## Настройка сценариев в Jmeter

Для целей автоматизации тестирования можно воспользоваться популярным инструментом Apache JMeter[^4].
В каталог [jmeter](jmeter/) вы можете найти jmx-файл с примером сценария тестирования, в котором каждый из 7 запросов для каждого трех движков размещен в отдельной thread group. При использовании примера не забудьте указан в настройках соединения адрес мастер-сервера, логин и пароль пользователя вашего кластера Greenplum.

Также тестирование можно провести при помощи сервиса Yandex Load Testing[^5], поддерживающий сценарии тестирования в JMeter-формате. Данный сервис поможет сохранять и сравнивать результаты тестов между собой, а также применять jmx-конфигурации, размещенными в Yandex Object Storage.

## Статистика выполнения запросов

Все результаты приведены в миллисекундах:
| Запрос и движок | Запуск 1 | Запуск 2 | Запуск 3 | Запуск 4 | Запуск 5 |
|-|-|-|-|-|-|
| Запрос 1 | 
| 1.1 - Greenplum | 9164 | 8707 | 8869 | 8919 | 9026 |
| 1.2 - Yezzey | 10522 | 9926 | 9932 | 10047 | 9902 |
| 1.3 - PXF | 130204 | 101167 | 99613 | 100392 |101300 |
|||||||
| Запрос 2 | 
| 2.1 - Greenplum | 11420 | 11124 | 10816 | 10751 | 10889 |
| 2.2 - Yezzey | 11916 | 12052 | 11884 | 11776 | 11877 |
| 2.3 - PXF | 104182 | 104409 | 104557 | 105440 | 104294 |
|||||||
| Запрос 3 | 
| 3.1 - Greenplum | 17003 | 16661 | 17324 | 15417 | 15089 |
| 3.2 - Yezzey | 25376 | 21662 | 21422 | 19963 | 20173 |
| 3.3 - PXF | 135290 | 144987 | 137866 | 127623 | 135787 |
|||||||
| Запрос 4 | 
| 4.1 - Greenplum | 19351 | 18616 | 18232 | 17499 | 17512 |
| 4.2 - Yezzey | 18538 | 18591 | 18395 | 18255 | 18116 |
| 4.3 - PXF | 28787 | 28305 | 28691 | 31650 | 32707 |
|||||||
| Запрос 5 | 
| 5.1 - Greenplum | 57245 | 55375 | 57146 | 59396 | 57214 |
| 5.2 - Yezzey | 57258 | 57023 | 58675 | 57575 | 58836 |
| 5.3 - PXF | 87905 | 85268 | 90521 | 83933 | 83531 |
|||||||
| Запрос 6 | 
| 6.1 - Greenplum | 11986 | 11654 | 11656 | 11517 | 11785 |
| 6.2 - Yezzey | 14392 | 13688 | 13716 | 13721 | 13498 |
| 6.3 - PXF | 68833 | 71021 | 59199 | 74602 | 66264 | 
|||||||
| Запрос 7 | 
| 7.1 - Greenplum | 23991 | 23576 | 23812 | 23739 | 23750 |
| 7.2 - Yezzey | 29564 | 28134 | 28418 | 28058 | 28132 |
| 7.3 - PXF | 134189 | 140649 | 133689 | 133278 | 142266 |

Из полученных результатов выполнения запросов следует, что движок Yezzey в инфраструктуре Yandex Cloud способен демонстрировать сопоставимую с движком внутренних таблиц Greenplum производительность, а PXF был значительно медленнее.

### Анализ планов запросов

Планы запросов для внутренних таблица GP и Yezzey не имеют отличий, кроме более долгого чтения данных для Yezzey, а также объемов выделяемой памяти.
Пример:

```sql
/* План запроса для запроса над внутренней таблицей Greenplum */
Gather Motion 8:1 (slice3; segments: 8) (cost=0.00..92453.64 rows=185 width=18) (actual time=77885.505..77885.570 rows=265 loops=1)
-> HashAggregate (cost=0.00..92453.63 rows=24 width=18) (actual time=77884.209..77884.222 rows=37 loops=1)
Group Key: ny_taxi_zones_src_gp.locationid
Extra Text: (seg1) Hash chain length 1.7 avg, 4 max, using 22 of 32 buckets; total 0 expansions.
-> Redistribute Motion 8:8 (slice2; segments: 8) (cost=0.00..92453.62 rows=24 width=18) (actual time=74069.634..77884.048 rows=296 loops=1)
Hash Key: ny_taxi_zones_src_gp.locationid
-> Result (cost=0.00..92453.62 rows=24 width=18) (actual time=74069.141..74069.205 rows=265 loops=1)
-> HashAggregate (cost=0.00..92453.62 rows=24 width=18) (actual time=74069.135..74069.170 rows=265 loops=1)
Group Key: ny_taxi_zones_src_gp.locationid
Extra Text: (seg1) Hash chain length 4.2 avg, 10 max, using 63 of 64 buckets; total 1 expansions.
-> Hash Left Join (cost=0.00..60548.60 rows=251092152 width=18) (actual time=1.556..54467.766 rows=126342503 loops=1)
Hash Cond: (ny_taxi_src_gp.pulocationid = (ny_taxi_zones_src_gp.locationid)::bigint)
Extra Text: (seg6) Hash chain length 1.0 avg, 1 max, using 265 of 262144 buckets.
-> Sequence (cost=0.00..11825.68 rows=126326829 width=24) (actual time=0.536..28761.826 rows=126342503 loops=1)
-> Partition Selector for ny_taxi_src_gp (dynamic scan id: 1) (cost=10.00..100.00 rows=13 width=4) (never executed)
Partitions selected: 13 (out of 13)
/* Отличие 1 здесь */ -> Dynamic Seq Scan on ny_taxi_src_gp (dynamic scan id: 1) (cost=0.00..11825.68 rows=126326829 width=24) (actual time=0.516..20393.447 rows=126342503 loops=1)
Partitions scanned: Avg 13.0 (out of 13) x 8 workers. Max 13 parts (seg0).
-> Hash (cost=431.01..431.01 rows=265 width=2) (actual time=0.119..0.119 rows=265 loops=1)
-> Broadcast Motion 8:8 (slice1; segments: 8) (cost=0.00..431.01 rows=265 width=2) (actual time=0.028..0.058 rows=265 loops=1)
-> Seq Scan on ny_taxi_zones_src_gp (cost=0.00..431.00 rows=34 width=2) (actual time=0.092..0.106 rows=37 loops=1)
Planning time: 45.328 ms
(slice0) Executor memory: 276K bytes.
(slice1) Executor memory: 172K bytes avg x 8 workers, 172K bytes max (seg0).
/* Отличие 2 здесь */ (slice2) Executor memory: 3090K bytes avg x 8 workers, 3096K bytes max (seg1). Work_mem: 7K bytes max.
(slice3) Executor memory: 120K bytes avg x 8 workers, 120K bytes max (seg0).
Memory used: 128000kB
Optimizer: Pivotal Optimizer (GPORCA)
Execution time: 77895.147 ms
```

```sql
/* План запроса для запроса над Yezzey-таблицей */
Gather Motion 8:1 (slice3; segments: 8) (cost=0.00..92452.68 rows=176 width=18) (actual time=92544.958..92545.189 rows=265 loops=1)
-> HashAggregate (cost=0.00..92452.67 rows=22 width=18) (actual time=92544.293..92544.305 rows=37 loops=1)
Group Key: ny_taxi_zones_src_gp.locationid
Extra Text: (seg1) Hash chain length 1.7 avg, 4 max, using 22 of 32 buckets; total 0 expansions.
-> Redistribute Motion 8:8 (slice2; segments: 8) (cost=0.00..92452.67 rows=22 width=18) (actual time=87429.478..92544.111 rows=296 loops=1)
Hash Key: ny_taxi_zones_src_gp.locationid
-> Result (cost=0.00..92452.67 rows=22 width=18) (actual time=87428.084..87428.161 rows=265 loops=1)
-> HashAggregate (cost=0.00..92452.67 rows=22 width=18) (actual time=87428.074..87428.114 rows=265 loops=1)
Group Key: ny_taxi_zones_src_gp.locationid
Extra Text: (seg0) Hash chain length 4.2 avg, 10 max, using 63 of 64 buckets; total 1 expansions.
-> Hash Left Join (cost=0.00..60548.28 rows=251087134 width=18) (actual time=932.328..70276.159 rows=126345890 loops=1)
Hash Cond: (ny_taxi_yezzey.pulocationid = (ny_taxi_zones_src_gp.locationid)::bigint)
Extra Text: (seg7) Hash chain length 1.0 avg, 1 max, using 265 of 262144 buckets.
-> Sequence (cost=0.00..11825.68 rows=126326829 width=24) (actual time=931.263..44905.535 rows=126345890 loops=1)
-> Partition Selector for ny_taxi_yezzey (dynamic scan id: 1) (cost=10.00..100.00 rows=13 width=4) (never executed)
Partitions selected: 13 (out of 13)
/* Отличие 1 здесь */ -> Dynamic Seq Scan on ny_taxi_yezzey (dynamic scan id: 1) (cost=0.00..11825.68 rows=126326829 width=24) (actual time=931.247..36429.835 rows=126345890 loops=1)
Partitions scanned: Avg 13.0 (out of 13) x 8 workers. Max 13 parts (seg0).
-> Hash (cost=431.01..431.01 rows=265 width=2) (actual time=0.151..0.151 rows=265 loops=1)
-> Broadcast Motion 8:8 (slice1; segments: 8) (cost=0.00..431.01 rows=265 width=2) (actual time=0.026..0.094 rows=265 loops=1)
-> Seq Scan on ny_taxi_zones_src_gp (cost=0.00..431.00 rows=34 width=2) (actual time=0.067..0.075 rows=37 loops=1)
Planning time: 44.083 ms
(slice0) Executor memory: 276K bytes.
(slice1) Executor memory: 172K bytes avg x 8 workers, 172K bytes max (seg0).
/* Отличие 2 здесь */ (slice2) Executor memory: 3886007K bytes avg x 8 workers, 3935163K bytes max (seg2). Work_mem: 7K bytes max.
(slice3) Executor memory: 120K bytes avg x 8 workers, 120K bytes max (seg0).
Memory used: 128000kB
Optimizer: Pivotal Optimizer (GPORCA)
Execution time: 92556.197 ms
```

# Заключение
Ваши предложения по модификации сценария вы можете направить через pull request.

Для вопросов, пожеланий и консультаций по сервисам платформы данных Yandex Cloud: группа https://t.me/YandexDataPlatform в Telegram

[^1]: https://github.com/yezzey-gp
[^2]: https://cloud.yandex.ru/docs/managed-Greenplum/tutorials/yezzey
[^3]: https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page
https://cloud.yandex.ru/docs/load-testing/tutorials/loadtesting-jmeter
[^4]: https://jmeter.apache.org
[^5]: https://cloud.yandex.ru/docs/load-testing/
# Cooling data in Object Storage for Managed Greenplum

# Introduction

Many modern database management systems (DBMS’s) and analytical data processing engines use a disaggregated approach for storage and compute layers. This approach enables one to rapidly recover the system availability even after major failures, including complete resource loss. It achieves this by using a geo-distributed, fault-tolerant file storage layer (S3-like) and an elastic compute layer that operates across different data centers (availability zones).

Greenplum uses a traditional method where data is stored locally on segment hosts. This approach has several downsides; for example, if a single disk fails, you may need to re-allocate (or recreate) a segment, or even the entire segment host. In Yandex Cloud, one more limitation is the limited disk size, stemming from the specifics of the cloud platform infrastructure. This can sometimes mean you need to expand the cluster because of insufficient disk resources, even when CPU and RAM performance is fine.

To address the issues mentioned above, the Yandex Cloud development team created Yezzey[^1], an extension to facilitate hybrid storage in Greenplum. With Yezzey, you can transfer AO and AOCS tables from Greenplum to object storage and back, either in full or by individual partitions, without reducing read performance in any significant way. From the end user perspective, the process of transferring data to object storage does not impact how objects are read. You do not need to create any additional views or custom transfer procedures, which are standard practices for data storage tiering in Greenplum.

# Working with Yezzey in Managed Greenplum

For more information about basic operations, see the Yandex Cloud documentation[^2]. Below, we will explore some features of the Yezzey module, including how it works with partitioned tables.

## Offloading data to Object Storage

To demonstrate how Yezzey works, let's create three tables: two of them will have partitions (one will include an `other` partition and the other will not), and one table will have no partitions at all.

```sql

/* Create three tables */
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
    /* Do not specify the `other` partition */
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
    /* Specify the `other` partition */
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

/* Then, insert 1 million rows into each table */
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

Now, transfer the table data to a hybrid storage. For `parted_table_fract_yezzey`, explicitly transfer the partitions; for other tables, use a transfer function that takes the table name as an argument.

```sql
SELECT yezzey_define_offload_policy('parted_table_full_yezzey');

SELECT yezzey_define_offload_policy('parted_table_fract_yezzey_1_prt_2');

SELECT yezzey_define_offload_policy('parted_table_fract_yezzey_1_prt_3');

SELECT yezzey_define_offload_policy('parted_table_fract_yezzey_1_prt_4');

SELECT yezzey_define_offload_policy('parted_table_fract_yezzey_1_prt_other');

SELECT yezzey_define_offload_policy('non_parted_table_yezzey');
```

For each partitioned table, note that its parent table remains in the `base` folder:

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

Let's add a partition to the `parted_table_full_yezzey` table. Note that you must explicitly set the type of the new partition to AO/AOCS; otherwise, you may encounter errors in the functions that work with the hybrid storage on the parent table.

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

The new partition will appear in `base`. To transfer it to the hybrid storage, you need to invoke the `yezzey_define_offload_policy` function for the new partition, or you can invoke it again for the entire parent table.

Now, let's attempt to split the partition that was offloaded into the hybrid storage:

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

Both new partition and the one we attempted to split were created in the `base` folder, but there was no automatic offloading to the hybrid storage.

# Testing the hybrid storage performance

To evaluate Yezzey's performance, we run a number of tests in the Yandex Cloud infrastructure. We compared the performance of queries against:
* Objects stored in the Greenplum internal local storage.
* Objects transferred into a hybrid storage with Yezzey.
* Objects from Object Storage (S3) connected to Greenplum as external PXF tables.

## Testing infrastructure

> [!NOTE]
> Below you can see the query performance measurements we run on a cluster with the following configuration:
> Master x2
> Host class: i3-c16-m128 (16 vCPUs, 100% vCPU rate, 128 GB RAM)
> Storage: 186 GB, `network-ssd-nonreplicated` 
> **Segment** x8
> Host class: i3-c16-m128 (16 vCPUs, 100% vCPU rate, 128 GB RAM)
> Storage: 7.36 TB, `network-ssd-nonreplicated` 
> Number of segments per host: 4
> Managed Greenplum version: `6.25.3-mdb+yezzey+yagpcc-r+dev.11.ge3a7c26b10 build dev-oss`

## Preparing data

For testing, we used the NY Yellow Taxi trip data[^3] from 2013 to 2022. We uploaded this data in Parquet format to an S3 bucket and then, into Greenplum through the PXF interface. We also created a separate table for each folder containing data from the respective calendar year, and then loaded those tables into the `ny_taxi_src_gp` table partitioned by calendar years.
![s3-bucket-list](img/s3-bucket-list.png?raw=true "Folder structure in Object Storage")

The fact tables containing trip data for all engines were intentionally distributed across segments randomly to ensure that the data is distributed evenly and the redistribution operation is invoked during data aggregation.

The [sql](/sql) folder contains all SQL queries for creating objects and testing.

[!NOTE]
> DDL and DML queries for scripts that create and populate PXF tables for various calendar periods may vary slightly in data types due to certain specifics of reading Parquet files.
> Also note that the query for creating the final view over the PXF tables has `UNION ALL`, and selecting each table was enhanced with an explicit `WHERE` clause to filter the respective calendar year. Thus, a query for a specific calendar period only addresses the table for that period.

The fact table for Yellow Taxi trips from 2013 to 2022 contained around 1 billion rows.

![ny-taxi-row-count](img/ny-taxi-row-count.png?raw=true "Number of rows in the dataset")

![ny-taxi-fact-table-row-count-per-year](img/ny-taxi-fact-table-row-count-per-year.png?raw=true "Number of rows in the dataset by year")

List of test queries:

* Select an aggregate (without redistribution): `select count(1) from <table>;`.

* Select an aggregate with a single slice: `select vendorid, count(1) from <table> group by vendorid;`.

* Select an aggregate with four slices (to evaluate how much the performance reduces when adding columns to the output): `select vendorid, pickup_date, passenger_count, payment_type, count(1) from <table> group by 1,2,3,4;`.

* Select statement with an analytic function over one partition: `select distinct dolocationid, max(total_amount) over (partition by dolocationid)from <table> where pickup_date between '2014-01-01'::date and '2014-12-31'::date;`.

* Select statement with an analytic function over three partitions: `select distinct dolocationid, max(total_amount) over (partition by dolocationid)from <table> where pickup_date between '2014-01-01'::date and '2016-12-31'::date;`.

* Select statement with a join to the reference table and aggregation across three partitions: `select z.locationid, count(r.vendorid), sum(r.total_amount) from ny_taxi_src_gp r left join ny_taxi_zones_src_gp z on r.pulocationid=z.locationid where r.pickup_date between '2014-01-01'::date and '2016-12-31'::date group by 1;`.

* Select statement with a join to the reference table and aggregation across the entire table: `select z.locationid, count(r.vendorid), sum(r.total_amount) from ny_taxi_src_gp r left join ny_taxi_zones_src_gp z on r.pulocationid=z.locationid group by 1;`.

For each of the engines, each query ran five times, to avoid random errors.

The queries ran individually, without any background load, to ensure full comparability of the results.

## Setting up Jmeter scenarios

You can automate your testing procedures with a popular tool, Apache JMeter[^4]. The [jmeter](jmeter/) folder houses a `.jmx` file containing an example testing script, with a separate thread group for each of the seven queries for each of the three engines. When running the example, make sure to provide the master server address, username, and user password for your Greenplum cluster in the connection settings.

You can also perform testing with Yandex Load Testing[^5], which supports testing scripts in JMeter format. With Yandex Load Testing, you can save and compare test results, as well as apply `jmx` configurations stored in Yandex Object Storage.

## Query execution statistics

All results provided below are in milliseconds.
| Query and engine | Run 1 | Run 2 | Run 3 | Run 4 | Run 5 |
|-|-|-|-|-|-|
| Query 1 | 
| 1.1 - Greenplum | 9164 | 8707 | 8869 | 8919 | 9026 |
| 1.2 - Yezzey | 10522 | 9926 | 9932 | 10047 | 9902 |
| 1.3 - PXF | 130204 | 101167 | 99613 | 100392 |101300 |
|||||||
| Query 2 | 
| 2.1 - Greenplum | 11420 | 11124 | 10816 | 10751 | 10889 |
| 2.2 - Yezzey | 11916 | 12052 | 11884 | 11776 | 11877 |
| 2.3 - PXF | 104182 | 104409 | 104557 | 105440 | 104294 |
|||||||
| Query 3 | 
| 3.1 - Greenplum | 17003 | 16661 | 17324 | 15417 | 15089 |
| 3.2 - Yezzey | 25376 | 21662 | 21422 | 19963 | 20173 |
| 3.3 - PXF | 135290 | 144987 | 137866 | 127623 | 135787 |
|||||||
| Query 4 | 
| 4.1 - Greenplum | 19351 | 18616 | 18232 | 17499 | 17512 |
| 4.2 - Yezzey | 18538 | 18591 | 18395 | 18255 | 18116 |
| 4.3 - PXF | 28787 | 28305 | 28691 | 31650 | 32707 |
|||||||
| Query 5 | 
| 5.1 - Greenplum | 57245 | 55375 | 57146 | 59396 | 57214 |
| 5.2 - Yezzey | 57258 | 57023 | 58675 | 57575 | 58836 |
| 5.3 - PXF | 87905 | 85268 | 90521 | 83933 | 83531 |
|||||||
| Query 6 | 
| 6.1 - Greenplum | 11986 | 11654 | 11656 | 11517 | 11785 |
| 6.2 - Yezzey | 14392 | 13688 | 13716 | 13721 | 13498 |
| 6.3 - PXF | 68833 | 71021 | 59199 | 74602 | 66264 | 
|||||||
| Query 7 | 
| 7.1 - Greenplum | 23991 | 23576 | 23812 | 23739 | 23750 |
| 7.2 - Yezzey | 29564 | 28134 | 28418 | 28058 | 28132 |
| 7.3 - PXF | 134189 | 140649 | 133689 | 133278 | 142266 |

The results show that the Yezzey engine performance in the Yandex Cloud infrastructure is comparable to that of the Greenplum internal table engine, while PXF was significantly slower.

### Analyzing the query plans

The query plans for the internal GP tables and Yezzey do not have any differences, apart from Yezzey having longer data reading time and different allocated memory size. Here is an example:

```sql
/* Query plan for a query over a Greenplum internal table */
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
/* This is the first difference */ -> Dynamic Seq Scan on ny_taxi_src_gp (dynamic scan id: 1) (cost=0.00..11825.68 rows=126326829 width=24) (actual time=0.516..20393.447 rows=126342503 loops=1)
Partitions scanned: Avg 13.0 (out of 13) x 8 workers. Max 13 parts (seg0).
-> Hash (cost=431.01..431.01 rows=265 width=2) (actual time=0.119..0.119 rows=265 loops=1)
-> Broadcast Motion 8:8 (slice1; segments: 8) (cost=0.00..431.01 rows=265 width=2) (actual time=0.028..0.058 rows=265 loops=1)
-> Seq Scan on ny_taxi_zones_src_gp (cost=0.00..431.00 rows=34 width=2) (actual time=0.092..0.106 rows=37 loops=1)
Planning time: 45.328 ms
(slice0) Executor memory: 276K bytes.
(slice1) Executor memory: 172K bytes avg x 8 workers, 172K bytes max (seg0).
/* This is the second difference */ (slice2) Executor memory: 3090K bytes avg x 8 workers, 3096K bytes max (seg1). Work_mem: 7K bytes max.
(slice3) Executor memory: 120K bytes avg x 8 workers, 120K bytes max (seg0).
Memory used: 128000kB
Optimizer: Pivotal Optimizer (GPORCA)
Execution time: 77895.147 ms
```

```sql
/* Query plan for a query over a Yezzey table */
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
/* This is the first difference */ -> Dynamic Seq Scan on ny_taxi_yezzey (dynamic scan id: 1) (cost=0.00..11825.68 rows=126326829 width=24) (actual time=931.247..36429.835 rows=126345890 loops=1)
Partitions scanned: Avg 13.0 (out of 13) x 8 workers. Max 13 parts (seg0).
-> Hash (cost=431.01..431.01 rows=265 width=2) (actual time=0.151..0.151 rows=265 loops=1)
-> Broadcast Motion 8:8 (slice1; segments: 8) (cost=0.00..431.01 rows=265 width=2) (actual time=0.026..0.094 rows=265 loops=1)
-> Seq Scan on ny_taxi_zones_src_gp (cost=0.00..431.00 rows=34 width=2) (actual time=0.067..0.075 rows=37 loops=1)
Planning time: 44.083 ms
(slice0) Executor memory: 276K bytes.
(slice1) Executor memory: 172K bytes avg x 8 workers, 172K bytes max (seg0).
/* This is the second difference */ (slice2) Executor memory: 3886007K bytes avg x 8 workers, 3935163K bytes max (seg2). Work_mem: 7K bytes max.
(slice3) Executor memory: 120K bytes avg x 8 workers, 120K bytes max (seg0).
Memory used: 128000kB
Optimizer: Pivotal Optimizer (GPORCA)
Execution time: 92556.197 ms
```

# Conclusion

You can submit your suggestions for modifying the scenario through a pull request.

For questions, suggestions, and advice on Yandex Cloud services, [join our Telegram chat](https://t.me/YandexDataPlatform).

[^1]: https://github.com/yezzey-gp
[^2]: https://yandex.cloud/docs/managed-Greenplum/tutorials/yezzey
[^3]: https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page
https://yandex.cloud/docs/load-testing/tutorials/loadtesting-jmeter
[^4]: https://jmeter.apache.org
[^5]: https://yandex.cloud/docs/load-testing/

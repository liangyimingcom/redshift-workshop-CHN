---
title: "5. SPECTRUM查询调优"
chapter: false
weight: 50
---



# SPECTRUM查询调优

在本实验中，我们将向您展示如何通过利用分区、优化存储和谓词下推来诊断您的 Redshift Spectrum 查询性能并优化性能。

## 内容

- [开始之前](https://redshift-immersion.workshop.aws/lab5.html#before-you-begin)
- [使用 Amazon Redshift Spectrum 查询](https://redshift-immersion.workshop.aws/lab5.html#querying-with-amazon-redshift-spectrum)
- [性能诊断](https://redshift-immersion.workshop.aws/lab5.html#performance-diagnostics)
- [使用分区优化](https://redshift-immersion.workshop.aws/lab5.html#optimizing-with-partitions)
- [存储优化](https://redshift-immersion.workshop.aws/lab5.html#storage-optimizations)
- [谓词下推](https://redshift-immersion.workshop.aws/lab5.html#predicate-pushdown)
- [Redshift Spectrum 请求加速器](https://redshift-immersion.workshop.aws/lab5.html#redshift-spectrum-request-accelerator)
- [Native Redshift 与 Redshift with Spectrum](https://redshift-immersion.workshop.aws/lab5.html#native-redshift-versus-redshift-with-spectrum)
- [离开前](https://redshift-immersion.workshop.aws/lab5.html#before-you-leave)

## 在你开始之前

本实验假设您已启动 Redshift 集群并配置了客户端工具。如果您尚未启动集群，请参阅 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html)。如果您尚未配置客户端工具，请参阅 [实验室 1 - 创建 Redshift 集群：配置客户端工具](https://redshift-immersion.workshop.aws/lab1.html#configure-client-tool)。对于本实验，您需要从 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html) 中收集有关集群的以下信息。

- [Your-Redshift_Role_Arn]
- [Your-Glue_Role]

本实验需要在 *US-WEST-2（俄勒冈）* 区域中的集群。

## 使用 Amazon Redshift Spectrum 进行查询

通过在 Redshift 集群中创建维度表和 S3 中的事实表来创建星型架构数据模型，如下图所示。

1. 通过从客户端工具运行此脚本来创建维度表。

```sql
DROP TABLE IF EXISTS customer;
CREATE TABLE customer (
  c_custkey     	integer        not null sortkey,
  c_name        	varchar(25)    not null,
  c_address     	varchar(25)    not null,
  c_city        	varchar(10)    not null,
  c_nation      	varchar(15)    not null,
  c_region      	varchar(12)    not null,
  c_phone       	varchar(15)    not null,
  c_mktsegment      varchar(10)    not null)
diststyle all;

DROP TABLE IF EXISTS dwdate;
CREATE TABLE dwdate (
  d_datekey            integer       not null sortkey,
  d_date               varchar(19)   not null,
  d_dayofweek	      varchar(10)   not null,
  d_month      	    varchar(10)   not null,
  d_year               integer       not null,
  d_yearmonthnum       integer  	 not null,
  d_yearmonth          varchar(8)	not null,
  d_daynuminweek       integer       not null,
  d_daynuminmonth      integer       not null,
  d_daynuminyear       integer       not null,
  d_monthnuminyear     integer       not null,
  d_weeknuminyear      integer       not null,
  d_sellingseason      varchar(13)    not null,
  d_lastdayinweekfl    varchar(1)    not null,
  d_lastdayinmonthfl   varchar(1)    not null,
  d_holidayfl          varchar(1)    not null,
  d_weekdayfl          varchar(1)    not null)
diststyle all;
```

2. 通过运行以下脚本将数据加载到维度表中。 您需要为 IAM 角色提供在集群上运行 COPY 命令的权限。 您可以使用之前确定的 IAM 角色。 这会将数据集从 S3 加载到您的 Redshift 集群中。 预计脚本需要几分钟才能完成。 客户和时间维度分别由 3M 条记录和 2556 条记录组成。

确保替换下面脚本中的“[Your-Redshift_Role_Arn]”值。

```sql
copy customer from 's3://awssampledbuswest2/ssbgz/customer'
iam_role '[Your-Redshift_Role_Arn]'
gzip region 'us-west-2';

copy dwdate from 's3://awssampledbuswest2/ssbgz/dwdate'
iam_role '[Your-Redshift_Role_Arn]'
gzip region 'us-west-2';
```

3. 接下来，创建一个 *External Schema* 来引用驻留在 Redshift 集群之外的数据集。 通过运行以下命令定义此架构。 您需要为 IAM 角色提供从集群中读取 S3 日期的权限。 这应该与上面 COPY 命令中使用的角色相同。 默认情况下，Redshift 将描述外部数据库和架构的元数据存储在 AWS Glue 数据目录中。 创建后，您可以从 Glue 或 Athena 查看架构。

确保替换下面脚本中的“[Your-Redshift_Role_Arn]”值。

```sql
CREATE EXTERNAL SCHEMA clickstream
from data catalog database 'clickstream'
iam_role '[Your-Redshift_Role_Arn]'
CREATE EXTERNAL DATABASE IF NOT EXISTS;
```

4. 使用 AWS Glue Crawler 在位置 s3://redshift-spectrum-bigdata-blog-datasets/clickstream-csv10 和 s3://redshift-spectrum 下创建外部表 clickstream.clickstream-csv10 和 clickstream.clickstream-parquet1 -bigdata-blog-datasets/clickstream-parquet1 分别。
   1. Navigate to the `Glue Crawler Page`. https://us-west-2.console.aws.amazon.com/glue/home?region=us-west-2#catalog:tab=crawlers
   [![img](https://redshift-immersion.workshop.aws/images/crawler_0.png)](https://redshift-immersion.workshop.aws/images/crawler_0.png)
   2. Click on *Add Crawler*, and enter the crawler name *clickstream* and click *Next*.[![img](https://redshift-immersion.workshop.aws/images/crawler_1_clickstream.png)](https://redshift-immersion.workshop.aws/images/crawler_1_clickstream.png)
   3. Select *Data stores* as the source type and click *Next*.[![img](https://redshift-immersion.workshop.aws/images/crawler_2.png)](https://redshift-immersion.workshop.aws/images/crawler_2.png)
   4. Choose *S3* as the data store and the include path of *s3://redshift-immersionday-labs/data/clickstream*[![img](https://redshift-immersion.workshop.aws/images/crawler_3_clickstream.png)](https://redshift-immersion.workshop.aws/images/crawler_3_clickstream.png)
   5. *Choose an existing IAM Role* and select a Role which Glue can assume and which has access to S3. If you don’t have a Glue Role, you can also select *Create an IAM role*.[![img](https://redshift-immersion.workshop.aws/images/crawler_4_clickstream.png)](https://redshift-immersion.workshop.aws/images/crawler_4_clickstream.png)
   6. Select *Run on demand* for the frequency.[![img](https://redshift-immersion.workshop.aws/images/crawler_5.png)](https://redshift-immersion.workshop.aws/images/crawler_5.png)
   7. Select the Database *clickstream* from the list.[![img](https://redshift-immersion.workshop.aws/images/crawler_6_clickstream.png)](https://redshift-immersion.workshop.aws/images/crawler_6_clickstream.png)
   8. Select all remaining defaults. Once the Crawler has been created, click on *Run Crawler*.[![img](https://redshift-immersion.workshop.aws/images/crawler_7_clickstream.png)](https://redshift-immersion.workshop.aws/images/crawler_7_clickstream.png)
   9. Once the Crawler has completed its run, you will see two new tables in the Glue Catalog. https://us-west-2.console.aws.amazon.com/glue/home?region=us-west-2#catalog:tab=tables
   [![img](https://redshift-immersion.workshop.aws/images/crawler_8_clickstream.png)](https://redshift-immersion.workshop.aws/images/crawler_8_clickstream.png)
   10. Click on the *uservisits_parquet1* table. Notice the recordCount of 3.8 billion.[![img](https://redshift-immersion.workshop.aws/images/crawler_9_clickstream.png)](https://redshift-immersion.workshop.aws/images/crawler_9_clickstream.png)
   11. Navigate back to the Glue Catalog https://us-west-2.console.aws.amazon.com/glue/home?region=us-west-2#catalog:tab=tables. Click on the *uservisits_csv10* table. Notice the column names have not been set. Also notice that field *col0* is set to a datatype of *String*. This field represents *adRevenue* and should be set as a datatype of *double*.
       [![img](https://redshift-immersion.workshop.aws/images/crawler_10_clickstream.png)](https://redshift-immersion.workshop.aws/images/crawler_10_clickstream.png)
   12. Click on *Edit Schema* and adjust the column names and datatypes. Click *Save*.[![img](https://redshift-immersion.workshop.aws/images/crawler_11_clickstream.png)](https://redshift-immersion.workshop.aws/images/crawler_11_clickstream.png)
5. 导航回您的 SQL 客户端工具并运行下面的查询。 此查询在 Redshift 中的维度表和 S3 中的点击流事实表之间执行连接，有效地混合了来自数据湖和数据仓库的数据。 此查询按客户 1 到 3 的细分市场返回我们数据集过去 3 个月的总广告收入。广告收入数据源自 S3，而客户和时间属性（如市场细分）源自 Redshift 中的维度表。

```sql
SELECT c.c_name, c.c_mktsegment, t.prettyMonthYear, SUM(uv.adRevenue)
FROM clickstream.uservisits_csv10 as uv
RIGHT OUTER JOIN customer as c ON c.c_custkey = uv.custKey
INNER JOIN (
  SELECT DISTINCT d_yearmonthnum, (d_month||','||d_year) as prettyMonthYear
  FROM dwdate
  WHERE d_yearmonthnum >= 199810) as t ON uv.yearMonthKey = t.d_yearmonthnum
WHERE c.c_custkey <= 3
GROUP BY c.c_name, c.c_mktsegment, t.prettyMonthYear, uv.yearMonthKey
ORDER BY c.c_name, c.c_mktsegment, uv.yearMonthKey  ASC
```

预计此查询需要几分钟才能完成，因为将访问近 38 亿条记录。 查询结果应如下所示：

| c_name             | c_mktsegment | Prettymonthyear | totalrevenue |
| :----------------- | :----------- | :-------------- | :----------- |
| Customer#000000001 | BUILDING     | October,1998    | 3596847.84   |
| Customer#000000001 | BUILDING     | November,1998   | 3776957.04   |
| Customer#000000001 | BUILDING     | December,1998   | 3674480.43   |
| Customer#000000002 | AUTOMOBILE   | October,1998    | 3593281.28   |
| Customer#000000002 | AUTOMOBILE   | November,1998   | 3777930.64   |
| Customer#000000002 | AUTOMOBILE   | December,1998   | 3671834.14   |
| Customer#000000003 | AUTOMOBILE   | October,1998    | 3596234.31   |
| Customer#000000003 | AUTOMOBILE   | November,1998   | 3776715.02   |
| Customer#000000003 | AUTOMOBILE   | December,1998   | 3674360.28   |

## 性能诊断

有一些实用程序可以提供对 Redshift Spectrum 的可见性：

- [EXPLAIN](http://docs.aws.amazon.com/redshift/latest/dg/r_EXPLAIN.html) - 提供查询执行计划，其中包括有关哪些处理被推送到 Spectrum 的信息。计划中包含前缀 S3 的步骤在 Spectrum 上执行；例如，上面查询的计划有一个步骤“S3 Seq Scan clickstream.uservisits_csv10”，表明作为查询执行的一部分，Spectrum 在 S3 上执行扫描。
- [SVL_S3QUERY_SUMMARY](http://docs.aws.amazon.com/redshift/latest/dg/r_SVL_S3QUERY_SUMMARY.html) - 提供存储在此表中的 Redshift Spectrum 查询的统计信息。虽然执行计划提供了成本估计，但该表存储了过去查询运行的实际统计信息。
- [SVL_S3PARTITION](https://docs.aws.amazon.com/redshift/latest/dg/r_SVL_S3PARTITION.html) - 提供有关分段和节点切片级别的 Amazon Redshift Spectrum 分区修剪的详细信息。

1. 对 SVL_S3QUERY_SUMMARY 表运行以下查询：

```sql
select elapsed, s3_scanned_rows, s3_scanned_bytes,
  s3query_returned_rows, s3query_returned_bytes, files, avg_request_parallelism
from svl_s3query_summary
where query = pg_last_query_id()
order by query,segment;
```

查询应返回类似于以下内容的结果：
| elapsed   | s3_scanned_rows | s3_scanned_bytes | s3query_returned_rows | s3query_returned_bytes | files | avg_request_parallelism |
| :-------- | :-------------- | :--------------- | :-------------------- | :--------------------- | :---- | :---------------------- |
| 209773697 | 3758774345      | 6.61358E+11      | 66270117              | 1060321872             | 5040  | 9.77                    |

诊断结果揭示了为什么我们的查询花费了这么长时间。 例如，*s3_scanned_row* 显示查询扫描了近 3.8B 条记录，这是整个数据集。

使用以下查询，您可以看到 Spectrum 扫描所有分区。

```sql
SELECT query, segment,
       MIN(starttime) AS starttime,
       MAX(endtime) AS endtime,
       datediff(ms,MIN(starttime),MAX(endtime)) AS dur_ms,
       MAX(total_partitions) AS total_partitions,
       MAX(qualified_partitions) AS qualified_partitions,
       MAX(assignment) as assignment_type
FROM svl_s3partition
WHERE query=pg_last_query_id()
GROUP BY query, segment;
```

2. 再次运行相同的 Redshift Spectrum 查询，但使用 EXPLAIN
```sql
EXPLAIN
SELECT c.c_name, c.c_mktsegment, t.prettyMonthYear, SUM(uv.adRevenue)
FROM clickstream.uservisits_csv10 as uv
RIGHT OUTER JOIN customer as c ON c.c_custkey = uv.custKey
INNER JOIN (
  SELECT DISTINCT d_yearmonthnum, (d_month||','||d_year) as prettyMonthYear
  FROM dwdate WHERE d_yearmonthnum >= 199810) as t ON uv.yearMonthKey = t.d_yearmonthnum
WHERE c.c_custkey <= 3
GROUP BY c.c_name, c.c_mktsegment, t.prettyMonthYear, uv.yearMonthKey
ORDER BY c.c_name, c.c_mktsegment, uv.yearMonthKey  ASC
```

输出将类似于下面的示例。 此时不用担心了解查询计划的细节。
```
QUERY PLAN
XN Merge  (cost=1141056800867.31..1141056801917.31 rows=420000 width=78)
Merge Key: c.c_name, c.c_mktsegment, uv.yearmonthkey
->  XN Network  (cost=1141056800867.31..1141056801917.31 rows=420000 width=78)
Send to leader
->  XN Sort  (cost=1141056800867.31..1141056801917.31 rows=420000 width=78)
Sort Key: c.c_name, c.c_mktsegment, uv.yearmonthkey
->  XN HashAggregate  (cost=141056760589.25..141056761639.25 rows=420000 width=78)
->  XN Hash Join DS_DIST_ALL_NONE  (cost=37.34..141056596142.86 rows=13155711 width=78)
Hash Cond: ("outer".yearmonthkey = "inner".d_yearmonthnum)
->  XN Hash Join DS_DIST_ALL_NONE  (cost=4.50..141051766084.98 rows=375877435 width=46)
Hash Cond: ("outer".custkey = "inner".c_custkey)
->  XN Partition Loop  (cost=0.00..94063327993.62 rows=3758774345000 width=16)
->  XN Seq Scan PartitionInfo of clickstream.uservisits_csv10 uv  (cost=0.00..10.00 rows=1000 width=0)
->  XN S3 Query Scan uv  (cost=0.00..93969358.62 rows=3758774345 width=16)
->  S3 Seq Scan clickstream.uservisits_csv10 uv location:"s3://redshift-spectrum-datastore-csv10" format:TEXT  (cost=0.00..56381615.17 rows=3758774345 width=16)
Filter: ((custkey <= 3) AND (yearmonthkey >= 199810))
->  XN Hash  (cost=3.75..3.75 rows=300 width=38)
->  XN Seq Scan on customer c  (cost=0.00..3.75 rows=300 width=38)
Filter: (c_custkey <= 3)
->  XN Hash  (cost=32.82..32.82 rows=7 width=36)
->  XN Subquery Scan t  (cost=0.00..32.82 rows=7 width=36)
->  XN Unique  (cost=0.00..32.75 rows=7 width=18)
->  XN Seq Scan on dwdate  (cost=0.00..32.43 rows=64 width=18)
Filter: (d_yearmonthnum >= 199810)
```

要点是查询计划揭示了 Redshift Spectrum 如何在查询中发挥作用。 下面的行表明 Redshift Spectrum 被用作查询执行的一部分来执行扫描。 它还表明未使用分区。 我们将在实验室的下一部分中更详细地探讨这一点。

```
->  S3 Seq Scan clickstream.uservisits_csv10 uv location:"s3://redshift-spectrum-datastore-csv10" format:TEXT  (cost=0.00..56381615.17 rows=3758774345 width=16)
```

## 使用分区优化

在本节中，您将了解分区，以及如何使用它们来提高 Redshift Spectrum 查询的性能。 分区是提高扫描效率的关键手段。 以前，我们运行了glue crawler，它创建了我们的外部表和分区。 导航回 Glue 目录 https://console.aws.amazon.com/glue/home?#catalog:tab=tables。 单击 *uservisits_csv10* 表。 列 *customer* 和 *visityearmonth* 被设置为分区键。

[![img](https://redshift-immersion.workshop.aws/images/crawler_11_clickstream.png)](https://redshift-immersion.workshop.aws/images/crawler_11_clickstream.png)

如果您有兴趣了解如何设置分区的详细信息，请参阅 [文档](http://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-external-tables.html)。 您可以通过导航到以下位置来探索为我们的 Redshift Spectrum 数据集提供服务的 S3 存储桶：

```
https://s3.console.aws.amazon.com/s3/buckets/redshift-immersionday-labs/data/clickstream/uservisits_csv10/
```

整个 38 亿行的数据集被组织成一个大文件的集合，其中每个文件都包含特定客户和一年中特定月份的数据。这允许您按客户和年/月将数据划分为逻辑子集，如上例所示。使用分区，查询引擎可以定位文件的子集：

- 仅针对特定客户
- 仅特定月份的数据
- 特定客户和年/月的组合

请注意，分区的正确选择取决于您的工作负载。应根据您要优化的主要查询和您的数据配置文件来选择分区。对于那些实施他们自己的点击流分析的人来说，像年/月/地区这样的分区方案通常是有意义的。在分区方案中使用 customer 的选择对于有大量客户且每个客户的数据很少的用例并不是最佳选择。本示例中使用的数据集和方案对于多租户广告技术平台或物联网平台等场景是实用的。在这些情况下，有中等数量的客户（租户）和每个客户的大量数据。

1. 通过运行以下查询，观察利用分区对我们的查询的影响。

```sql
SELECT c.c_name, c.c_mktsegment, t.prettyMonthYear, SUM(uv.adRevenue)
FROM clickstream.uservisits_csv10 as uv
RIGHT OUTER JOIN customer as c ON c.c_custkey = uv.customer
INNER JOIN
  (SELECT DISTINCT d_yearmonthnum, (d_month||','||d_year) as prettyMonthYear
   FROM dwdate
   WHERE d_yearmonthnum >= 199810) as t ON uv.yearMonthKey = t.d_yearmonthnum
WHERE c.c_custkey <= 3
GROUP BY c.c_name, c.c_mktsegment, t.prettyMonthYear, uv.yearMonthKey
ORDER BY c.c_name, c.c_mktsegment, uv.yearMonthKey  ASC
```

上一个查询的连接条件已被修改。 我们不使用合成键 *custKey*，而是使用分区键 *customer*，这是我们在数据建模过程中创建的。 这个查询应该比之前的查询运行速度大约“快 2 倍”。

2. 再次运行相同的 Redshift Spectrum 查询，但使用 EXPLAIN。 与以前不同，您应该看到一个 Filter 子句作为 PartitionInfo 扫描的一部分，它指示分区修剪是作为查询计划的一部分执行的：

```
->  XN Seq Scan PartitionInfo of clickstream.uservisits_csv10 uv  (cost=0.00..12.50 rows=334 width=4)
         Filter: ((customer <= 3) AND (subplan 4: (customer = $2)))
```

3. 重新运行 SVL_S3QUERY_SUMMARY 查询：

```sql
select elapsed, s3_scanned_rows, s3_scanned_bytes,
  s3query_returned_rows, s3query_returned_bytes, files, avg_request_parallelism
from svl_s3query_summary
where query = pg_last_query_id()
order by query,segment;
```

您应该观察到以下结果：

| elapsed   | s3_scanned_rows | s3_scanned_bytes | s3query_returned_rows | s3query_returned_bytes | files | avg_request_parallelism |
| :-------- | :-------------- | :--------------- | :-------------------- | :--------------------- | :---- | :---------------------- |
| 113561847 | 1898739653      | 3.34084E+11      | 66270117              | 795241404              | 2520  | 9.71                    |

请注意，*s3_scanned_rows* 表明与之前的查询相比，扫描的行数减半。 这解释了为什么我们的查询运行速度大约快两倍。

结果是由于我们的数据均匀分布在所有客户中，并且通过使用我们的客户分区键查询 6 个客户中的 3 个，数据库引擎能够智能扫描包含客户 1,2 和 3 的数据子集 整个数据集。 然而，扫描仍然非常低效，我们也可以从使用我们的年/月分区键中受益。

4. 运行以下查询：

```sql
SELECT c.c_name, c.c_mktsegment, t.prettyMonthYear, SUM(uv.adRevenue)
FROM clickstream.uservisits_csv10 as uv
RIGHT OUTER JOIN customer as c ON c.c_custkey = uv.customer
INNER JOIN (
  SELECT DISTINCT d_yearmonthnum, (d_month||','||d_year) as prettyMonthYear
  FROM dwdate
  WHERE d_yearmonthnum >= 199810) as t ON uv.visitYearMonth = t.d_yearmonthnum
WHERE c.c_custkey <= 3
GROUP BY c.c_name, c.c_mktsegment, t.prettyMonthYear, uv.yearMonthKey
ORDER BY c.c_name, c.c_mktsegment, uv.yearMonthKey  ASC
```

上一个查询的连接条件已被修改。 我们不使用合成键 *yearMonthKey*，而是使用分区键 *visitYearMonth*。 我们最新的查询同时使用了客户和时间分区，如果您多次运行此查询，您应该会看到执行时间在 8 秒范围内，这是我们原始查询的“22.5 倍改进”！

5. 重新运行 SVL_S3QUERY_SUMMARY 查询：

```sql
select elapsed, s3_scanned_rows, s3_scanned_bytes,
  s3query_returned_rows, s3query_returned_bytes, files, avg_request_parallelism
from svl_s3query_summary
where query = pg_last_query_id()
order by query,segment;
```

查看此查询的统计信息后，您应该观察到 Redshift Spectrum 会扫描并返回计算查询所需的确切行数 (66,270,117)。

| elapsed | s3_scanned_rows | s3_scanned_bytes | s3query_returned_rows | s3query_returned_bytes | files | avg_request_parallelism |
| :------ | :-------------- | :--------------- | :-------------------- | :--------------------- | :---- | :---------------------- |
| 7124877 | 66270117        | 11660676734      | 66270117              | 795241404              | 90    | 5.87                    |

## 存储优化

Redshift Spectrum 通过 Redshift 集群外部的大规模基础设施执行处理。它针对在 S3 上执行大型扫描和聚合进行了优化；事实上，通过适当的优化，Redshift Spectrum 甚至可以在这些类型的工作负载上胜过中小型 Redshift 集群。优化大型扫描和聚合需要考虑两个重要变量：

- `文件大小和计数` - 作为一般规则，对于不可拆分的文件，使用 50-500MB 之间的文件大小，这是 Redshift Spectrum 的最佳选择。但是，对查询进行操作的文件数量与查询可实现的并行度直接相关。文件大小和数量之间存在反比关系：文件越大，相同数据集的文件越少。因此，在优化对象读取性能和可在特定查询上实现的并行量之间存在权衡。大文件最适合大扫描，因为查询可能对足够多的文件进行操作。对于更具选择性且正在运行的文件较少的查询，您可能会发现较小的文件允许更多的并行性。
- `数据格式` Redshift Spectrum 支持各种数据格式。 Parquet 等列格式有时可以通过为某些工作负载提供压缩和更高效的 I/O 来带来显着的性能优势。一般来说，像 Parquet 这样的格式类型应该用于涉及大扫描和高属性选择性的查询工作负载。同样，由于 Parquet 等格式需要比纯文本更多的计算能力来处理，因此需要进行权衡。对于较小数据子集的查询，Parquet 的 I/O 效率优势会减弱。在某些时候，Parquet 的执行速度可能与纯文本相同或更慢。延迟、压缩率以及用户体验和成本之间的权衡应该会推动您做出决定。在大多数情况下，Parquet 等格式是最佳的。

为了帮助说明 Spectrum 如何在这些大型聚合工作负载上执行，让我们考虑在 Redshift Spectrum 上聚合整个 3.7B+ 记录数据集的基本查询，并将其与仅在 Redshift 上运行查询进行比较：

```sql
SELECT uv.custKey, COUNT(uv.custKey)
FROM <your clickstream table> as uv
GROUP BY uv.custKey
ORDER BY uv.custKey ASC
```

由于时间关系，我们不会在实验室中进行这个练习；尽管如此，了解运行此测试的结果还是有帮助的。

对于 Redshift-only 测试用例，点击流数据被加载，并通过 Redshift 的 ANALYZE 命令规定的最佳列压缩编码均匀分布在所有节点上（均匀分布样式）。

Redshift Spectrum 测试用例使用 Parquet 数据格式，其中一个文件包含某个特定客户在一个月内的所有数据；这导致文件大部分在 220-280MB 范围内，实际上，这是此分区方案的最大文件大小。如果您使用提供的其他数据集运行测试，您将看到此数据格式和大小是最佳的，并且将超过其他数据集 60 倍以上。

请注意，不应笼统地应用所提供的量化，因为性能差异会因场景而异。相反，请注意 Spectrum 可能产生性能优势的测试策略、证据和工作负载的特征。

下图比较了两种场景的查询执行时间。结果表明，您需要为 12 X DC1.Large 节点付费，才能获得与在此特定场景中使用 Spectrum 并支持小型 Redshift 集群的性能相当的性能。另外，请注意上图中 Spectrum 平台的性能。如果查询涉及从更多文件聚合数据，我们也会看到性能的持续线性改进。

## Predicate Pushdown

在上一节中，我们了解到 Spectrum 擅长执行大型聚合。在本节中，我们将试验将更多工作下放到 Redshift Spectrum 的结果。

1. 运行以下查询。多次运行此查询后，您应该观察到执行时间在 4 秒范围内。
```sql
SELECT c.c_name, c.c_mktsegment, t.prettyMonthYear, uv.totalRevenue
FROM (
  SELECT customer, visitYearMonth, SUM(adRevenue) as totalRevenue
  FROM clickstream.uservisits_parquet1
  WHERE customer <= 3 and visitYearMonth >= 199810
  GROUP BY  customer, visitYearMonth) as uv
RIGHT OUTER JOIN customer as c ON c.c_custkey = uv.customer
INNER JOIN (
  SELECT DISTINCT d_yearmonthnum, (d_month||','||d_year) as prettyMonthYear
  FROM dwdate WHERE d_yearmonthnum >= 199810) as t ON uv.visitYearMonth = t.d_yearmonthnum
ORDER BY c.c_name, c.c_mktsegment, uv.visitYearMonth ASC;
```

这个查询在几个方面改进了我们之前的查询。

- 我们查询的是 clickstream.uservisits_parquet1 表，而不是 clickstream.uservisits_csv10。这两个表包含相同的数据集，但它们的处理方式不同。表 clickstream.uservisits_parquet1 包含镶木地板格式的数据。 Parquet 是一种列式格式，通过对查询选择的属性进行压缩和高效检索，为分析工作负载带来 I/O 优势。此外，“1”与“10”后缀表示每个分区的所有数据都存储在单个文件中，而不是像 CSV 数据集那样有 10 个文件。后一种情况涉及处理大型扫描和聚合的开销较少。
- 聚合工作已下推至 Redshift Spectrum。之前分析查询计划时，我们观察到 Spectrum 用于扫描。当您分析上述查询时，您会看到聚合也在 Spectrum 层执行。

2. 重新运行 SVL_S3QUERY_SUMMARY 查询：

```sql
select elapsed, s3_scanned_rows, s3_scanned_bytes,
  s3query_returned_rows, s3query_returned_bytes, files, avg_request_parallelism
from svl_s3query_summary
where query = pg_last_query_id()
order by query,segment;
```

您将获得以下结果：
| elapsed | s3_scanned_rows | s3_scanned_bytes | s3query_returned_rows | s3query_returned_bytes | files | avg_request_parallelism |
| :------ | :-------------- | :--------------- | :-------------------- | :--------------------- | :---- | :---------------------- |
| 1990509 | 66270117        | 531159030        | 9                     | 72                     | 9     | 0.88                    |

统计数据揭示了一些性能改进的来源：

- 即使由于压缩而扫描的行数相同，扫描的字节数也会减少。
- 返回的行数从 ~66.3M 减少到 9。 这导致 Spectrum 层仅返回 72 个字节，而 795MB。 这是将聚合下推到 Spectrum 层的结果。 我们的数据以日级粒度存储，我们的查询将其滚动到月级。 通过将聚合下推到 Spectrum 队列，我们只需要返回将广告收入聚合到月级别的 9 条记录，以便将它们与所需的维度属性连接起来。

3. 使用 EXPLAIN 再次运行查询：

查询计划应包括 *S3 Aggregate* 步骤，这表明 Spectrum 层卸载了此查询的聚合处理。
```
QUERY PLAN
XN Merge  (cost=1000094008880.16..1000094008880.18 rows=7 width=78)
  Merge Key: c.c_name, c.c_mktsegment, uv.visityearmonth
  ->  XN Network  (cost=1000094008880.16..1000094008880.18 rows=7 width=78)
        Send to leader
        ->  XN Sort  (cost=1000094008880.16..1000094008880.18 rows=7 width=78)
              Sort Key: c.c_name, c.c_mktsegment, uv.visityearmonth
              ->  XN Hash Join DS_DIST_ALL_NONE  (cost=94008878.97..94008880.06 rows=7 width=78)
                    Hash Cond: ("outer".customer = "inner".c_custkey)
                    ->  XN Hash Join DS_DIST_ALL_NONE  (cost=93971378.97..93971379.61 rows=7 width=48)
                          Hash Cond: ("outer".visityearmonth = "inner".d_yearmonthnum)
                          ->  XN Subquery Scan uv  (cost=93971346.13..93971346.42 rows=23 width=16)
                                ->  XN HashAggregate  (cost=93971346.13..93971346.19 rows=23 width=16)
                                      ->  XN Partition Loop  (cost=93969358.63..93970506.13 rows=112000 width=16)
                                            ->  XN Seq Scan PartitionInfo of clickstream.uservisits_parquet1  (cost=0.00..17.50 rows=112 width=8)
                                                  Filter: ((customer <= 3) AND (visityearmonth >= 199810) AND (visityearmonth >= 199810))
                                            ->  XN S3 Query Scan uservisits_parquet1  (cost=46984679.32..46984689.32 rows=1000 width=8)
                                                  ->  S3 Aggregate  (cost=46984679.32..46984679.32 rows=1000 width=8)
                                                        ->  S3 Seq Scan clickstream.uservisits_parquet1 location:"s3://redshift-spectrum-datastore-parquet1" format:PARQUET  (cost=0.00..37587743.45 rows=3758774345 width=8)
                          ->  XN Hash  (cost=32.82..32.82 rows=7 width=36)
                                ->  XN Subquery Scan t  (cost=0.00..32.82 rows=7 width=36)
                                      ->  XN Unique  (cost=0.00..32.75 rows=7 width=18)
                                            ->  XN Seq Scan on dwdate  (cost=0.00..32.43 rows=64 width=18)
                                                  Filter: (d_yearmonthnum >= 199810)
                    ->  XN Hash  (cost=30000.00..30000.00 rows=3000000 width=38)
                          ->  XN Seq Scan on customer c  (cost=0.00..30000.00 rows=3000000 width=38)
```

## Redshift Spectrum请求加速器

Redshift Spectrum 请求加速器 (SRA) 自动且透明地启用，显着提高了对 Amazon S3 中数据的查询性能。

1. 对 1992 年的数据运行以下查询。此查询应在 10 秒内运行。 修改过滤器并在 1993 年再次运行它。

```sql
SELECT c.c_name, c.c_mktsegment, uv.visitYearMonth, uv.totalRevenue
FROM (
  SELECT customer, visitYearMonth, SUM(adRevenue) as totalRevenue
  FROM clickstream.uservisits_parquet1
  WHERE customer <= 3 and visitYearMonth between 199201 and 199212
  GROUP BY  customer, visitYearMonth) as uv
RIGHT OUTER JOIN customer as c ON c.c_custkey = uv.customer
ORDER BY c.c_name, c.c_mktsegment, uv.visitYearMonth ASC;
```

2. 现在让我们对 1992 年到 1994 年三年内的数据运行查询。此查询将访问大约 3 倍的数据，但花费的时间不到 3 倍。 这是因为 SRA 缓存了 1992 年和 1993 年的 Spectrum 结果。这个查询可以重用它而不是再次处理它们。
3. 通过运行查询再次检查 SVL_S3QUERY_SUMMARY：

```sql
select query, starttime, elapsed, s3_scanned_rows, s3_scanned_bytes,
  s3query_returned_rows, s3query_returned_bytes, files, avg_request_parallelism
from svl_s3query_summary
order by query desc,segment;
```

您应该观察以下结果。 请注意，此查询扫描了近 1.1B 条记录。
| query  | starttime                  | elapsed | s3_scanned_rows | s3_scanned_bytes | s3query_returned_rows | s3query_returned_bytes | files | avg_request_parallelism |
| :----- | :------------------------- | :------ | :-------------- | :--------------- | :-------------------- | :--------------------- | :---- | :---------------------- |
| 207376 | 2019-11-06 21:16:54.594125 | 2303302 | 1102046642      | 8848259230       | 299                   | 117806                 | 144   | 5.2                     |
| 207325 | 2019-11-06 21:12:53.917315 | 923104  | 265401542       | 2130891630       | 72                    | 28368                  | 36    | 5.3                     |
| 207312 | 2019-11-06 21:12:22.648347 | 1616744 | 265426602       | 2131090671       | 72                    | 28368                  | 36    | 4.9                     |

需要处理大量数据，但查询运行时间不到 10 秒。这比我们在实验室开始时运行的查询要快得多，后者查询相同的数据集。性能上的差异是我们对数据格式、大小和将聚合工作向下推到 Spectrum 层所做的改进的结果。

## Native Redshift 与 Redshift with Spectrum

此时，您可能会问自己，为什么我不使用 Spectrum？好吧，您仍然可以通过将数据加载到 Redshift 中获得额外的价值。事实上，当我们只在原生 Redshift 中执行时，我们的最后一个查询运行得更快。运行完整测试超出了我们实验室的时间，因此让我们回顾一下测试结果，这些测试结果将使用 Redshift Spectrum 运行最后一个查询与仅使用 Redshift 在各种集群大小上进行比较。

根据经验，不受 I/O 支配并涉及多个连接的查询在本机 Redshift 中得到更好的优化。此外，原生 Redshift 中延迟的可变性要低得多。对于对查询具有严格性能 SLA 的用例，您可能需要考虑专门使用 Redshift 来支持这些查询。

另一方面，当您需要执行大型扫描时，您可以受益于两全其美：以更低的成本获得更高的性能。例如，假设我们需要让我们的业务分析师能够以交互方式发现大量历史数据的洞察力。

## 在你离开之前

如果您使用完集群，请考虑将其停用，以避免为未使用的资源付费。
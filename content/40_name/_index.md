---
title: "4. SPECTRUM现代化改造"
chapter: false
weight: 40
---



# 使用SPECTRUM进行现代化改造

在本实验中，我们将向您展示如何使用 Amazon Redshift 查询 PB 级数据以及 Amazon S3 数据湖中 EB 级数据，而无需加载或移动对象。我们还将演示如何利用视图合并直接附加存储和 S3 Datalake 中的数据来创建单一的事实来源。最后，我们将演示将旧数据老化到 S3 中并仅在 Amazon Redshift 直连存储中维护最新数据的策略。

## 内容

- [开始之前](https://redshift-immersion.workshop.aws/lab4.html#before-you-begin)
- [2016 年发生了什么](https://redshift-immersion.workshop.aws/lab4.html#what-happened-in-2016)
- [回到过去](https://redshift-immersion.workshop.aws/lab4.html#go-back-in-time)
- [创建单一版本的真相](https://redshift-immersion.workshop.aws/lab4.html#create-a-single-version-of-truth)
- [未来计划](https://redshift-immersion.workshop.aws/lab4.html#plan-for-the-future)
- [离开前](https://redshift-immersion.workshop.aws/lab4.html#before-you-leave)

## 在你开始之前

本实验假设您已经启动了 Redshift 集群、配置了客户端工具并加载了 TPC Benchmark 数据。如果您尚未启动集群，请参阅 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html)。如果您尚未加载，请参阅 [实验室 2 - 数据加载](https://redshift-immersion.workshop.aws/lab2.html)。如果您尚未配置客户端工具，请参阅 [实验室 1 - 创建 Redshift 集群：配置客户端工具](https://redshift-immersion.workshop.aws/lab1.html#configure-client-tool)。对于本实验，您需要从 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html) 中收集有关集群的以下信息。

- [Your-Redshift_Role_Arn]

本实验需要在 *US-WEST-2（俄勒冈）* 区域中的集群。

## 架构设计

下面是该实验室中涉及的架构和步骤的概述。[![img](https://redshift-immersion.workshop.aws/images/lab4_architecture.png)](https://redshift-immersion.Workshop.aws/images/lab4_architecture.png)

## 2016 年发生了什么

在本实验的第一部分，我们将执行以下活动：

- 使用 COPY 将 Green 公司 2016 年 1 月的数据加载到 Redshift 直连存储 (DAS) 中。(Load the Green company data for January 2016 into Redshift direct-attached storage (DAS) with COPY.)
- 收集支持/反驳 2016 年 1 月暴风雪对出租车使用影响的证据。(Collect supporting/refuting evidence for the impact of the January, 2016 blizzard on taxi usage.)
- Amazon S3 上的 CSV 数据是按月计算的。这是 S3 控制台的快速屏幕截图：(The CSV data is by month on Amazon S3. Here’s a quick screenshot from the S3 console:)

```
https://s3.console.aws.amazon.com/s3/buckets/us-west-2.serverless-analytics/NYC-Pub/green/?region=us-west-2&tab=overview&prefixSearch=green_tripdata_2016
```

[![img](https://redshift-immersion.workshop.aws/images/green_2016.png)](https://redshift-immersion.workshop.aws/images/green_2016.png)

- 这是来自一个文件的示例数据，可以直接在 S3 控制台中预览：

```
https://s3.console.aws.amazon.com/s3/object/us-west-2.serverless-analytics/NYC-Pub/green/green_tripdata_2013-08.csv?region=us-west-2&tab=select
```

[![img](https://redshift-immersion.workshop.aws/images/green_preview.png)](https://redshift-immersion.workshop.aws/images/green_preview.png)

### 构建你的 DDL

为将驻留在 Redshift 计算节点上的表（也称为 Redshift 直连存储 (DAS) 表）创建模式“workshop_das”和表“workshop_das.green_201601_csv”。

```sql
CREATE SCHEMA workshop_das;

CREATE TABLE workshop_das.green_201601_csv
(
  vendorid                VARCHAR(4),
  pickup_datetime         TIMESTAMP,
  dropoff_datetime        TIMESTAMP,
  store_and_fwd_flag      VARCHAR(1),
  ratecode                INT,
  pickup_longitude        FLOAT4,
  pickup_latitude         FLOAT4,
  dropoff_longitude       FLOAT4,
  dropoff_latitude        FLOAT4,
  passenger_count         INT,
  trip_distance           FLOAT4,
  fare_amount             FLOAT4,
  extra                   FLOAT4,
  mta_tax                 FLOAT4,
  tip_amount              FLOAT4,
  tolls_amount            FLOAT4,
  ehail_fee               FLOAT4,
  improvement_surcharge   FLOAT4,
  total_amount            FLOAT4,
  payment_type            VARCHAR(4),
  trip_type               VARCHAR(4)
)
DISTSTYLE EVEN
SORTKEY (passenger_count,pickup_datetime);
```

### 构建您的复制命令

- 构建您的复制命令以从 Amazon S3 复制数据。 该数据集包含 2016 年 1 月的出租车乘车次数。

确保替换下面脚本中的“[Your-Redshift_Role_Arn]”值。

```sql
COPY workshop_das.green_201601_csv
FROM 's3://us-west-2.serverless-analytics/NYC-Pub/green/green_tripdata_2016-01.csv'
IAM_ROLE '[Your-Redshift_Role_Arn]'
DATEFORMAT 'auto'
IGNOREHEADER 1
DELIMITER ','
IGNOREBLANKLINES
REGION 'us-west-2'
;
```

- 确定您刚刚加载了多少行。

```sql
select count(1) from workshop_das.green_201601_csv;
--1445285
```

### 定位暴雪日
这个月，有一个日期因为暴风雪而乘坐出租车的次数最少。 你能找到那个日期吗？

```sql
SELECT TO_CHAR(pickup_datetime, 'YYYY-MM-DD'),
COUNT(*)
FROM workshop_das.green_201601_csv
GROUP BY 1
ORDER BY 2;
```

## 回到过去

在本实验的下一部分中，我们将执行以下活动：

- 通过为 Redshift Spectrum 创建外部数据库来查询驻留在 S3 上的历史数据。
- 内省历史数据，也许以新颖的方式汇总数据以查看随时间或其他维度的趋势。
- 通过 Redshift Spectrum 特定的查询监控规则 (QMR) 强制合理使用集群。
   - 通过编写过度使用的查询来测试 QMR 设置。

> 注意分区方案是年、月、类型（其中类型是出租车公司）。 这是一个快速截图：

```
https://s3.console.aws.amazon.com/s3/buckets/us-west-2.serverless-analytics/canonical/NY-Pub/
```

[![img](https://redshift-immersion.workshop.aws/images/canonical_year.png)](https://redshift-immersion.workshop.aws/images/canonical_year.png)

```
https://s3.console.aws.amazon.com/s3/buckets/us-west-2.serverless-analytics/canonical/NY-Pub/year%253D2016/
```

[![img](https://redshift-immersion.workshop.aws/images/canonical_month.png)](https://redshift-immersion.workshop.aws/images/canonical_month.png)

```
https://s3.console.aws.amazon.com/s3/buckets/us-west-2.serverless-analytics/canonical/NY-Pub/year%253D2016/month%253D1/
```

[![img](https://redshift-immersion.workshop.aws/images/canonical_type.png)](https://redshift-immersion.workshop.aws/images/canonical_type.png)

### 为 Redshift Spectrum 创建外部架构（和数据库）

由于外部表存储在共享的 Glue 目录中以在 AWS 生态系统中使用，因此可以使用一些不同的工具来构建和维护它们，例如 Athena, Redshift, and Glue。

- 使用 AWS Glue Crawler 在位置 s3://us-west-2.serverless-analytics/canonical/NY-Pub/ 下创建以parquet format格式存储的外部表 adb305.ny_pub。

  1. Navigate to the `Glue Crawler Page`. https://us-west-2.console.aws.amazon.com/glue/home?region=us-west-2#catalog:tab=crawlers

  [![img](https://redshift-immersion.workshop.aws/images/crawler_0.png)](https://redshift-immersion.workshop.aws/images/crawler_0.png)
  2. Click on *Add Crawler*, and enter the crawler name *NYTaxiCrawler* and click *Next*.[![img](https://redshift-immersion.workshop.aws/images/crawler_1.png)](https://redshift-immersion.workshop.aws/images/crawler_1.png)
  3. Select *Data stores* as the source type and click *Next*.[![img](https://redshift-immersion.workshop.aws/images/crawler_2.png)](https://redshift-immersion.workshop.aws/images/crawler_2.png)
  4. Choose *S3* as the data store and the include path of *s3://us-west-2.serverless-analytics/canonical/NY-Pub*[![img](https://redshift-immersion.workshop.aws/images/crawler_3.png)](https://redshift-immersion.workshop.aws/images/crawler_3.png)
  5. *Create an IAM Role* and enter the name AWSGlueServiceRole-*RedshiftImmersion*.
     [![img](https://redshift-immersion.workshop.aws/images/crawler_4.png)](https://redshift-immersion.workshop.aws/images/crawler_4.png)
  6. Select *Run on demand* for the frequency.[![img](https://redshift-immersion.workshop.aws/images/crawler_5.png)](https://redshift-immersion.workshop.aws/images/crawler_5.png)
  7. Click on *Add database* and enter the Database of *spectrumdb*[![img](https://redshift-immersion.workshop.aws/images/crawler_6.png)](https://redshift-immersion.workshop.aws/images/crawler_6.png)
  8. Select all remaining defaults. Once the Crawler has been created, click on *Run Crawler*.[![img](https://redshift-immersion.workshop.aws/images/crawler_7.png)](https://redshift-immersion.workshop.aws/images/crawler_7.png)
  9. Once the Crawler has completed its run, you will see a new table in the Glue Catalog. https://us-west-2.console.aws.amazon.com/glue/home?region=us-west-2#catalog:tab=tables
  [![img](https://redshift-immersion.workshop.aws/images/crawler_8.png)](https://redshift-immersion.workshop.aws/images/crawler_8.png)
  10. Click on the *ny_pub* table, notice the recordCount of 2.87 billion.[![img](https://redshift-immersion.workshop.aws/images/crawler_9.png)](https://redshift-immersion.workshop.aws/images/crawler_9.png)
- Now that the table has been cataloged, switch back to your Redshift query editor and create an external schema `adb305` pointing to your Glue Catalog Database `spectrumdb`

Make sure to replace `[Your-Redshift_Role_Arn]` value in the script below.

```sql
CREATE external SCHEMA adb305
FROM data catalog DATABASE 'spectrumdb'
IAM_ROLE '[Your-Redshift_Role_Arn]'
CREATE external DATABASE if not exists;
```

- 使用外部表而不是直接附加存储 (DAS) 运行上一步中的查询。

```sql
SELECT TO_CHAR(pickup_datetime, 'YYYY-MM-DD'),
COUNT(*)
FROM adb305.ny_pub
WHERE YEAR = 2016 and Month = 01
GROUP BY 1
ORDER BY 1;
```

## 创建 Single Version of Truth

在本实验的下一部分中，我们将演示如何创建一个视图，其中包含通过 Spectrum 和 Redshift 直连存储从 S3 整合的数据。

### 创建视图

创建一个包含 2016 年 1 月 Green 公司 DAS 表和驻留在 S3 上的历史数据的视图，为 Green 数据科学家专门制作一个表。 使用 CTAS 为 Green 公司创建一个包含 2016 年 1 月数据的表。 比较运行时以将其与之前的 COPY 运行时填充。

```sql
CREATE TABLE workshop_das.taxi_201601 AS
SELECT * FROM adb305.ny_pub
WHERE year = 2016 AND month = 1 AND type = 'green';
```

注意：列压缩/编码怎么样？ 请记住，在 CTAS 上，Amazon Redshift 会自动分配压缩编码，如下所示：

- 定义为排序键的列被分配了 RAW 压缩。
- 定义为 BOOLEAN、REAL 或 DOUBLE PRECISION 或 GEOMETRY 数据类型的列被指定为 RAW 压缩。
- 定义为 SMALLINT、INTEGER、BIGINT、DECIMAL、DATE、TIMESTAMP 或 TIMESTAMPTZ 的列被分配了 AZ64 压缩。
- 定义为 CHAR 或 VARCHAR 的列被分配 LZO 压缩。

```
https://docs.aws.amazon.com/redshift/latest/dg/r_CTAS_usage_notes.html
```
```
ANALYZE COMPRESSION workshop_das.taxi_201601
```

Here’s the output in case you want to use it:

| Column           | Encoding | Est_reduction_pct |
| :--------------- | :------- | :---------------- |
| vendorid         | zstd     | 76.85             |
| pickup_datetime  | az64     | 0.00              |
| dropoff_datetime | az64     | 0.00              |
| ratecode         | zstd     | 74.49             |
| passenger_count  | zstd     | 45.69             |
| trip_distance    | zstd     | 73.49             |
| fare_amount      | bytedict | 85.90             |
| total_amount     | zstd     | 75.38             |
| payment_type     | az64     | 0.00              |
| year             | zstd     | 90.51             |
| month            | zstd     | 90.83             |
| type             | zstd     | 89.94             |

### 完成填充表格

将其他出租车公司的 INSERT/SELECT 语句添加到 2016 年 1 月的表中。

```sql
INSERT INTO workshop_das.taxi_201601 (
  SELECT *
  FROM adb305.ny_pub
  WHERE year = 2016 AND month = 1 AND type != 'green');
```

### 删除频谱表中的重叠

现在我们已经加载了 2016 年 1 月的所有数据，我们可以从 Spectrum 表中删除分区，这样直连存储 (DAS) 表和 Spectrum 表之间就没有重叠。

```sql
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='yellow');
```

### 创建一个没有架构绑定的视图

从 `workshop_das.taxi_201601` 创建一个视图 `adb305_view_NYTaxiRides`，允许无缝查询 DAS 和 Spectrum 数据。

```sql
CREATE VIEW adb305_view_NYTaxiRides AS
  SELECT * FROM workshop_das.taxi_201601
  UNION ALL
  SELECT * FROM adb305.ny_pub
WITH NO SCHEMA BINDING;
```

### 这是有效的 SQL 令人惊讶吗？

- 注意 SELECT 和 WHERE 子句中分区列的使用。 Spectrum 表定义中的那些列在哪里？
- 请注意在查询的 Spectrum 部分（相对于 Redshift DAS 部分）中的分区或文件级别应用的过滤器。
- 如果您实际运行查询（而不仅仅是生成解释计划），运行时是否会让您感到惊讶？ 为什么或者为什么不？
```sql
EXPLAIN
SELECT year, month, type, COUNT(*)
FROM adb305_view_NYTaxiRides
WHERE year = 2016 AND month IN (1) AND passenger_count = 4
GROUP BY 1,2,3 ORDER BY 1,2,3;
```
```sql
QUERY PLAN
XN Merge  (cost=1000090025653.20..1000090025653.21 rows=2 width=48)
  Merge Key: derived_col1, derived_col2, derived_col3
  ->  XN Network  (cost=1000090025653.20..1000090025653.21 rows=2 width=48)
        Send to leader
        ->  XN Sort  (cost=1000090025653.20..1000090025653.21 rows=2 width=48)
              Sort Key: derived_col1, derived_col2, derived_col3
              ->  XN HashAggregate  (cost=90025653.19..90025653.19 rows=2 width=48)
                    ->  XN Subquery Scan adb305_view_nytaxirides  (cost=25608.12..90025653.17 rows=2 width=48)
                          ->  XN Append  (cost=25608.12..90025653.15 rows=2 width=38)
                                ->  XN Subquery Scan "*SELECT* 1"  (cost=25608.12..25608.13 rows=1 width=18)
                                      ->  XN HashAggregate  (cost=25608.12..25608.12 rows=1 width=18)
                                            ->  XN Seq Scan on t201601_pqt  (cost=0.00..25292.49 rows=31563 width=18)
                                                  <b>Filter: ((passenger_count = 4) AND ("month" = 1) AND ("year" = 2016))</b>
                                ->  XN Subquery Scan "*SELECT* 2"  (cost=90000045.00..90000045.02 rows=1 width=38)
                                      ->  XN HashAggregate  (cost=90000045.00..90000045.01 rows=1 width=38)
                                            ->  XN Partition Loop  (cost=90000000.00..90000035.00 rows=1000 width=38)
                                                  ->  XN Seq Scan PartitionInfo of adb305.nytaxirides  (cost=0.00..15.00 rows=1 width=30)
                                                       <b> Filter: (("month" = 1) AND ("year" = 2016))</b>
                                                  ->  XN S3 Query Scan nytaxirides  (cost=45000000.00..45000010.00 rows=1000 width=8)
                                                        ->  S3 Aggregate  (cost=45000000.00..45000000.00 rows=1000 width=0)
                                                              ->  S3 Seq Scan adb305.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..37500000.00 rows=3000000000 width=0)
                                                                  <b> Filter: (passenger_count = 4)</b>
```

- 现在通过添加数据在 Spectrum 中的月份来包含 Spectrum 数据

```sql
EXPLAIN
SELECT year, month, type, COUNT(*)
FROM adb305_view_NYTaxiRides
WHERE year = 2016 AND month IN (1,2) AND passenger_count = 4
GROUP BY 1,2,3 ORDER BY 1,2,3;
```
```sql
QUERY PLAN
XN Merge  (cost=1000090029268.92..1000090029268.92 rows=2 width=48)
  Merge Key: derived_col1, derived_col2, derived_col3
  ->  XN Network  (cost=1000090029268.92..1000090029268.92 rows=2 width=48)
        Send to leader
        ->  XN Sort  (cost=1000090029268.92..1000090029268.92 rows=2 width=48)
              Sort Key: derived_col1, derived_col2, derived_col3
              ->  XN HashAggregate  (cost=90029268.90..90029268.90 rows=2 width=48)
                    ->  XN Subquery Scan adb305_view_nytaxirides  (cost=29221.33..90029268.88 rows=2 width=48)
                          ->  XN Append  (cost=29221.33..90029268.86 rows=2 width=38)
                                ->  XN Subquery Scan "*SELECT* 1"  (cost=29221.33..29221.34 rows=1 width=18)
                                      ->  XN HashAggregate  (cost=29221.33..29221.33 rows=1 width=18)
                                            ->  XN Seq Scan on t201601_pqt  (cost=0.00..28905.70 rows=31563 width=18)
                                                 <b> Filter: ((passenger_count = 4) AND ("year" = 2016) AND (("month" = 1) OR ("month" = 2))) </b>
                                ->  XN Subquery Scan "*SELECT* 2"  (cost=90000047.50..90000047.52 rows=1 width=38)
                                      ->  XN HashAggregate  (cost=90000047.50..90000047.51 rows=1 width=38)
                                            ->  XN Partition Loop  (cost=90000000.00..90000037.50 rows=1000 width=38)
                                                  ->  XN Seq Scan PartitionInfo of adb305.nytaxirides  (cost=0.00..17.50 rows=1 width=30)
                                                       <b> Filter: (("year" = 2016) AND (("month" = 1) OR ("month" = 2)))</b>
                                                  ->  XN S3 Query Scan nytaxirides  (cost=45000000.00..45000010.00 rows=1000 width=8)
                                                        ->  S3 Aggregate  (cost=45000000.00..45000000.00 rows=1000 width=0)
                                                              ->  S3 Seq Scan adb305.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..37500000.00 rows=3000000000 width=0)
                                                                  <b> Filter: (passenger_count = 4)</b>
```
```sql
EXPLAIN
SELECT passenger_count, COUNT(*)
FROM adb305.ny_pub
WHERE year = 2016 AND month IN (1,2)
GROUP BY 1 ORDER BY 1;
```
```sql
QUERY PLAN
XN Merge  (cost=1000090005026.64..1000090005027.14 rows=200 width=12)
  <b>Merge Key: nytaxirides.derived_col1</b>
  ->  XN Network  (cost=1000090005026.64..1000090005027.14 rows=200 width=12)
        Send to leader
        ->  XN Sort  (cost=1000090005026.64..1000090005027.14 rows=200 width=12)
              <b>Sort Key: nytaxirides.derived_col1</b>
              ->  XN HashAggregate  (cost=90005018.50..90005019.00 rows=200 width=12)
                    ->  XN Partition Loop  (cost=90000000.00..90004018.50 rows=200000 width=12)
                          ->  XN Seq Scan PartitionInfo of adb305.nytaxirides  (cost=0.00..17.50 rows=1 width=0)
                               Filter: (("year" = 2016) AND (("month" = 1) OR ("month" = 2)))
                          ->  XN S3 Query Scan nytaxirides  (cost=45000000.00..45002000.50 rows=200000 width=12)
                                <b> ->  S3 HashAggregate  (cost=45000000.00..45000000.50 rows=200000 width=4)</b>
                                      ->  S3 Seq Scan adb305.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..30000000.00 rows=3000000000 width=4)
```
```sql
EXPLAIN
SELECT type, COUNT(*)
FROM adb305.ny_pub
WHERE year = 2016 AND month IN (1,2)
GROUP BY 1 ORDER BY 1 ;
```
```sql
QUERY PLAN
XN Merge  (cost=1000075000042.52..1000075000042.52 rows=1 width=30)
  <b>Merge Key: nytaxirides."type"</b>
  ->  XN Network  (cost=1000075000042.52..1000075000042.52 rows=1 width=30)
        Send to leader
        ->  XN Sort  (cost=1000075000042.52..1000075000042.52 rows=1 width=30)
              <b>Sort Key: nytaxirides."type"</b>
              ->  XN HashAggregate  (cost=75000042.50..75000042.51 rows=1 width=30)
                    ->  XN Partition Loop  (cost=75000000.00..75000037.50 rows=1000 width=30)
                          ->  XN Seq Scan PartitionInfo of adb305.nytaxirides  (cost=0.00..17.50 rows=1 width=22)
                               Filter: (("year" = 2016) AND (("month" = 1) OR ("month" = 2)))
                          ->  XN S3 Query Scan nytaxirides  (cost=37500000.00..37500010.00 rows=1000 width=8)
                              <b>  ->  S3 Aggregate  (cost=37500000.00..37500000.00 rows=1000 width=0)</b>
                                      ->  S3 Seq Scan adb305.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..30000000.00 rows=3000000000 width=0)
```

## 规划未来

在本实验的最后一部分中，我们将通过执行以下步骤，比较在 Redshift 直连存储中维护较新的或 *HOT* 数据以及在 S3 中保留较旧的 *COLD* 数据的不同策略：

- 通过将 2015 年第 4 季度数据添加到 Redshift DAS，允许追踪 5 个季度的报告：
  - 预计我们将要在 3 个月的基础上“淘汰”最早的季度，请构建您的 DAS 表以使其易于维护和查询。
  - 调整您的 Redshift Spectrum 表以排除 2015 年第四季度的数据。
- 制定并执行将 2015 年第四季度数据移至 S3 的计划。
  - 要执行的离散步骤是什么？
  - 必须利用哪些额外的 Redshift 功能？
  - 使用现有 Parquet 数据模拟额外的 Redshift 步骤，淘汰来自 Redshift DAS 的 2015 年第四季度数据，并执行任何需要的步骤以维护单一版本的事实。
- 有多种选择可以实现这一目标。预计我们将要在 3 个月的基础上“淘汰”最早的季度，设计您的 DAS 表以使其易于维护和查询。这样的事情怎么样？

```sql
CREATE OR REPLACE VIEW adb305_view_NYTaxiRides AS
  SELECT * FROM workshop_das.taxi_201504
UNION ALL
  SELECT * FROM workshop_das.taxi_201601
UNION ALL
  SELECT * FROM workshop_das.taxi_201602
UNION ALL
  SELECT * FROM workshop_das.taxi_201603
UNION ALL
  SELECT * FROM workshop_das.taxi_201604
UNION ALL
  SELECT * FROM adb305.ny_pub
WITH NO SCHEMA BINDING;
```

- 或者类似的东西？ Redshift 中的 Bulk DELETE-s 实际上非常快（对 VACUUM 有一次个位数的分钟时间），所以这也是一个有效的配置：

```sql
CREATE OR REPLACE VIEW adb305_view_NYTaxiRides AS
   SELECT * FROM workshop_das.taxi_current
UNION ALL
  SELECT * FROM adb305.ny_pub
WITH NO SCHEMA BINDING;
```

- 如果需要，还可以使用 COPY 从 Parquet 数据填充 Redshift DAS 表。 注意：当我们创建 Parquet 数据时，这将突出显示数据设计

> COPY with Parquet 目前不包括将分区列指定为填充目标 Redshift DAS 表的源的方法。 当前的预期是，由于在 S3 上将分区数据存储为实际列没有开销（性能方面）且成本很低，因此客户也将存储分区列数据。

- 我们将展示如何处理未遵循此模式的场景。 在此示例中使用单表选项

```sql
CREATE TABLE workshop_das.taxi_current
DISTSTYLE EVEN
SORTKEY(year, month, type) AS
SELECT * FROM adb305.ny_pub WHERE 1 = 0;
```

- 并且，创建一个不包含 Redshift Spectrum 表中的分区列的辅助表。

```sql
CREATE TABLE workshop_das.taxi_loader AS
  SELECT vendorid, pickup_datetime, dropoff_datetime, ratecode, passenger_count,
  	trip_distance, fare_amount, total_amount, payment_type
  FROM workshop_das.taxi_current
  WHERE 1 = 0;
```

### Parquet 副本继续

- 人口可以很容易地编写； 还有一些可以遵循的不同模式。 下面是一个脚本，它为“type=green”的每个分区发出单独的复制命令。 完成后，需要将单独的脚本用于其他 `type` 分区。

确保替换下面脚本中的“[Your-Redshift_Role_Arn]”值。

```sql
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2015/month=10/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2015/month=11/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2015/month=12/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=1/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=2/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=3/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=4/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=5/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=6/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=7/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=8/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=9/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=10/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=11/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=12/type=green' IAM_ROLE '[Your-Redshift_Role_Arn]' FORMAT AS PARQUET;
INSERT INTO workshop_das.taxi_current
  SELECT *, DATE_PART(year,pickup_datetime), DATE_PART(month,pickup_datetime), 'green'
  FROM workshop_das.taxi_loader;
TRUNCATE workshop_das.taxi_loader;
```

### Redshift Spectrum 当然也可以用来填充表。

```sql
DROP TABLE IF EXISTS workshop_das.taxi_201601;
CREATE TABLE workshop_das.taxi_201601 AS SELECT * FROM adb305.ny_pub WHERE year = 2016 AND month IN (1,2,3);
CREATE TABLE workshop_das.taxi_201602 AS SELECT * FROM adb305.ny_pub WHERE year = 2016 AND month IN (4,5,6);
CREATE TABLE workshop_das.taxi_201603 AS SELECT * FROM adb305.ny_pub WHERE year = 2016 AND month IN (7,8,9);
CREATE TABLE workshop_das.taxi_201604 AS SELECT * FROM adb305.ny_pub WHERE year = 2016 AND month IN (10,11,12);
```

### 调整您的 Redshift Spectrum 表以排除 2015 年第四季度的数据

```sql
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=10, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=10, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=10, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=11, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=11, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=11, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=12, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=12, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2015, month=12, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=1, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=2, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=2, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=2, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=3, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=3, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=3, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=4, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=4, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=4, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=5, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=5, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=5, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=6, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=6, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=6, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=7, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=7, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=7, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=8, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=8, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=8, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=9, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=9, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=9, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=10, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=10, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=10, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=11, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=11, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=11, type='green');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=12, type='yellow');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=12, type='fhv');
ALTER TABLE adb305.ny_pub DROP PARTITION(year=2016, month=12, type='green');
```

- 现在，无论采用何种方法，Redshift DAS 中都有一个涵盖过去 5 个季度的视图，以及 Redshift Spectrum 上的所有时间，对视图用户完全透明。 “淘汰”2015 年第四季度数据的步骤是什么？
   1. 将 Redshift DAS 表中的数据副本放入 S3。 命令是什么？
      - 卸载
   2. 使用 Redshift Spectrum 扩展 Redshift Spectrum 表以涵盖 2015 年第四季度的数据。
      - 添加分区。
   3. 从 Redshift DAS 表中删除数据：
      - DELETE 或 DROP TABLE（取决于实现）。

## 在你离开之前

如果您使用完集群，请考虑将其停用，以避免为未使用的资源付费。
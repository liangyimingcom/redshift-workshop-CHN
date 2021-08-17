---
title: "3. 表设计和查询调优"
chapter: false
weight: 30
---



# 表设计和查询调优

在本实验中，您将分析压缩、反规范化、分布和排序对 Redshift 查询性能的影响。

## 内容

- [开始之前](https://redshift-immersion.workshop.aws/lab3.html#before-you-begin)
- [结果集缓存和执行计划重用](https://redshift-immersion.workshop.aws/lab3.html#result-set-caching-and-execution-plan-reuse)
- [选择性过滤](https://redshift-immersion.workshop.aws/lab3.html#selective-filtering)
- [压缩](https://redshift-immersion.workshop.aws/lab3.html#compression)
- [加入策略](https://redshift-immersion.workshop.aws/lab3.html#join-strategies)
- [离开前](https://redshift-immersion.workshop.aws/lab3.html#before-you-leave)

## 开始之前

本实验假设您已经启动了 Redshift 集群、配置了客户端工具并加载了 TPC Benchmark 数据。如果您尚未启动集群，请参阅 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html)。如果您尚未加载，请参阅 [实验室 2 - 数据加载](https://redshift-immersion.workshop.aws/lab2.html)。如果您尚未配置客户端工具，请参阅 [实验室 1 - 创建 Redshift 集群：配置客户端工具](https://redshift-immersion.workshop.aws/lab1.html#configure-client-tool)。

## 结果集缓存和执行计划重用

查询编辑器仅运行可在 10 分钟内完成的简短查询。 查询结果集分页为每页 100 行。

当 Redshift 知道基础表中的数据没有改变时，它使结果集缓存能够加快数据的检索。 当仅查询的谓词发生更改时，它还可以重用已编译的查询计划。

1. 执行以下查询，注意查询执行时间。 由于这是此查询的第一次执行，Redshift 将需要编译查询并缓存结果集。
```sql
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM customer c
JOIN orders o on c_custkey = o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

2. 再次执行相同的查询并记下查询执行时间。 在第二次执行中，redshift 将利用结果集缓存并立即返回。
```sql
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM customer c
JOIN orders o on c_custkey = o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

3. 更新表中的数据并再次运行查询。 当基础表中的数据发生更改时，Redshift 将意识到更改并使与查询关联的结果集缓存无效。 请注意，执行时间不如步骤 2 快，但比步骤 1 快，因为虽然它不能重用缓存，但可以重用已编译的计划。

```sql
UPDATE customer
SET c_mktsegment = c_mktsegment
WHERE c_mktsegment = 'MACHINERY';

VACUUM DELETE ONLY customer;

SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM customer c
JOIN orders o on c_custkey = o_custkey
GROUP BY c_mktsegment, o_orderpriority;
```

4. 使用谓词执行新查询并记下查询执行时间。 由于这是此查询的第一次执行，Redshift 将需要编译查询并缓存结果集。

```sql
SELECT c_mktsegment, count(1)
FROM Customer c
WHERE c_mktsegment = 'MACHINERY'
GROUP BY c_mktsegment;
```

5. 使用稍微不同的谓词执行查询，并注意执行时间比先前执行快，即使扫描和聚合的数据量非常相似。 这种行为是由于重新使用了编译缓存，因为只有谓词发生了变化。 这种类型的模式对于 BI 报告是典型的，其中 SQL 模式与检索与不同谓词关联的数据的不同用户保持一致。

```sql
SELECT c_mktsegment, count(1)
FROM customer c
WHERE c_mktsegment = 'BUILDING'
GROUP BY c_mktsegment;
```

6. 在本实验的其余部分，关闭结果集缓存以确保运行时代表临时用户查询。

确保替换下面脚本中的“[Your-Redshift_User]”值。

```sql
ALTER USER [Your-Redshift_User] set enable_result_cache_for_session to false;
```

## 选择性过滤

Redshift 利用区域映射，它允许优化器在知道过滤条件不匹配时跳过读取数据块。 在 *orders* 表的情况下，因为我们在 o_order_date 上定义了一个排序键，利用该字段作为谓词的查询将返回得更快。

7. 执行以下查询两次，注意第二次执行的执行时间。 第一次执行是为了确保计划被编译。 第二个更能代表end_User体验。

```sql
SELECT count(1), sum(o_totalprice)
FROM orders
WHERE o_orderdate between '1992-07-05' and '1992-07-07';

SELECT count(1), sum(o_totalprice)
FROM orders
WHERE o_orderdate between '1992-07-05' and '1992-07-07';
```

8. 执行以下查询两次，注意第二次执行的执行时间。 同样，第一个查询是确保计划被编译。 注意：与上一步中使用的查询相比，此查询具有不同的过滤条件，但扫描的数据行数相对相同。

```sql
SELECT count(1), sum(o_totalprice)
FROM orders
where o_orderkey < 600001;

SELECT count(1), sum(o_totalprice)
FROM orders
where o_orderkey < 600001;
```

9. 执行以下命令以比较每个查询的执行时间。 您会注意到第二个查询比上一步中的查询花费的时间长得多，即使聚合的行数相似。 这是因为第一个查询能够利用表上定义的排序键 (o_orderdate)。

```SQL
SELECT query, TRIM(querytxt) as SQL, starttime, endtime, DATEDIFF(microsecs, starttime, endtime) AS duration
FROM STL_QUERY
WHERE TRIM(querytxt) like '%orders%'
ORDER BY starttime DESC
LIMIT 4;
```

## 压缩

Redshift 处理大量数据。 为了优化 Redshift 工作负载，关键原则之一是减少存储的数据量。 Redshift 不是处理包含不同类型和函数的值的整行数据，而是以柱状方式运行。 这提供了实现可以对可以独立压缩的单列数据进行操作的算法的机会。

10. 如果您参考 [LAB 2 - 数据加载](https://redshift-immersion.workshop.aws/lab2.html)，*lineitem* 表的定义没有任何指定的压缩编码。 相反，在加载数据时，会使用默认值自动应用编码，因为在 COPY 语句中使用了 COMPUPDATE PRESET 子句。 执行以下查询以确定用于 *lineitem* 表的压缩。
```sql
SELECT tablename, "column", encoding
FROM pg_table_def
WHERE schemaname = 'public' AND tablename = 'lineitem'
```

11. 创建 *lineitem* 表的副本，将每列的 ENCODING 设置为 RAW 并使用 lineitem 数据加载该表。

```sql
DROP TABLE IF EXISTS lineitem_v1;
CREATE TABLE lineitem_v1 (
  L_ORDERKEY bigint NOT NULL ENCODE RAW       ,
  L_PARTKEY bigint ENCODE RAW                 ,
  L_SUPPKEY bigint ENCODE RAW                 ,
  L_LINENUMBER integer NOT NULL ENCODE RAW    ,
  L_QUANTITY decimal(18,4) ENCODE RAW         ,
  L_EXTENDEDPRICE decimal(18,4) ENCODE RAW    ,
  L_DISCOUNT decimal(18,4) ENCODE RAW         ,
  L_TAX decimal(18,4) ENCODE RAW              ,
  L_RETURNFLAG varchar(1) ENCODE RAW          ,
  L_LINESTATUS varchar(1) ENCODE RAW          ,
  L_SHIPDATE date ENCODE RAW                  ,
  L_COMMITDATE date ENCODE RAW                ,
  L_RECEIPTDATE date ENCODE RAW               ,
  L_SHIPINSTRUCT varchar(25) ENCODE RAW       ,
  L_SHIPMODE varchar(10) ENCODE RAW           ,
  L_COMMENT varchar(44) ENCODE RAW
)
distkey (L_ORDERKEY)
sortkey (L_RECEIPTDATE);

INSERT INTO lineitem_v1
SELECT * FROM lineitem;

ANALYZE lineitem_v1;
```
12. Redshift 提供了 ANALYZE COMPRESSION 命令。 此命令将确定每列的编码，这将产生最大的压缩。 对刚刚加载的表执行 ANALYZE COMPRESSION 命令。 记下结果并将它们与步骤 12 的结果进行比较。

```sql
ANALYZE COMPRESSION lineitem_v1;
```

> 注意：虽然大多数列具有相同的编码，但如果更改编码，某些列将获得更好的压缩。

13. 分析这些表的存储空间，有无压缩。 该表按列存储使用的存储量（以 MB 为单位）。 与第一个表相比，您应该看到第二个表的存储节省了大约 70%。 此查询为您提供每个表的每列存储要求，然后是该表的总存储量（在每一行重复相同）。
```sql
SELECT
  CAST(d.attname AS CHAR(50)),
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) = 'lineitem'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb,
  SUM(CASE WHEN CAST(d.relname AS CHAR(50)) = 'lineitem_v1'
THEN b.size_in_mb ELSE 0 END) AS size_in_mb_v1,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) = 'lineitem'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb,
  SUM(SUM(CASE WHEN CAST(d.relname AS CHAR(50)) = 'lineitem_v1'
THEN b.size_in_mb ELSE 0 END)) OVER () AS total_mb_v1
FROM (
  SELECT relname, attname, attnum - 1 as colid
  FROM pg_class t
  INNER JOIN pg_attribute a ON a.attrelid = t.oid
  WHERE t.relname LIKE 'lineitem%') d
INNER JOIN (
  SELECT name, col, MAX(blocknum) AS size_in_mb
  FROM stv_blocklist b
  INNER JOIN stv_tbl_perm p ON b.tbl=p.id
  GROUP BY name, col) b
ON d.relname = b.name AND d.colid = b.col
GROUP BY d.attname
ORDER BY d.attname;
```

## 加入策略

因为或者 Redshift 的分布式架构，为了处理连接在一起的数据，数据可能必须从一个节点广播到另一个节点。 分析查询的解释计划以确定正在使用哪些连接策略以及如何改进它很重要。

14. 对以下查询执行 EXPLAIN。 当这些表在 [LAB 2 - 数据加载](https://redshift-immersion.workshop.aws/lab2.html) 中加载时，*customer* 表的 DISTSTYLE 设置为 ALL。 对于相对较小的维度表，ALL 分布是一种很好的做法。 这导致“DS_DIST_ALL_NONE”的连接策略和相对较低的成本。 *orders* 和 *lineitem* 表的 DISTKEY 是 *orderkey*。 由于这两个表分布在同一个键上，因此数据位于同一位置，并且可以利用“DS_DIST_NONE”的连接策略。

```sql
EXPLAIN
SELECT c_mktsegment,COUNT(o_orderkey) AS orders_count, sum(l_quantity) as quantity, sum (l_extendedprice) as extendedprice
FROM lineitem
JOIN orders on l_orderkey = o_orderkey
JOIN customer c on o_custkey = c_custkey
WHERE l_commitdate between '1992-01-01T00:00:00Z' and '1992-12-31T00:00:00Z'
GROUP BY c_mktsegment;
```
```sql
XN HashAggregate  (cost=77743573.02..77743573.06 rows=5 width=43)
  ->  XN Hash Join DS_DIST_ALL_NONE  (cost=1137500.00..77351933.42 rows=39163960 width=43)
        Hash Cond: ("outer".o_custkey = "inner".c_custkey)
        ->  XN Hash Join DS_DIST_NONE  (cost=950000.00..70800289.92 rows=39163960 width=39)
              Hash Cond: ("outer".l_orderkey = "inner".o_orderkey)
              ->  XN Seq Scan on lineitem  (cost=0.00..8985568.32 rows=79308960 width=31)
                    Filter: ((l_commitdate <= '1992-12-31'::date) AND (l_commitdate >= '1992-01-01'::date))
              ->  XN Hash  (cost=760000.00..760000.00 rows=76000000 width=16)
                    ->  XN Seq Scan on orders  (cost=0.00..760000.00 rows=76000000 width=16)
        ->  XN Hash  (cost=150000.00..150000.00 rows=15000000 width=20)
              ->  XN Seq Scan on customer c  (cost=0.00..150000.00 rows=15000000 width=20)
```

15. 现在执行查询两次，注意第二次执行的执行时间。 第一次执行是为了确保计划被编译。 第二个更能代表end_User体验。

```sql
SELECT c_mktsegment,COUNT(o_orderkey) AS orders_count, sum(l_quantity) as quantity, sum (l_extendedprice) as extendedprice
FROM lineitem
JOIN orders on l_orderkey = o_orderkey
JOIN customer c on o_custkey = c_custkey
WHERE l_commitdate between '1992-01-01T00:00:00Z' and '1992-12-31T00:00:00Z'
GROUP BY c_mktsegment;
```

16. 创建使用 *custkey* 分发的 *customer* 表的新版本。 执行 EXPLAIN 并注意这会导致成本更高的“DS_BCAST_INNER”连接策略。 这是因为 *customer* 或 *orders* 表都位于同一位置，并且必须广播来自内部表的数据才能完成连接。

```sql
DROP TABLE IF EXISTS customer_v1;
CREATE TABLE customer_v1
DISTKEY (c_custkey) as
SELECT * FROM customer;

EXPLAIN
SELECT c_mktsegment,COUNT(o_orderkey) AS orders_count, sum(l_quantity) as quantity, sum (l_extendedprice) as extendedprice
FROM lineitem
JOIN orders on l_orderkey = o_orderkey
JOIN customer_v1 c on o_custkey = c_custkey
WHERE l_commitdate between '1992-01-01T00:00:00Z' and '1992-12-31T00:00:00Z'
GROUP BY c_mktsegment;
```
```sql
XN HashAggregate  (cost=4200077745781.50..4200077745781.54 rows=5 width=43)
  ->  XN Hash Join DS_BCAST_INNER  (cost=1137500.00..4200077353037.66 rows=39274384 width=43)
        Hash Cond: ("outer".o_custkey = "inner".c_custkey)
        ->  XN Hash Join DS_DIST_NONE  (cost=950000.00..70800289.92 rows=39163960 width=39)
              Hash Cond: ("outer".l_orderkey = "inner".o_orderkey)
              ->  XN Seq Scan on lineitem  (cost=0.00..8985568.32 rows=79308960 width=31)
                    Filter: ((l_commitdate <= '1992-12-31'::date) AND (l_commitdate >= '1992-01-01'::date))
              ->  XN Hash  (cost=760000.00..760000.00 rows=76000000 width=16)
                    ->  XN Seq Scan on orders  (cost=0.00..760000.00 rows=76000000 width=16)
        ->  XN Hash  (cost=150000.00..150000.00 rows=15000000 width=20)
              ->  XN Seq Scan on customer_v1 c  (cost=0.00..150000.00 rows=15000000 width=20)
```

17. 现在执行查询两次，注意第二次执行的执行时间。 第一次执行是为了确保计划被编译。 第二个更能代表end_User体验。

```sql
SELECT c_mktsegment,COUNT(o_orderkey) AS orders_count, sum(l_quantity) as quantity, sum (l_extendedprice) as extendedprice
FROM lineitem
JOIN orders on l_orderkey = o_orderkey
JOIN customer_v1 c on o_custkey = c_custkey
WHERE l_commitdate between '1992-01-01T00:00:00Z' and '1992-12-31T00:00:00Z'
GROUP BY c_mktsegment;
```

18. 最后，创建一个新版本的 *orders* 表，该表使用 EVEN 分布进行分布。 执行 EXPLAIN 并注意在将大 *lineitem* 表连接到 *orders* 表时会导致“DS_DIST_INNER”的连接策略，因为它们没有分布在同一个键上。 此外，当将这些结果连接到 *customer* 表时，数据需要广播到节点，如“DS_BCAST_INNER”连接策略所证明的那样。

```sql
DROP TABLE IF EXISTS orders_v1;
CREATE TABLE orders_v1
DISTSTYLE EVEN as
SELECT * FROM orders;

EXPLAIN
SELECT c_mktsegment,COUNT(o_orderkey) AS orders_count, sum(l_quantity) as quantity, sum (l_extendedprice) as extendedprice
FROM lineitem
JOIN orders_v1 on l_orderkey = o_orderkey
JOIN customer_v1 c on o_custkey = c_custkey
WHERE l_commitdate between '1992-01-01T00:00:00Z' and '1992-12-31T00:00:00Z'
GROUP BY c_mktsegment;
```
```sql
XN HashAggregate  (cost=10280077745781.50..10280077745781.54 rows=5 width=43)
  ->  XN Hash Join DS_BCAST_INNER  (cost=1137500.00..10280077353037.66 rows=39274384 width=43)
        Hash Cond: ("outer".o_custkey = "inner".c_custkey)
        ->  XN Hash Join DS_DIST_INNER  (cost=950000.00..6080070800289.92 rows=39163960 width=39)
              Inner Dist Key: orders_v1.o_orderkey
              Hash Cond: ("outer".l_orderkey = "inner".o_orderkey)
              ->  XN Seq Scan on lineitem  (cost=0.00..8985568.32 rows=79308960 width=31)
                    Filter: ((l_commitdate <= '1992-12-31'::date) AND (l_commitdate >= '1992-01-01'::date))
              ->  XN Hash  (cost=760000.00..760000.00 rows=76000000 width=16)
                    ->  XN Seq Scan on orders_v1  (cost=0.00..760000.00 rows=76000000 width=16)
        ->  XN Hash  (cost=150000.00..150000.00 rows=15000000 width=20)
              ->  XN Seq Scan on customer_v1 c  (cost=0.00..150000.00 rows=15000000 width=20)
```

19. 现在执行查询两次，注意第二次执行的执行时间。 第一次执行是为了确保计划被编译。 第二个更能代表end_User体验。

```sql
SELECT c_mktsegment,COUNT(o_orderkey) AS orders_count, sum(l_quantity) as quantity, sum (l_extendedprice) as extendedprice
FROM lineitem
JOIN orders_v1 on l_orderkey = o_orderkey
JOIN customer c on o_custkey = c_custkey
WHERE l_commitdate between '1992-01-01T00:00:00Z' and '1992-12-31T00:00:00Z'
GROUP BY c_mktsegment;
```

## 在你离开之前

如果您使用完集群，请考虑将其停用，以避免为未使用的资源付费。

请务必打开结果集缓存。 替换下面脚本中的“[Your-Redshift_User]”值。

```sql
ALTER USER [Your-Redshift_User] set enable_result_cache_for_session to true;
```

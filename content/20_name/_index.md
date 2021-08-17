---
title: "2. 数据加载"
chapter: false
weight: 20
---



# 数据加载

在本实验中，您将使用基于 TPC Benchmark 数据模型的一组 8 个表。您在 Redshift 集群中创建这些表，并使用存储在 S3 中的示例数据加载这些表。

## 内容

- [开始之前](https://redshift-immersion.workshop.aws/lab2.html#before-you-begin)
- [云编队](https://redshift-immersion.workshop.aws/lab2.html#cloud-formation)
- [创建表格](https://redshift-immersion.workshop.aws/lab2.html#create-tables)
- [加载数据](https://redshift-immersion.workshop.aws/lab2.html#loading-data)
- [表维护 - 分析](https://redshift-immersion.workshop.aws/lab2.html#table-maintenance---analyze)
- [表维护 - VACUUM](https://redshift-immersion.workshop.aws/lab2.html#table-maintenance---vacuum)
- [故障排除负载](https://redshift-immersion.workshop.aws/lab2.html#troubleshooting-loads)
- [离开前](https://redshift-immersion.workshop.aws/lab2.html#before-you-leave)

## 开始之前

本实验假设您已启动 Redshift 集群并配置了客户端工具。如果您尚未配置客户端工具，请参阅 [实验室 1 - 创建 Redshift 集群：配置客户端工具](https://redshift-immersion.workshop.aws/lab1.html#configure-client-tool)。对于本实验，您需要从 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html) 中收集有关集群的以下信息。

- [DatabaseHostName]
- [Your-Redshift_Role_Name]
- [Your-Redshift_Role_Arn]

![image-20210816205212949](/images/image-20210816205212949.png)

![image-20210816205319123](/images/image-20210816205319123.png)

![image-20210816211125728](/images/image-20210816211125728.png)

如上图所示，

- [DatabaseHostName] = redshiftcluster-rhhjinvkhhku.crtunilk1uss.us-west-2.redshift.amazonaws.com
- [Your-Redshift_Role_Arn]=arn:aws:iam::526739712280:role/RedshiftImmersionRole
- [Your-Redshift_Role_Name]=RedshiftImmersionRole
- [Your_AWS_Account_Id] = 526739712280

## Cloud Formation

要*跳过本实验*并使用Cloud Formation将此示例数据加载到“现有集群”中，请使用以下链接。

[![启动](https://redshift-immersion.workshop.aws/images/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?#/stacks/new?stackName=ImmersionLab2&templateURL=https://s3-us-west-2.amazonaws.com/redshift-immersionday-labs/lab2.yaml)

![image-20210816210013740](/images/image-20210816210013740.png)



创建成功后，如下图所示：

![image-20210816210127239](/images/image-20210816210127239.png)



{{% notice warning %}}
实验特别提醒：
1）使用了 Cloud Formation 创建集群后，相当于自动完成了全部手动创建步骤；
2）使用了 Cloud Formation 创建集群后，下面的手动操作步骤不需要重复进行；
{{% /notice %}}

{{% notice note %}}
这些Cloud Formation模板将创建一个 Lambda 函数，该函数将触发异步 Glue Python Shell 脚本。要监控加载过程并诊断任何加载错误，请参阅 Cloudwatch 日志流。

为集群选择区域时，请考虑 *US-WEST-2（俄勒冈）*。虽然大多数这些实验室可以在任何区域完成，但一些实验室查询位于 *US-WEST-2* 的 S3 中的数据。

该模板将使用默认 CIDR 块 0.0.0.0/0，它提供从任何 IP 地址的访问。最好的做法是替换这个应该具有访问权限的 IP 地址范围。出于这些实验的目的，请将其替换为您的 IP 地址 x.x.x.x/32。
{{% /notice %}}


## 创建表

查询编辑器仅运行可在 10 分钟内完成的简短查询。查询结果集分页为每页 100 行。

复制以下 create table 语句以在模拟 TPC Benchmark 数据模型的数据库中创建表。

[![img](https://redshift-immersion.workshop.aws/images/Model.png)](https://redshift-immersion.workshop.aws/images/Model.png)

```
DROP TABLE IF EXISTS partsupp;
DROP TABLE IF EXISTS lineitem;
DROP TABLE IF EXISTS supplier;
DROP TABLE IF EXISTS part;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS customer;
DROP TABLE IF EXISTS nation;
DROP TABLE IF EXISTS region;

CREATE TABLE region (
  R_REGIONKEY bigint NOT NULL,
  R_NAME varchar(25),
  R_COMMENT varchar(152))
diststyle all;

CREATE TABLE nation (
  N_NATIONKEY bigint NOT NULL,
  N_NAME varchar(25),
  N_REGIONKEY bigint,
  N_COMMENT varchar(152))
diststyle all;

create table customer (
  C_CUSTKEY bigint NOT NULL,
  C_NAME varchar(25),
  C_ADDRESS varchar(40),
  C_NATIONKEY bigint,
  C_PHONE varchar(15),
  C_ACCTBAL decimal(18,4),
  C_MKTSEGMENT varchar(10),
  C_COMMENT varchar(117))
diststyle all;

create table orders (
  O_ORDERKEY bigint NOT NULL,
  O_CUSTKEY bigint,
  O_ORDERSTATUS varchar(1),
  O_TOTALPRICE decimal(18,4),
  O_ORDERDATE Date,
  O_ORDERPRIORITY varchar(15),
  O_CLERK varchar(15),
  O_SHIPPRIORITY Integer,
  O_COMMENT varchar(79))
distkey (O_ORDERKEY)
sortkey (O_ORDERDATE);

create table part (
  P_PARTKEY bigint NOT NULL,
  P_NAME varchar(55),
  P_MFGR  varchar(25),
  P_BRAND varchar(10),
  P_TYPE varchar(25),
  P_SIZE integer,
  P_CONTAINER varchar(10),
  P_RETAILPRICE decimal(18,4),
  P_COMMENT varchar(23))
diststyle all;

create table supplier (
  S_SUPPKEY bigint NOT NULL,
  S_NAME varchar(25),
  S_ADDRESS varchar(40),
  S_NATIONKEY bigint,
  S_PHONE varchar(15),
  S_ACCTBAL decimal(18,4),
  S_COMMENT varchar(101))
diststyle all;                                                              

create table lineitem (
  L_ORDERKEY bigint NOT NULL,
  L_PARTKEY bigint,
  L_SUPPKEY bigint,
  L_LINENUMBER integer NOT NULL,
  L_QUANTITY decimal(18,4),
  L_EXTENDEDPRICE decimal(18,4),
  L_DISCOUNT decimal(18,4),
  L_TAX decimal(18,4),
  L_RETURNFLAG varchar(1),
  L_LINESTATUS varchar(1),
  L_SHIPDATE date,
  L_COMMITDATE date,
  L_RECEIPTDATE date,
  L_SHIPINSTRUCT varchar(25),
  L_SHIPMODE varchar(10),
  L_COMMENT varchar(44))
distkey (L_ORDERKEY)
sortkey (L_RECEIPTDATE);

create table partsupp (
  PS_PARTKEY bigint NOT NULL,
  PS_SUPPKEY bigint NOT NULL,
  PS_AVAILQTY integer,
  PS_SUPPLYCOST decimal(18,4),
  PS_COMMENT varchar(199))
diststyle even;
```

## 加载数据中

COPY 命令比使用 INSERT 语句更有效地加载大量数据，并且也更有效地存储数据。 使用单个 COPY 命令从多个文件加载一个表的数据。 然后，Amazon Redshift 会自动并行加载数据。 为方便起见，您将使用的示例数据在公共 Amazon S3 存储桶中可用。 为确保 Redshift 执行压缩分析，请在 COPY 命令中将 COMPUPDATE 参数设置为 ON。

确保替换下面脚本中的“[Your-Redshift_Role_Arn]”值。

```
COPY region FROM 's3://redshift-immersionday-labs/data/region/region.tbl.lzo'
iam_role '[Your-Redshift_Role_Arn]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

COPY nation FROM 's3://redshift-immersionday-labs/data/nation/nation.tbl.'
iam_role '[Your-Redshift_Role_Arn]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy customer from 's3://redshift-immersionday-labs/data/customer/customer.tbl.'
iam_role '[Your-Redshift_Role_Arn]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy orders from 's3://redshift-immersionday-labs/data/orders/orders.tbl.'
iam_role '[Your-Redshift_Role_Arn]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy part from 's3://redshift-immersionday-labs/data/part/part.tbl.'
iam_role '[Your-Redshift_Role_Arn]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy supplier from 's3://redshift-immersionday-labs/data/supplier/supplier.json' manifest
iam_role '[Your-Redshift_Role_Arn]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy lineitem from 's3://redshift-immersionday-labs/data/lineitem-part/'
iam_role '[Your-Redshift_Role_Arn]'
region 'us-west-2' gzip delimiter '|' COMPUPDATE PRESET;

copy partsupp from 's3://redshift-immersionday-labs/data/partsupp/partsupp.tbl.'
iam_role '[Your-Redshift_Role_Arn]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;
```

如果您使用 2 个 ra3.xlplus 集群节点，加载数据的估计时间如下，注意您可以在 redshift 控制台的性能和查询选项卡中检查有关操作的时间信息：
- REGION (5 rows) - 20s
- NATION (25 rows) - 20s
- CUSTOMER (15M rows) – 3m
- ORDERS - (76M rows) - 1m
- PART - (20M rows) - 4m
- SUPPLIER - (1M rows) - 1m
- LINEITEM - (600M rows) - 13m
- PARTSUPPLIER - (80M rows) 3m

> 注意：上述 COPY 语句的一些关键要点。

1. COMPUPDATE PRESET ON 将使用与列数据类型相关的 Amazon Redshift 最佳实践分配压缩，但不分析表中的数据。
2. REGION 表的 COPY 指向特定文件 (region.tbl.lzo)，而其他表的 COPY 指向多个文件的前缀 (lineitem.tbl.)
3. SUPPLIER 表的 COPY 指向一个清单文件 (supplier.json)

## 表维护 - 分析

您应该定期更新查询计划程序用于构建和选择最佳计划的统计元数据。 您可以通过运行 ANALYZE 命令显式分析表。 当您使用 COPY 命令加载数据时，您可以通过将 STATUPDATE 选项设置为 ON 来自动对增量加载的数据执行分析。 加载到空表时，COPY 命令默认执行 ANALYZE 操作。

对 CUSTOMER 表运行 ANALYZE 命令。
```
analyze customer;
```

要找出运行 ANALYZE 命令的时间，您可以查询系统表和视图，例如 STL_QUERY 和 STV_STATEMENTTEXT，并包括对 padb_fetch_sample 的限制。 例如，要找出上次分析 CUSTOMER 表的时间，请运行以下查询：

```
select query, rtrim(querytxt), starttime
from stl_query
where
querytxt like 'padb_fetch_sample%' and
querytxt like '%customer%'
order by query desc;
```

> 注意：ANALYZE 的时间戳将与执行 COPY 命令的时间相关，并且不会有第二个分析语句的条目。 Redshift 知道它不需要运行 ANALYZE 操作，因为表中没有数据发生变化。

## 表维护 - VACUUM

您应该在大量删除或更新之后运行 VACUUM 命令。 为了执行更新，Amazon Redshift 会删除原始行并附加更新后的行，因此每次更新实际上都是一次删除和一次插入。 虽然 Amazon Redshift 最近启用了一项自动定期回收空间的功能，但最好了解如何手动执行此操作。 您可以运行完全真空、仅删除真空或仅排序真空。

捕获 ORDERS 表的初始空间使用情况。
```
select col, count(*)
from stv_blocklist, stv_tbl_perm
where stv_blocklist.tbl = stv_tbl_perm.id and
stv_blocklist.slice = stv_tbl_perm.slice and
stv_tbl_perm.name = 'orders' and
col <= 5
group by col
order by col;
```

| col  | count |
| :--- | :---- |
| 0    | 280   |
| 1    | 248   |
| 2    | 24    |
| 3    | 304   |
| 4    | 312   |
| 5    | 208   |

从 ORDERS 表中删除行。
```
delete orders where o_orderdate between '1997-01-01' and '1998-01-01';
```

通过再次运行以下查询并注意值未更改，确认 Redshift 没有自动回收空间。
```
select col, count(*)
from stv_blocklist, stv_tbl_perm
where stv_blocklist.tbl = stv_tbl_perm.id
and stv_blocklist.slice = stv_tbl_perm.slice
and stv_tbl_perm.name = 'orders' and
col <= 5
group by col
order by col;
```

运行 VACUUM 命令
```
vacuum delete only orders;
```

通过再次运行以下查询并注意值已更改来确认 VACUUM 命令回收了空间。
```
select col, count(*)
from stv_blocklist, stv_tbl_perm
where stv_blocklist.tbl = stv_tbl_perm.id
and stv_blocklist.slice = stv_tbl_perm.slice
and stv_tbl_perm.name = 'orders' and
col <= 5
group by col
order by col;
```

| col  | count |
| :--- | :---- |
| 0    | 244   |
| 1    | 216   |
| 2    | 24    |
| 3    | 264   |
| 4    | 272   |
| 5    | 184   |

> 注意：如果您有一个列数很少但行数非常多的表，三个隐藏的元数据标识列（INSERT_XID、DELETE_XID、ROW_ID）将占用不成比例的表磁盘空间。为了优化隐藏列的压缩，请尽可能在单个复制事务中加载表。如果使用多个单独的 COPY 命令加载表，则 INSERT_XID 列将无法很好地压缩，并且多个真空操作将无法改善 INSERT_XID 的压缩。

## 负载故障排除

有两个 Amazon Redshift 系统表可以帮助排查数据加载问题：

- STL_LOAD_ERRORS
- STL_FILE_SCAN

此外，您无需实际加载表即可验证数据。将 NOLOAD 选项与 COPY 命令一起使用，以确保在运行实际数据加载之前，您的数据文件将加载而没有任何错误。使用 NOLOAD 选项运行 COPY 比加载数据快得多，因为它只解析文件。

让我们尝试使用列不匹配的不同数据文件加载 CUSTOMER 表。

确保替换下面脚本中的“[Your-Redshift_Role_Arn]”值。

```
COPY customer FROM 's3://redshift-immersionday-labs/data/nation/nation.tbl.'
iam_role '[Your-Redshift_Role_Arn]'
region 'us-west-2' lzop delimiter '|' noload;
```

您将收到以下错误。
```
ERROR: Load into table 'customer' failed.  Check 'stl_load_errors' system table for details. [SQL State=XX000]
```

有关详细信息，请查询 STL_LOAD_ERROR 系统表。
```
select * from stl_load_errors;
```

您还可以创建一个返回有关加载错误详细信息的视图。 以下示例将 STL_LOAD_ERRORS 表连接到 STV_TBL_PERM 表，以将表 ID 与实际表名相匹配。
```
create view loadview as
(select distinct tbl, trim(name) as table_name, query, starttime,
trim(filename) as input, line_number, colname, err_code,
trim(err_reason) as reason
from stl_load_errors sl, stv_tbl_perm sp
where sl.tbl = sp.id);

-- Query the LOADVIEW view to isolate the problem.
select * from loadview where table_name='customer';
```

## 在你离开之前

如果您使用完集群，请考虑将其停用，以避免为未使用的资源付费。

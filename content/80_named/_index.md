---
title: "8. ETL/ELT 策略"
chapter: false
weight: 80
---



# ETL/ELT 策略

本实验演示了如何使用“物化视图”、“存储过程”和“查询调度”来实现 ETL/ELT 流程的现代化，从而在 Redshift 中转换数据。

## 内容

- [开始之前](https://redshift-immersion.workshop.aws/lab13.html#before-you-begin)
- [1 实体化视图](https://redshift-immersion.workshop.aws/lab13.html#1-materialized-views)
- [2 存储过程](https://redshift-immersion.workshop.aws/lab13.html#2-stored-procedures)
- [3 查询调度](https://redshift-immersion.workshop.aws/lab13.html#3-query-scheduling)
- [4 整合](https://redshift-immersion.workshop.aws/lab13.html#4-bringing-it-together)
- [离开前](https://redshift-immersion.workshop.aws/lab13.html#before-you-leave)

## 在你开始之前

本实验假设您已经启动了 Redshift 集群、配置了客户端工具并加载了 TPC Benchmark 数据。如果您尚未启动集群，请参阅 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html)。如果您尚未加载，请参阅 [实验室 2 - 数据加载](https://redshift-immersion.workshop.aws/lab2.html)。如果您尚未配置客户端工具，请参阅 [实验室 1 - 创建 Redshift 集群：配置客户端工具](https://redshift-immersion.workshop.aws/lab1.html#configure-client-tool)。对于本实验，您需要从 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html) 中收集有关集群的以下信息。

- [Your-Redshift_Role_Arn]

在开始本实验之前，让我们从 *lineitem* 表中删除一些先前加载的数据。

```sql
delete from lineitem
using orders
where l_orderkey = o_orderkey
and datepart(year, o_orderdate) = 1998 and datepart(month, o_orderdate) = 8;
```

## 1 物化视图

在数据仓库环境中，应用程序通常需要对大型表执行复杂的查询——例如，对包含数十亿行的表执行多表连接和聚合的 SELECT 语句。 就系统资源和计算结果所需的时间而言，处理这些查询可能会很昂贵。 Amazon Redshift 中的物化视图提供了解决这些问题的方法。 物化视图包含一个预先计算的结果集，基于对一个或多个基表的 SQL 查询。 在这里，您将学习如何创建、查询和刷新物化视图。

- 让我们举一个例子，您想按发货数量生成顶级供应商的报告。 这将连接像 *lineitem* 和 *suppliers* 这样的大表并扫描大量数据。 您可以编写如下查询：

```sql
select n_name, s_name, l_shipmode,
  SUM(L_QUANTITY) Total_Qty
from lineitem
join supplier on l_suppkey = s_suppkey
join nation on s_nationkey = n_nationkey
where datepart(year, L_SHIPDATE) > 1997
group by 1,2,3
order by 3 desc
limit 1000;
```

此查询需要时间来执行，并且因为它正在扫描大量数据将使用大量 I/O 和 CPU 资源。考虑一种情况，组织中的多个用户需要获得上述供应商级别的指标。每个人都可能编写类似的繁重查询，这可能是耗时且昂贵的操作。取而代之的是，您可以使用物化视图来存储预先计算的结果，以加速可预测和重复的查询。

Amazon Redshift 提供了一些方法来使物化视图保持最新。您可以配置自动刷新选项以在更新母马的基表时刷新物化视图。自动刷新操作在集群资源可用时运行，以最大限度地减少对其他工作负载的干扰。

- 执行以下查询以创建物化视图，该视图将 *lineitem* 数据聚合到 *supplier* 级别。请注意，AUTO REFRESH 选项设置为 YES，并且我们在 MV 中包含了其他列，以防其他用户可以利用此聚合数据。

```sql
CREATE MATERIALIZED VIEW supplier_shipmode_agg
AUTO REFRESH YES AS
select l_suppkey, l_shipmode, datepart(year, L_SHIPDATE) l_shipyear,
  SUM(L_QUANTITY)	TOTAL_QTY,
  SUM(L_DISCOUNT) TOTAL_DISCOUNT,
  SUM(L_TAX) TOTAL_TAX,
  SUM(L_EXTENDEDPRICE) TOTAL_EXTENDEDPRICE  
from LINEITEM
group by 1,2,3;
```

- 现在执行下面的查询，该查询已被重写以使用物化视图。 请注意查询执行时间的差异。 您可以在几秒钟内获得相同的结果。

```sql
select n_name, s_name, l_shipmode,
  SUM(TOTAL_QTY) Total_Qty
from supplier_shipmode_agg
join supplier on l_suppkey = s_suppkey
join nation on s_nationkey = n_nationkey
where l_shipyear > 1997
group by 1,2,3
order by 3 desc
limit 1000;
```

物化视图的另一个强大功能是“自动”查询重写。 Amazon Redshift 可以自动重写查询以使用物化视图，即使查询未显式引用物化视图也是如此。

- 现在，重新运行引用 *lineitem* 表的“原始”查询，并看到此查询现在执行得更快，因为 Redshift 已重新编写此查询以利用物化视图而不是基表。

```sql
select n_name, s_name, l_shipmode, SUM(L_QUANTITY) Total_Qty
from lineitem
join supplier on l_suppkey = s_suppkey
join nation on s_nationkey = n_nationkey
where datepart(year, L_SHIPDATE) > 1997
group by 1,2,3
order by 3 desc
limit 1000;
```

您可以通过运行解释操作来验证查询重写器是否正在使用 MV：

```sql
explain
select n_name, s_name, l_shipmode, SUM(L_QUANTITY) Total_Qty
from lineitem
join supplier on l_suppkey = s_suppkey
join nation on s_nationkey = n_nationkey
where datepart(year, L_SHIPDATE) > 1997
group by 1,2,3
order by 3 desc
limit 1000;
```

下面的输出显示了在物化视图上成功自动重写和执行查询。[![img](https://redshift-immersion.workshop.aws/images/lab13_explain1.png)](https://redshift-immersion.workshop.aws/images/lab13_explain1.png)

编写可以利用物化视图但不直接引用它的其他查询。 例如，*总扩展价格*（按*地区*）。

## 2 存储过程

存储过程通常用于封装数据转换、数据验证和特定于业务的逻辑的逻辑。 通过将多个 SQL 步骤组合到一个存储过程中，您可以减少应用程序和数据库之间的往返次数。 除了 SELECT 查询之外，存储过程还可以合并数据定义语言 (DDL) 和数据操作语言 (DML)。 存储过程不必返回值。 您可以使用 PL/pgSQL 过程语言（包括循环和条件表达式）来控制逻辑流。

让我们看看如何在 Redshift 中创建和调用存储过程。 这里我们的目标是增量刷新 *lineitem* 数据。 执行以下查询以创建 *lineitem* 临时表：

```sql
create table stage_lineitem (
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
  L_COMMENT varchar(44));
```

- 执行下面的脚本来创建一个存储过程。 此存储过程执行以下任务：

1. truncate staging table 清理旧数据
2. 使用 COPY 命令在 *stage_lineitem* 表中加载数据。
3. 合并现有*lineitem* 表中的更新记录。

确保替换下面脚本中的“[Your-Redshift_Role_Arn]”值。

```sql
CREATE OR REPLACE PROCEDURE lineitem_incremental()
AS $$
BEGIN

truncate stage_lineitem;  

copy stage_lineitem from 's3://redshift-immersionday-labs/data/lineitem-part/l_orderyear=1998/l_ordermonth=8/'
iam_role '[Your-Redshift_Role_Arn]'
region 'us-west-2' gzip delimiter '|' COMPUPDATE PRESET;

delete from lineitem using stage_lineitem
where stage_lineitem.l_orderkey=lineitem.l_orderkey and stage_lineitem.l_linenumber = lineitem.l_linenumber;

insert into lineitem
select * from stage_lineitem;

END;
$$ LANGUAGE plpgsql;
```

在调用此存储过程之前，请使用物化视图捕获度量。 我们将在存储过程加载新数据后比较此值，以演示物化视图自动刷新功能。

```sql
select SUM(TOTAL_QTY) Total_Qty from supplier_shipmode_agg;
```

[![img](https://redshift-immersion.workshop.aws/images/lab13_procedure1.png)](https://redshift-immersion.workshop.aws/images/lab13_procedure1.png)

使用 CALL 语句调用此存储过程。 执行时，它将执行增量加载：

```sql
call lineitem_incremental();
```

1. ## 3 查询调度

   Amazon Redshift 允许您安排 SQL 查询以定期执行。您现在可以定期安排时间敏感或长时间运行的查询、加载或卸载数据、存储过程或刷新物化视图。您可以使用 Amazon Redshift 控制台或 Amazon Redshift Data API 来安排您的 SQL 查询。

   我们将完成的步骤包括：

   - [准备 IAM 权限](https://redshift-immersion.workshop.aws/lab13.html#prepare-iam-permissions)
   - [安排您的查询](https://redshift-immersion.workshop.aws/lab13.html#schedule-your-query)
   - [监控您的查询](https://redshift-immersion.workshop.aws/lab13.html#monitory-your-query)

   ### 准备 IAM 权限

   要安排查询，需要进行以下 AWS Identity and Access Management (IAM) 配置：

   以下步骤可能已由您的实验室管理员完成，如果已完成，您可以跳至 [安排您的查询](https://redshift-immersion.workshop.aws/lab13.html#schedule-your-query)。

   

   1. 必须创建一个 IAM 角色，该角色可由 Redshift Scheduler 服务、Event Bridge 和您的 IAM 原则（用户或角色）承担。此角色将附加到计划中，并且还必须能够检索执行查询所需的 Redshift 凭据。见[1。创建集群：修改调度程序访问的 IAM 角色](https://redshift-immersion.workshop.aws/lab1.html#modify-iam-role-for-scheduler-access)

   2. 您的 IAM 原则（用户或角色）必须修改才能承担上述角色。如果您在 AWS 账户中以管理员身份登录，则您应该已经拥有此权限。如果没有，请添加以下内容：

      确保替换下面脚本中的“[Your-Redshift_Role_Arn]”值。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
        "Sid": "AssumeIAMRole",
        "Effect": "Allow",
        "Action": "sts:AssumeRole",
        "Resource": "[Your-Redshift_Role_Arn]"
        }
    ]
}
```

### 安排您的查询

导航，返回 Redshift 查询编辑器，确保调用存储过程的查询在编辑器中，然后单击 Schedule 按钮。

```
https://console.aws.amazon.com/redshiftv2/home?#query-editor:
call lineitem_incremental();
```

[![img](https://redshift-immersion.workshop.aws/images/lab13_schedule1.png)](https://redshift-immersion.workshop.aws/images/lab13_schedule1.png)

选择您在上一步中创建的 IAM 角色。 此外，选择集群，并提供数据库名称和数据库用户。[![img](https://redshift-immersion.workshop.aws/images/lab13_schedule2.png)](https://redshift-immersion.workshop.aws/images/lab13_schedule2.png)

输入查询名称以及查询文本。

[![img](https://redshift-immersion.workshop.aws/images/lab13_schedule3.png)](https://redshift-immersion.workshop.aws/images/lab13_schedule3.png)

提供重复依据、重复间隔和重复时间的值。 当您选择“Repeat at time (UTC)”时，输入一个比当前时间稍晚的时间，以便您可以观察执行情况。 或者，您可以通过 Amazon SNS 通知启用监控。 如果您尚未为 Redshift 集群启用 SNS，请参阅 [事件订阅](https://redshift-immersion.workshop.aws/lab7.html#event-subscriptions)。 对于此示例，我们可以保留此 *Disabled*。

[![img](https://redshift-immersion.workshop.aws/images/lab13_schedule4.png)](https://redshift-immersion.workshop.aws/images/lab13_schedule4.png)

### 监控您的查询

导航到计划查询选项卡，您可以看到您的查询计划程序已创建，如下所示：

[![img](https://redshift-immersion.workshop.aws/images/lab13_monitor1.png)](https://redshift-immersion.workshop.aws/images/lab13_monitor1.png)

点击计划，按计划时间执行成功后，可以看到状态为“成功”。[![img](https://redshift-immersion.workshop.aws/images/lab13_monitor2.png)](https://redshift-immersion.workshop.aws/images/lab13_monitor2.png)

重新编写存储过程，传入年月参数。 修改您的计划以动态获取当前年份和当前月份，并将它们用作存储过程的参数。

## 4 把它放在一起

让我们看看 Redshift 是否在 *lineitem* 表更改后自动刷新了物化视图。 请注意，物化视图刷新是*异步*的。 对于本实验，在调用 *lineitem_incremental* 过程后，预计大约 5 分钟的数据会刷新：

```sql
select db_name, userid, schema_name, mv_name, status, refresh_type from SVL_MV_REFRESH_STATUS
where mv_name = 'supplier_shipmode_agg';
```

输出显示 MV 已自动增量刷新。[![img](https://redshift-immersion.workshop.aws/images/lab13_procedure2.png)](https://redshift-immersion.workshop.aws/images/lab13_procedure2.png)

在 MV 上运行以下查询并与您之前记录的值进行比较。 您将看到 SUM 已更改，这表明 Redshift 已识别出一个或多个基表中发生的更改，然后将这些更改应用于物化视图。

```sql
select SUM(TOTAL_QTY) Total_Qty from supplier_shipmode_agg;
```

[![img](https://redshift-immersion.workshop.aws/images/lab13_procedure3.png)](https://redshift-immersion.workshop.aws/images/lab13_procedure3.png)

## 在你离开之前

如果您使用完集群，请考虑将其停用，以避免为未使用的资源付费。


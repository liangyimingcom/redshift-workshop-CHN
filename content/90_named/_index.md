---
title: "9. 数据共享"
chapter: false
weight: 90
---



# 数据共享

在数据仓库环境中，您的工作负载通常与各种用户组和分析用例混合在一起。虽然 Redshift 本身具有使用机器学习构建的工作负载管理，但可能存在需要保护*关键工作负载*免受不可预测的用户行为影响的情况。此外，您可能希望*成本分配*分析支出，以便数据的每个消费者都可以对他们产生的数据以及他们使用的计算负责。最后，移动和管理数据的*多个副本*可能很麻烦。对于所有这些用例，可以使用 Redshift 数据共享。本实验演示了如何通过在 2 个 Redshift 集群之间“共享数据”来“隔离工作负载”。

## 内容

- [开始之前](https://redshift-immersion.workshop.aws/lab14.html#before-you-begin)
- [1 创建消费者集群](https://redshift-immersion.workshop.aws/lab14.html#1-create-a-consumer-cluster)
- [2 创建数据共享](https://redshift-immersion.workshop.aws/lab14.html#2-create-a-data-share)
- [3 地图和查询共享数据](https://redshift-immersion.workshop.aws/lab14.html#3-map-and-query-shared-data)
- [离开前](https://redshift-immersion.workshop.aws/lab14.html#before-you-leave)

## 在你开始之前

本实验假设您已经启动了 Redshift 集群、配置了客户端工具并加载了 TPC Benchmark 数据。如果您尚未启动集群，请参阅 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html)。如果您尚未加载，请参阅 [实验室 2 - 数据加载](https://redshift-immersion.workshop.aws/lab2.html)。如果您尚未配置客户端工具，请参阅 [实验室 1 - 创建 Redshift 集群：配置客户端工具](https://redshift-immersion.workshop.aws/lab1.html#configure-client-tool)。对于本实验，您需要从 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html) 中收集有关集群的以下信息。

Redshift 数据共享需要“RA3”节点类型。

它还假定您有权访问已配置的客户端工具。有关将 SQL Workbench/J 配置为客户端工具的更多详细信息，请参阅 [实验室 1 - 创建 Redshift 集群：配置客户端工具](https://redshift-immersion.workshop.aws/lab1.html#configure-client-tool) .作为替代方案，您可以使用 Redshift 提供的不需要安装的在线查询编辑器。

```
https://console.aws.amazon.com/redshift/home?#query:
```

## 1 创建消费者集群

在与包含 TPC 基准数据的集群相同的账户和区域中启动新集群。您可以使用 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html#launch-redshift-cluster) 中的说明。您可以使用与原始集群关联的相同 VPC、子网组、安全组和 IAM 角色。

Redshift 数据共享需要“RA3”节点类型。

捕获有关此新集群的以下信息并使用它来配置您的客户端工具。有关配置客户端工具的更多详细信息，请参阅 [实验室 1 - 创建 Redshift 集群：配置客户端工具](https://redshift-immersion.workshop.aws/lab1.html#configure-client-tool)。

- [Your-Redshift_Hostname]
- [你的 Redshift_Port]
- [您的 Redshift_用户名]
- [Your-Redshift_Password]

创建集群并成功连接后，执行以下命令并捕获“[Consumer-Namespace]”，稍后您将使用它。

```sql
select current_namespace;
```

## 2 创建数据共享

1. 在新窗口中，使用加载的 TPC 基准数据连接到生产者集群并创建包含所有表的数据共享，并将此数据共享的使用权授予您刚刚创建的消费者集群。

将 `[Consumer-Namespace]` 替换为您之前检索到的值。

```sql
CREATE DATASHARE tpc_share SET PUBLICACCESSIBLE TRUE;
ALTER DATASHARE tpc_share ADD SCHEMA public;
ALTER DATASHARE tpc_share ADD ALL TABLES IN SCHEMA public;
GRANT USAGE ON DATASHARE tpc_share TO NAMESPACE '[Consumer-Namespace]';
```

创建数据共享后，您可以使用以下命令验证集群上存在哪些数据共享。

```sql
show datashares;
```

您还可以使用以下命令获取有关数据共享的信息

```sql
select * from SVV_DATASHARES;
```

以下命令显示 datashare 包含的对象。

```sql
select * from svv_datashare_objects;
```

当生产者集群中的公共模式中添加或删除新对象时会发生什么情况，带有 TPC 数据，它会自动反映在消费者集群中吗？

<details style="box-sizing: border-box; display: block; background-color: rgb(238, 238, 238); padding: 4px; margin: 0px; box-shadow: rgb(187, 187, 187) 1px 1px 2px; color: rgb(35, 47, 62); font-family: &quot;Amazon Ember&quot;, &quot;Work Sans&quot;, Helvetica, Tahoma, Geneva, Arial, sans-serif; font-size: 16px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 300; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><summary style="box-sizing: border-box; display: block;">&nbsp;Answer - Click to expand</summary></details>

2. 假设您不想向消费者集群公开客户详细信息，您还可以从数据共享中删除一个表。 执行以下语句：

```sql
ALTER DATASHARE tpc_share REMOVE TABLE public.customer;
```

3. 执行以下命令并捕获“[TPC-Namespace]”，您将在稍后将数据共享映射到您的消费者集群时使用。

```sql
select current_namespace;
```

## 3 映射和查询共享数据

4. 切换回消费者连接，执行以下命令创建一个连接到数据共享的新数据库：

将 `[TPC-Namespace]` 替换为您之前检索到的值。

```sql
CREATE DATABASE tpc FROM DATASHARE tpc_share OF NAMESPACE '[TPC-Namespace]';
```

5. 测试查询您应该有权访问的表和您无权访问的表：

```sql
select * from tpc.public.region limit 100;
select * from tpc.public.customer limit 100;
```



新数据出现在消费者集群中需要多长时间？ 在 TPC 集群 *region* 表中插入一条新记录，看看它在消费者集群中是否可用。

- In the `producer` cluster and execute following statement to insert a row into the region table:

```sql
insert into public.region values (5, 'New_region','Test Region');
```

- In the `consumer` cluster execute the following statement to validate new data is available in region table:

```sql
select * from tpc.public.region
where r_name = 'New_region';
```




6. 除了 [db].[schema].[table] 表示法之外，您还可以将数据共享模式映射到本地模式：

```sql
CREATE EXTERNAL SCHEMA tpc
FROM REDSHIFT DATABASE 'tpc' SCHEMA 'public';
select * from tpc.region limit 100;
select * from tpc.orders limit 100;
```



从消费者集群查询时，使用 TPC 基准数据的生产者集群上的资源利用率是多少？

- Navigate to the Amazon Redshift Console and note the CPU utilization of the producer and consumer clusters.

```
https://console.aws.amazon.com/redshiftv2/home?#dashboard
```



## 在你离开之前

如果您使用完集群，请考虑将其停用，以避免为未使用的资源付费。

---
title: "6. 使用联邦查询 AURORA POSTGRESQL"
chapter: false
weight: 60
---



# 使用联邦查询 AURORA POSTGRESQL

## 内容

- [开始之前](https://redshift-immersion.workshop.aws/lab6.html#before-you-begin)
- [启动 Aurora PostgreSQL 数据库](https://redshift-immersion.workshop.aws/lab6.html#launch-an-aurora-postgresql-db)
- [加载样本数据](https://redshift-immersion.workshop.aws/lab6.html#load-sample-data)
- [设置外部架构](https://redshift-immersion.workshop.aws/lab6.html#setup-external-schema)
- [执行联合查询](https://redshift-immersion.workshop.aws/lab6.html#execute-federated-queries)
- [执行 ETL 流程](https://redshift-immersion.workshop.aws/lab6.html#execute-etl-processes)
- [离开前](https://redshift-immersion.workshop.aws/lab6.html#before-you-leave)

## 在你开始之前

本实验假设您已经启动了 Redshift 集群、配置了客户端工具并加载了 TPC Benchmark 数据。 如果您尚未启动集群，请参阅 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html)。 如果您尚未加载，请参阅 [实验室 2 - 数据加载](https://redshift-immersion.workshop.aws/lab2.html)。 如果您尚未配置客户端工具，请参阅 [实验室 1 - 创建 Redshift 集群：配置客户端工具](https://redshift-immersion.workshop.aws/lab1.html#configure-client-tool)。
## 启动 Aurora PostgreSQL 数据库

导航到 RDS 控制台并启动新的 Amazon Aurora PostgreSQL 数据库。

```
https://console.aws.amazon.com/rds/home?#launch-dbinstance:gdb=false;s3-import=false
```

1. 选择`Amazon Aurora`作为引擎，`PostgreSQL`作为版本，`Serverless`。[![img](https://redshift-immersion.workshop.aws/images/RDS1.png)](https://redshift-immersion.workshop.aws/images/RDS1.png)

2. 滚动到设置并指定数据库集群标识符、主用户名和主密码。
[![img](https://redshift-immersion.workshop.aws/images/RDS2.png)](https://redshift-immersion.workshop.aws/images/RDS2.png)

3. 滚动到其他连接配置并选中“数据 API”选项以启用通过在线查询编辑器进行访问。
![img](https://redshift-immersion.workshop.aws/images/RDS3.png)](https://redshift-immersion.workshop.aws/images/RDS3.png)

默认情况下，RDS 将在您的默认 VPC 中创建一个数据库。为了让 Redshift 集群能够与 RDS 数据库通信，这两个数据库应该具有网络连接。一种选择是选择与 Redshift 集群相同的 VPC 和安全组。如果您希望将数据库配置为在特定 VPC 中启动，请进行适当的更改。

## 加载示例数据

导航到在线查询编辑器并连接到您新启动的数据库。

```
https://console.aws.amazon.com/rds/home?#query-editor:
```

1. 为之前捕获的数据库实例、用户名和密码输入适当的值。 使用值 `postgres` 作为数据库名称。[![img](https://redshift-immersion.workshop.aws/images/RDS4.png)](https://redshift-immersion.workshop.aws/images/RDS4.png)
2. 执行以下脚本来创建表并加载一些示例数据。

```sql
drop table if exists customer;
create table customer (
  C_CUSTKEY bigint NOT NULL PRIMARY KEY,
  C_NAME varchar(25),
  C_ADDRESS varchar(40),
  C_NATIONKEY bigint,
  C_PHONE varchar(15),
  C_ACCTBAL decimal(18,4),
  C_MKTSEGMENT varchar(10),
  C_COMMENT varchar(117),
  C_UPDATETS timestamp);

insert into Customer values
(1, 'Customer#000000001', '1 Main St.', 1, '555-555-5555', 1234, 'BUILDING', 'comment1', current_timestamp),
(2, 'Customer#000000002', '2 Main St.', 2, '555-555-5555', 1235, 'MACHINERY', 'comment2', current_timestamp),
(3, 'Customer#000000003', '3 Main St.', 3, '555-555-5555', 1236, 'AUTOMOBILE', 'comment3', current_timestamp),
(4, 'Customer#000000004', '4 Main St.', 4, '555-555-5555', 1237, 'HOUSEHOLD', 'comment4', current_timestamp),
(5, 'Customer#000000005', '5 Main St.', 5, '555-555-5555', 1238, 'FURNITURE', 'comment5', current_timestamp);
```

## 设置外部架构

1. 通过导航到以下服务并选择与您的 Aurora PostgreSQL 数据库关联的“rds-db-credentials”，确定与您的 RDS 数据库凭证关联的 Secrets ARN。

```
https://console.aws.amazon.com/secretsmanager/home?#/listSecrets
```

2. 使用以下权限创建名为“RedshiftPostgreSQLSecret-lab”的 IAM 策略，并将 [YOUR SECRET ARN] 替换为上述权限。 修改与您的 Redshift 集群关联的角色并将此策略附加到该角色。

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AccessSecret",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetResourcePolicy",
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret",
                "secretsmanager:ListSecretVersionIds"
            ],
            "Resource": "[YOUR SECRET ARN]"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetRandomPassword",
                "secretsmanager:ListSecrets"
            ],
            "Resource": "*"
        }
    ]
}
```

3. 使用查询编辑器应用程序登录您的 Redshift 集群。 执行以下语句，替换 `[YOUR POSTGRES HOST]`、`[YOUR IAM ROLE]` 和 `[YOUR SECRET ARN]` 的值。

```sql
CREATE EXTERNAL SCHEMA postgres
FROM POSTGRES
DATABASE 'postgres'
URI '[YOUR POSTGRES HOST]'
IAM_ROLE '[YOUR IAM ROLE]'
SECRET_ARN '[YOUR SECRET ARN]'
```

## 执行联合查询

此时，您将可以通过 *postgres* 模式访问 PostgreSQL 数据库中的所有表。

1. 从联合客户表中执行一个简单的选择。

```sql
select * From postgres.customer;
```

2. 在联合客户表与本地区域和国家表之间执行连接。

```sql
select c_name, n_name, r_name
From postgres.customer
join nation on c_nationkey = n_nationkey
join region on n_regionkey = r_regionkey
```

## 执行 ETL 过程

您还可以查询联邦表以在 ETL 过程中使用。 传统上，这些表需要在本地暂存以执行变更检测逻辑，但是，通过联合，它们可以被直接查询和连接。

```sql
insert into customer (c_custkey, c_name, c_address, c_nationkey, c_acctbal, c_mktsegment, c_comment)
select p.c_custkey, p.c_name, p.c_address, p.c_nationkey, p.c_acctbal, p.c_mktsegment, p.c_comment
from postgres.customer p
left join customer c on p.c_custkey = c.c_custkey
where c_updatets > current_date and c.c_custkey is null

update customer
set c_custkey = p.c_custkey, c_name = p.c_name, c_address = p.c_address, c_nationkey = p.c_nationkey,
    c_acctbal = p.c_acctbal, c_mktsegment = p.c_mktsegment, c_comment = p.c_comment
from postgres.customer p
where p.c_custkey = customer.c_custkey
and c_updatets > current_date
```

## 在你离开之前

如果您使用完集群，请考虑将其停用，以避免为未使用的资源付费。
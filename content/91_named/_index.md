---
title: "10. 加载和查询半结构化数据"
chapter: false
weight: 91
---



# 加载和查询半结构化数据

在 [8.在 S3 中查询嵌套 JSON](https://redshift-immersion.workshop.aws/lab8.html) 您学习了如何在 S3 中查询半结构化数据。在本实验中，您将学习如何将半结构化 JSON 数据加载到 Redshift 中，并将其加载到具有 SUPER 数据类型的单个列中或使用自动粉碎。您将了解此策略如何帮助您的半结构化数据中的架构不断发展。

##  内容

- [开始之前](https://redshift-immersion.workshop.aws/lab15.html#before-you-begin)
- [加载 JSON 数据](https://redshift-immersion.workshop.aws/lab15.html#load-json-data)
- [取消嵌套数据](https://redshift-immersion.workshop.aws/lab15.html#unnest-the-data)
- [使用粉碎加载 JSON 数据](https://redshift-immersion.workshop.aws/lab15.html#load-json-data-with-shredding)
- [离开前](https://redshift-immersion.workshop.aws/lab15.html#before-you-leave)

## 在你开始之前

本实验假设您已启动 Redshift 集群并配置了客户端工具。如果您尚未启动集群，请参阅 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html)。如果您尚未配置客户端工具，请参阅 [实验室 1 - 创建 Redshift 集群：配置客户端工具](https://redshift-immersion.workshop.aws/lab1.html#configure-client-tool)。对于本实验，您需要从 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html) 中收集有关集群的以下信息。

- [Your-Redshift_Role_Arn]

## 加载 JSON 数据

假设您有一个登陆以下 s3 位置的交易数据提要。我们的目标是以尽可能简单的方式加载它：

https://s3.console.aws.amazon.com/s3/buckets/redshift-downloads?region=us-east-1&prefix=semistructured/tpch-nested/data/json/customer_orders_lineitem/&showversions=false

1. 创建一个表来存储这些数据，而不必担心架构或底层结构：

```SQL
drop table if exists transaction;
create table transaction (
	data_json super
);
```

2. 现在，使用 COPY 语句加载数据。 请注意 *noshred* 选项，它将导致数据加载到一个字段中。 稍后我们将看到如何[使用切碎加载 JSON 数据](https://redshift-immersion.workshop.aws/lab15.html#load-json-data-with-shredding)

将“[Your-Redshift_Role_Arn]”替换为您之前检索到的值。

```sql
copy transaction
from 's3://redshift-downloads/semistructured/tpch-nested/data/json/customer_orders_lineitem/customer_order_lineitem.json'
iam_role '[Your-Redshift_Role_Arn]' region 'us-east-1'
json 'noshred';
```

3. 验证数据已加载。 注意 JSON 结构。 在顶层，您有与客户相关的属性。

```sql
select * from transaction limit 1;
```

[![img](https://redshift-immersion.workshop.aws/images/lab15/structure.png)](https://redshift-immersion.workshop.aws/images/lab15/structure.png)

4. 记下记录计数，以便我们可以比较嵌套和非嵌套数据集之间的计数。

```sql
select count(1) from transaction;
```

[![img](https://redshift-immersion.workshop.aws/images/lab15/count.png)](https://redshift-immersion.workshop.aws/images/lab15/count.png)

5. 编写一个查询来提取与客户相关的属性。 由于这些位于 JSON 结构的顶层，因此您应该期望与前一行计数匹配的行计数：

```sql
select data_json.c_custkey, data_json.c_phone, data_json.c_acctbal from transaction;
```

[![img](https://redshift-immersion.workshop.aws/images/lab15/parsed.png)](https://redshift-immersion.workshop.aws/images/lab15/parsed.png)

6. 请注意，某些字段具有双引号包装器。 这是因为 Redshift 在从 SUPER 字段查询时不强制执行类型。 这有助于模式演变，以确保如果数据类型随时间发生变化，加载过程不会失败。 请参阅以下文档以了解有关 [动态类型](https://docs.aws.amazon.com/redshift/latest/dg/query-super.html#dynamic-typing-lax-processing) 的更多信息。 对提取的字段进行类型转换，以便它们正确显示。

```sql
select data_json.c_custkey::int, data_json.c_phone::varchar, data_json.c_acctbal::decimal(18,2) from transaction;
```

[![img](https://redshift-immersion.workshop.aws/images/lab15/parsed_typed.png)](https://redshift-immersion.workshop.aws/images/lab15/parsed_typed.png)

 ## 取消嵌套数据

   在我们的用例中，您可能已经注意到 JSON 文档包含一个 *c_orders* 字段，这是另一个复杂的数据结构。 此外，*c_orders* 结构包含另一个嵌套字段。 *o_lineitems*。

   7. 编写一个取消嵌套数据的查询。 确定订单和订单项的数量。 请注意，要取消嵌套数据，您可以利用 [PartiQL](https://partiql.org/) 语法。

```sql
select count(1) from transaction t, t.data_json.c_orders o;
select count(1) from transaction t, t.data_json.c_orders o, o.o_lineitems l;
```

[![img](https://redshift-immersion.workshop.aws/images/lab15/unnest_count1.png)](https://redshift-immersion.workshop.aws/images/lab15/unnest_count1.png)[![img](https://redshift-immersion.workshop.aws/images/lab15/unnest_count2.png)](https://redshift-immersion.workshop.aws/images/lab15/unnest_count2.png)

8. 要获取客户、订单和订单项详细信息的完整数据集，请执行从嵌套层次结构中的每个级别提取数据的查询。

```sql
select data_json.c_custkey::int, data_json.c_phone::varchar, data_json.c_acctbal::decimal(18,2), o.o_orderstatus::varchar, l.l_shipmode::varchar, l.l_extendedprice::decimal(18,2)
from transaction t, t.data_json.c_orders o, o.o_lineitems l;
```

[![img](https://redshift-immersion.workshop.aws/images/lab15/unnest_data.png)](https://redshift-immersion.workshop.aws/images/lab15/unnest_data.png)

## 使用粉碎加载 JSON 数据

   在某些用例中，顶级 JSON 数据不会经常更改和/或您的 ETL 过程不需要合并添加到顶级的新字段。 在这些情况下，您可以使用带有选项 *ignorecase* 参数的 *auto* 选项来加载您的 JSON 数据。

   9. 创建一个包含 JSON 文档第一级字段的表：

```sql
drop table if exists transaction_shred;
create table transaction_shred (
  c_custkey bigint,
  c_phone varchar(20),
  c_acctbal decimal(18,2),
  c_orders super  
);
```

10. 现在，使用 COPY 语句再次加载数据。 这次使用 *auto* 选项，这将导致将数据加载到与 JSON 对象中的列名称匹配的字段中。 列映射将使用基于 *ignorecase* 关键字的不区分大小写的匹配。

```sql
copy transaction_shred from 's3://redshift-downloads/semistructured/tpch-nested/data/json/customer_orders_lineitem/customer_order_lineitem.json'
iam_role '[Your-Redshift_Role_Arn]' region 'us-east-1'
json 'auto ignorecase';
```

11. 当表列与 JSON 文件不匹配时会发生什么？ 缺少一栏？ 有额外的列吗？ 列乱序？

   1. 让我们获取客户、订单和订单项详细信息的完整数据集，但这一次，利用层次结构的第一级已被切碎的事实。

```sql
select c_custkey, c_phone, c_acctbal, o.o_orderstatus::varchar, l.l_shipmode::varchar, l.l_extendedprice::decimal(18,2)
from transaction_shred t, t.c_orders o, o.o_lineitems l;
```

[![img](https://redshift-immersion.workshop.aws/images/lab15/unnest_data_shred.png)](https://redshift-immersion.workshop.aws/images/lab15/unnest_data_shred.png)

## 将 JSON 文档解析为 SUPER 列

   您可以使用 *JSON_PARSE* 函数将 JSON 数据插入或更新到 SUPER 列中。 此函数将数据解析为 JSON 格式并将其转换为 SUPER 数据类型，您可以在 INSERT 或 UPDATE 语句中使用该数据类型。 这包括验证数据中的 JSON 文档。

   12. 在 *transaction_shred* 表中插入一个新的客户订单。

```sql
INSERT INTO transaction_shred VALUES
  (1234,
   '800-867-5309',
   441989.88,
   JSON_PARSE(  
    '[{
      "o_orderstatus":"F",
      "o_clerk":"Clerk#0000001991",
      "o_lineitems":[
         {
            "l_returnflag":"R",
            "l_receiptdate":"2017-07-23",
            "l_tax":0.03,
            "l_shipmode":"TRUCK",
            "l_suppkey":4799,
            "l_shipdate":"2014-06-24",
            "l_commitdate":"2014-06-05",
            "l_partkey":54798,
            "l_quantity":4,
            "l_linestatus":"F",
            "l_comment":"Net new order for new customer",
            "l_extendedprice":28007.64,
            "l_linenumber":1,
            "l_discount":0.02,
            "l_shipinstruct":"TAKE BACK RETURN"
         }
      ],
      "o_orderdate":"2014-06-01",
      "o_shippriority":0,
      "o_totalprice":28308.25,
      "o_orderkey":1953697,
      "o_comment":"wing for 997.1 gt3",
      "o_orderpriority":"5-LOW"
    }],'));
```

由于 *JSON_PARSE* 函数检测到格式错误的 JSON 文档，因此插入错误。

[![img](https://redshift-immersion.workshop.aws/images/lab15/insert_json_parse_fail.png)](https://redshift-immersion.workshop.aws/images/lab15/insert_json_parse_fail.png)



13. 删除 JSON 末尾不需要的逗号，然后再次尝试插入数据。

```sql
INSERT INTO transaction_shred VALUES
  (1234,
   '800-867-5309',
   441989.88,
   JSON_PARSE(  '[{
      "o_orderstatus":"F",
      "o_clerk":"Clerk#0000001991",
      "o_lineitems":[
         {
            "l_returnflag":"R",
            "l_receiptdate":"2017-07-23",
            "l_tax":0.03,
            "l_shipmode":"TRUCK",
            "l_suppkey":4799,
            "l_shipdate":"2014-06-24",
            "l_commitdate":"2014-06-05",
            "l_partkey":54798,
            "l_quantity":4,
            "l_linestatus":"F",
            "l_comment":"Net new order for new customer",
            "l_extendedprice":28007.64,
            "l_linenumber":1,
            "l_discount":0.02,
            "l_shipinstruct":"TAKE BACK RETURN"
         }
      ],
      "o_orderdate":"2014-06-01",
      "o_shippriority":0,
      "o_totalprice":28308.25,
      "o_orderkey":1953697,
      "o_comment":"wing for 997.1 gt3",
      "o_orderpriority":"5-LOW"
   }]'));
```

14. 接下来，更新您刚刚插入的行，为订单添加新的行项目。

如果您更新 SUPER 数据列，Amazon Redshift 需要将完整文档传递给列值。 Amazon Redshift 不支持部分更新。

```sql
update transaction_shred 
set c_orders =  
   JSON_PARSE(  '[{
      "o_orderstatus":"F",
      "o_clerk":"Clerk#0000001991",
      "o_lineitems":[
         {
            "l_returnflag":"R",
            "l_receiptdate":"2017-07-23",
            "l_tax":0.03,
            "l_shipmode":"TRUCK",
            "l_suppkey":4799,
            "l_shipdate":"2014-06-24",
            "l_commitdate":"2014-06-05",
            "l_partkey":54798,
            "l_quantity":4,
            "l_linestatus":"F",
            "l_comment":"Net new order for new customer",
            "l_extendedprice":28007.64,
            "l_linenumber":1,
            "l_discount":0.02,
            "l_shipinstruct":"TAKE BACK RETURN"
         },
         {
            "l_returnflag":"R",
            "l_receiptdate":"2017-07-23",
            "l_tax":0.03,
            "l_shipmode":"TRUCK",
            "l_suppkey":4799,
            "l_shipdate":"2014-06-24",
            "l_commitdate":"2014-06-05",
            "l_partkey":54798,
            "l_quantity":4,
            "l_linestatus":"F",
            "l_comment":"Net new order2 for new customer",
            "l_extendedprice":28007.64,
            "l_linenumber":2,
            "l_discount":0.02,
            "l_shipinstruct":"TAKE BACK RETURN"
         }
      ],
      "o_orderdate":"2014-06-01",
      "o_shippriority":0,
      "o_totalprice":28308.25,
      "o_orderkey":1953697,
      "o_comment":"wing for 997.1 gt3",
      "o_orderpriority":"5-LOW"
   }]')
   where c_custkey = 1234;
```

15. 最后，让我们获取客户、订单和订单项详细信息的完整数据集。

```sql
select c_custkey, c_phone, c_acctbal, o.o_orderstatus::varchar, l.l_shipmode::varchar, l.l_extendedprice::decimal(18,2), l.l_linenumber::int
from transaction_shred t, t.c_orders o, o.o_lineitems l
order by c_custkey;
```

[![img](https://redshift-immersion.workshop.aws/images/lab15/select_inserted_rows.png)](https://redshift-immersion.workshop.aws/images/lab15/select_inserted_rows.png)

如果查询中缺少 *JSON_PARSE* 函数，Amazon Redshift 会将值视为单个字符串，而不是必须解析的 JSON 格式的字符串。

## 在你离开之前

如果您使用完集群，请考虑将其停用，以避免为未使用的资源付费。 有关在 Amazon Redshift 中提取和查询半结构化数据的更多文档，请参阅此处的文档：https://docs.aws.amazon.com/redshift/latest/dg/super-overview.html

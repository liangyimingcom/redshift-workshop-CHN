# REDSHIFT 沉浸式实验室

Amazon Redshift 是一种快速、完全托管的 PB 级数据仓库解决方案，它使用列式存储来最大限度地减少 IO，提供高数据压缩率并提供快速性能。这套研讨会提供了一系列练习，可帮助用户开始使用 Redshift 平台。它还有助于演示平台内置的许多功能。

## 实验室

| # |实验室名称 |实验室说明 |
| :--- | :------------------------------------------------- ---------- | :------------------------------------------------- ---------- |
| 1 | [创建集群](https://redshift-immersion.workshop.aws/lab1.html) |集群设置和连接查询编辑器 |
| 2 | [数据加载](https://redshift-immersion.workshop.aws/lab2.html) |表创建、数据加载和表维护 |
| 3 | [表设计和查询调优](https://redshift-immersion.workshop.aws/lab3.html) |设置分布和排序键、深拷贝、解释计划、系统表查询|
| 4 | [使用频谱实现现代化](https://redshift-immersion.workshop.aws/lab4.html) |使用 Redshift Spectrum 查询数据仓库中 PB 级数据和 S3 数据湖中 EB 级数据 |
| 5 | [频谱查询调优](https://redshift-immersion.workshop.aws/lab5.html) |诊断 Redshift Spectrum 查询性能并通过利用分区、优化存储和谓词下推进行优化。 |
| 6 | [使用联邦查询 Aurora PostgreSQL](https://redshift-immersion.workshop.aws/lab6.html) |利用联合功能加入 Amazon Redshift 和 Amazon RDS PostgreSQL。 |
| 7 | [操作](https://redshift-immersion.workshop.aws/lab7.html) |逐步执行 Redshift 管理员可能需要执行的一些常见操作来维护其 Redhshift 环境，包括事件订阅、集群加密、跨区域快照和弹性调整 |
| 8 | [在 S3 中查询嵌套 JSON](https://redshift-immersion.workshop.aws/lab8.html) |查询嵌套的 JSON 数据类型（数组、结构、映射）并将嵌套数据类型加载到扁平结构中。 |
| 9 | [将 SAML 2.0 用于带 Redshift 的 SSO](https://redshift-immersion.workshop.aws/lab9.html) |使用 Redshift BrowserSAML 插件和任何 SAML 2.0 提供程序启用 SSO。 |
| 10 | [使用 Redshift 加速预测模型训练](https://redshift-immersion.workshop.aws/lab10.html) |了解如何使用 Redshift 进行数据整理和加速机器学习用例。 |
| 11 | 【Oracle 到 Redshift 迁移】(https://redshift-immersion.workshop.aws/lab11.html) |使用 AWS Schema Conversion Tool (AWS SCT) 和 AWS Database Migration Service (DMS) 将数据和代码从 Oracle 数据库迁移到 Amazon Redshift。 |
| 12 | 【SQL Server 到 Redshift 迁移】(https://redshift-immersion.workshop.aws/lab12.html) |使用 AWS Schema Conversion Tool (AWS SCT) 将数据和代码从 Microsoft SQL Server 数据库迁移到 Amazon Redshift。 |
| 13 | [ETL/ELT 策略](https://redshift-immersion.workshop.aws/lab13.html) |使用物化视图、存储过程和查询调度使您的 ETL/ELT 流程现代化。 |
| 14 | [数据共享](https://redshift-immersion.workshop.aws/lab14.html) |通过在 2 个 Redshift 集群之间共享数据来隔离您的工作负载。 |
| 15 | [加载和查询半结构化数据](https://redshift-immersion.workshop.aws/lab15.html) |使用 SUPER 数据类型将半结构化 JSON 数据加载到 Redshift 并管理架构演变。 |
| 16 | [Redshift 数据 API](https://redshift-immersion.workshop.aws/lab16.html) | 通过在暴露给 API 网关的 AWS Lambda 中创建 Python 函数，通过 Redshift Data API 查询数据。 |
| 17a | [使用 Redshift ML 进行机器学习](https://redshift-immersion.workshop.aws/lab17a.html) | 使用自动特征创建模型。 |
| 17b | [使用 Redshift ML 进行机器学习](https://redshift-immersion.workshop.aws/lab16b.html) | 创建一个指定 PROBLEM_TYPE 和 OBJECTIVE 的模型。 |
| 17c | [使用 Redshift ML 进行机器学习](https://redshift-immersion.workshop.aws/lab16c.html) | 创建模型并提供 MODEL_TYPE 、 OBJECTIVE、 PREPROCESSORS 和 HYPER PARAMETER。 |
| 18 | [Power BI with Redshift](https://redshift-immersion.workshop.aws/lab18.html) | 使用 Power BI 针对存储在 Amazon Redshift 中的数据创建仪表板 |

** ** **
** ** **

# REDSHIFT IMMERSION LABS

Amazon Redshift is a fast, fully managed, petabyte-scale data warehouse solution that uses columnar storage to minimise IO, provides high data compression rates, and offers fast performance. This set of workshops provides a series of exercises which help users get started using the Redshift platform. It also helps demonstrate the many features built into the platform.

## Labs

| #    | Lab Name                                                     | Lab Description                                              |
| :--- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 1    | [Creating a Cluster](https://redshift-immersion.workshop.aws/lab1.html) | Cluster setup and connecting with Query Editor               |
| 2    | [Data Loading](https://redshift-immersion.workshop.aws/lab2.html) | Table creation, data load, and table maintenance             |
| 3    | [Table Design and Query Tuning](https://redshift-immersion.workshop.aws/lab3.html) | Setting distribution and sort keys, deep copy, explain plans, system table queries |
| 4    | [Modernize w/ Spectrum](https://redshift-immersion.workshop.aws/lab4.html) | Query petabytes of data in your data warehouse and exabytes of data in your S3 data lake, using Redshift Spectrum |
| 5    | [Spectrum Query Tuning](https://redshift-immersion.workshop.aws/lab5.html) | Diagnose Redshift Spectrum query performance and optimize by leveraging partitions, optimizing storage, and predicate pushdown. |
| 6    | [Query Aurora PostgreSQL using Federation](https://redshift-immersion.workshop.aws/lab6.html) | Leverage the Federation capability to JOIN Amazon Redshift AND Amazon RDS PostgreSQL. |
| 7    | [Operations](https://redshift-immersion.workshop.aws/lab7.html) | Step through some common operations a Redshift Administrator may have to do to maintain their Redhshift environment including Event Subscriptions, Cluster Encryption, Cross Region Snapshots, and Elastic Resize |
| 8    | [Querying nested JSON in S3](https://redshift-immersion.workshop.aws/lab8.html) | Query Nested JSON datatypes (array, struct, map) and load nested data types into flattened structures. |
| 9    | [Use SAML 2.0 for SSO with Redshift](https://redshift-immersion.workshop.aws/lab9.html) | Enable SSO using the Redshift BrowserSAML plugin with any SAML 2.0 provider. |
| 10   | [Speedup predicative model training with Redshift](https://redshift-immersion.workshop.aws/lab10.html) | Learn how to use Redshift to do Data Wrangling and speedup machine learning use case. |
| 11   | [Oracle to Redshift Migration](https://redshift-immersion.workshop.aws/lab11.html) | Use AWS Schema Conversion Tool (AWS SCT) and AWS Database Migration Service (DMS) to migrate data and code from an Oracle database to Amazon Redshift. |
| 12   | [SQL Server to Redshift Migration](https://redshift-immersion.workshop.aws/lab12.html) | Use AWS Schema Conversion Tool (AWS SCT) to migrate data and code from a Microsoft SQL Server database to Amazon Redshift. |
| 13   | [ETL/ELT Strategies](https://redshift-immersion.workshop.aws/lab13.html) | Modernize your ETL/ELT process using Materialized Views, Stored Procedures, and Query Scheduling. |
| 14   | [Data Sharing](https://redshift-immersion.workshop.aws/lab14.html) | Isolate your workloads by sharing data between 2 Redshift clusters. |
| 15   | [Loading & querying semi-structured data](https://redshift-immersion.workshop.aws/lab15.html) | Load semi-structured JSON data into Redshift and manage schema evolution by using the SUPER data type. |
| 16   | [Redshift Data API](https://redshift-immersion.workshop.aws/lab16.html) | Query data via the Redshift Data API by creating Python function in AWS Lambda which is exposed to the API Gateway. |
| 17a  | [Machine Learning using Redshift ML](https://redshift-immersion.workshop.aws/lab17a.html) | Create a model using auto features.                          |
| 17b  | [Machine Learning using Redshift ML](https://redshift-immersion.workshop.aws/lab16b.html) | Create a model specifying PROBLEM_TYPE and OBJECTIVE.        |
| 17c  | [Machine Learning using Redshift ML](https://redshift-immersion.workshop.aws/lab16c.html) | Create a model and provide the MODEL_TYPE , OBJECTIVE, PREPROCESSORS and HYPER PARAMETER. |
| 18   | [Power BI with Redshift](https://redshift-immersion.workshop.aws/lab18.html) | Create dashboards using Power BI on data stored in Amazon Redshift |

** ** **

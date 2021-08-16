---
title: "AWS Redshift数仓动手训练营"
chapter: false
weight: 1
---



# AWS Redshift 数仓 + 帆软 BI 智能分析 实验日
## 日程安排

- Amazon Redshift 是一种快速、完全托管的 PB 级数据仓库解决方案，它使用列式存储来最大限度地减少 IO，提供高数据压缩率并提供快速性能。
- 帆软BI是Finereport+FineBI的组合，包含以IT为中心的预定义报表平台+以业务为中心的自助大数据分析平台
- 本次实验日提供了一系列练习与课程，可帮助用户开始使用 Redshift 平台，从而快速学习Redshift的多项先进功能。

 

| 欢迎与介绍                                                   | 10 分钟 | 让我们开始使用AWS和帆软BI智能分析！                          | 欢迎   |
| ------------------------------------------------------------ | ------- | ------------------------------------------------------------ | ------ |
| 亚马逊Redshift  简介                                         | 30分钟  | Redshift 解决方案架构套件的一部分。此模块将客户介绍给亚马逊 Redshift，涵盖数据趋势、云数据仓库需求以及 Redshift 的架构。 | 介绍   |
| 实验室1：数据加载  Lab1: Data  Loading                       | 40 分钟 | 在这个实验室中，您将使用基于 TPC 基准数据模型的一组八个表。您在"Redshift "集群中创建这些表，并将这些表加载到存储在 S3 中的示例数据中。 | 实验室 |
| 实验室2：ETL/ELT 策略  Lab2: ETL/ELT Strategies              | 30 分钟 | 此实验室演示了如何使用实现视图、存储程序和查询调度实现 ETL/ELT 流程的现代化，以在  Redshift 中转换数据。 | 实验室 |
| 实验室3：Spectrum现代化  Lab3: Modernize with  Spectrum      | 40 分钟 | 在这个实验室中，我们向您展示如何在不加载或移动对象的情况下，使用 Amazon Redshift 和亚马逊 S3 数据湖中联合查询PB级别数据。 | 实验室 |
| 实验室4：使用联合查询关系型数据库  Lab4: Query Aurora  PostgreSQL using Federation | 30 分钟 | 在数据仓库环境中，您的工作负载通常与各种关系型数据库进行联合查询；在这个实验室中，我们向您展示如何在不加载或移动关系型数据库的情况下，使用 Amazon Redshift联合查询关系型数据库。 | 实验室 |
| 实验室5：帆软BI智能分析 DEMO演示与实验                       | 40 分钟 | 帆软BI是Finereport+FineBI的组合，包含以IT为中心的预定义报表平台+以业务为中心的自助大数据分析平台，通过双模IT的配合模式，最大程度切合国内企业用户的实际信息化建设需求。FineReport企业级Web报表工具，需简单的拖拽操作便可以设计复杂的中国式报表，搭建数据决策分析系统。 | 实验室 |
| 总结                                                         | 15 分钟 |                                                              | 结论   |

课程内容将根据实际会议时间略有调整，谢谢。





** ** **
# REDSHIFT 沉浸式实验室

Amazon Redshift 是一种快速、完全托管的 PB 级数据仓库解决方案，它使用列式存储来最大限度地减少 IO，提供高数据压缩率并提供快速性能。这套研讨会提供了一系列练习，可帮助用户开始使用 Redshift 平台。它还有助于演示平台内置的许多功能。

## 实验室

| # |实验室名称 |实验室说明 |
| :--- | :----------------------------------------------------------- | :----------------------------------------------------------- |
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




---
title: "11. REDSHIFT 数据接口API"
chapter: false
weight: 92
---

# REDSHIFT  数据接口API

在本实验中，您将学习如何通过 Redshift Data API 查询数据。 Amazon Redshift Data API 简化了来自 AWS 开发工具包支持的编程语言和平台（例如 Python、Go、Java、Node.js）的数据访问、摄取和输出。 js、PHP、Ruby 和 C++。我们将在 AWS Lambda 中创建 Python 函数以通过 Redshift Data API 进行交互。

数据 API 不需要与集群的持久连接。相反，它提供了一个安全的 HTTP 端点以及与 AWS 开发工具包的集成。您可以使用端点来运行 SQL 语句，而无需管理连接。对数据 API 的调用是异步的。

这是客户希望以编程方式对 Redshift 集群执行查询而不进行数据共享或开发 Rest API 的常见场景。

## 内容

- [内容](https://redshift-immersion.workshop.aws/lab16.html#contents)
- [开始之前](https://redshift-immersion.workshop.aws/lab16.html#before-you-begin)
- [准备 IAM 权限](https://redshift-immersion.workshop.aws/lab16.html#prepare-iam-permissions)
- [创建 AWS Lambda 函数](https://redshift-immersion.workshop.aws/lab16.html#create-aws-lambda-function)
- [与 API 网关集成](https://redshift-immersion.workshop.aws/lab16.html#integrate-with-api-gateway)
- [在浏览器中测试 API](https://redshift-immersion.workshop.aws/lab16.html#test-api-in-browser)
- [离开前](https://redshift-immersion.workshop.aws/lab16.html#before-you-leave)

## 在你开始之前

本实验假设您已启动 Redshift 集群，并已将 TPC 基准数据加载到该集群中。如果您尚未启动集群，请参阅 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html)。如果您还没有加载它，请参阅 [2.数据加载](https://redshift-immersion.workshop.aws/lab2.html)

对于本实验，您需要从 [实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html) 中收集有关集群的以下信息。

- [您的 AWS_Account]
- [Your-Redshift_Cluster_Identifier]
- [Your-Redshift_Cluster_Database]
- [Your-Redshift_Cluster_User]

## 准备 IAM 权限

我们需要向 Lambda 提供访问您的 Redshift 集群的权限。在顶部搜索栏中输入“IAM”，然后从结果中右键单击并在新选项卡中打开

[![img](https://redshift-immersion.workshop.aws/images/lab16/10_IAM.png)](https://redshift-immersion.workshop.aws/images/lab16/10_IAM.png)

从左侧菜单中选择“Policy”，然后单击“Create policy”蓝色按钮[![img](https://redshift-immersion.workshop.aws/images/lab16/11_IAM_policy.png)](https://redshift-immersion.workshop.aws/images/lab16/11_IAM_policy.png)

将下面的 JSON 文档复制到剪贴板。 这些是获取 redshift 集群凭据、执行 redshift 数据 API 的必要权限。

```json
{
"Version": "2012-10-17",
"Statement": [
  {
    "Effect": "Allow",
    "Action": ["redshift:GetClusterCredentials"],
    "Resource": [
      "arn:aws:redshift:*:[AWS_Account]:dbname:[Redshift_Cluster_Identifier]/[Redshift_Cluster_Database]",
      "arn:aws:redshift:*:[AWS_Account]:dbuser:[Redshift_Cluster_Identifier]/[Redshift_Cluster_User]"
    ]
  },
  {
    "Effect": "Allow",
    "Action": "redshift-data:*",
    "Resource": "*"
  }
]
}
```

选择“JSON”选项卡并在其中替换下面的文档。 用您的实际值替换“AWS_Account”、“Redshift_Cluster_Identifier”、“Redshift_Cluster_Database”和“Redshift_Cluster_User”[![img](https://redshift-immersion.workshop.aws/images/lab16/4_dataapipolicy.png)](https://redshift-immersion.workshop.aws/images/lab16/4_dataapipolicy.png)

单击下一步，直到到达最终的创建策略页面。 将您的策略命名为“RedshiftDataAPIPolicy”。[![img](https://redshift-immersion.workshop.aws/images/lab16/5_dataapipolicy.png)](https://redshift-immersion.workshop.aws/images/lab16/5_dataapipolicy.png)

现在让我们创建一个新的 IAM 角色 `Redshift-data-api-role` 并附加我们刚刚创建的这个策略。[![img](https://redshift-immersion.workshop.aws/images/lab16/6_datapi_access.gif)](https://redshift-immersion.workshop.aws/images/lab16/6_datapi_access.gif)

## 创建 AWS Lambda 函数

在 AWS 控制台上搜索 Lambda 并在新选项卡中打开。 单击右侧的“创建函数”[![img](https://redshift-immersion.workshop.aws/images/lab16/1_lambda_create.png)](https://redshift-immersion.workshop.aws/images/lab16/1_lambda_create.png)

让我们将该函数命名为 `redshift-data-api-demo` 并从 `Runtime` 下拉列表中选择 `Python 3.8` 或更高版本，以最新版本显示，切换执行角色并选择现有角色单选按钮并选择 `Redshift-data -api-role` 来自下拉列表。 单击底部的“创建函数”按钮[![img](https://redshift-immersion.workshop.aws/images/lab16/2_l_runtime.png)](https://redshift-immersion.workshop.aws/images/lab16/2_l_runtime.png)

在代码源部分下，双击文件“lambda_function.py”，将函数代码加载到右侧面板的编辑器中[![img](https://redshift-immersion.workshop.aws/images/lab16/3_l_code.png)](https://redshift-immersion.workshop.aws/images/lab16/3_l_code.png)

在这里，您复制粘贴以下代码以完全替换现有代码。 单击“部署”以保存和部署您的函数代码

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

import sys
import os
import boto3
import json
import datetime

# initialize redshift-data client in boto3
redshift_client = boto3.client("redshift-data")

def call_data_api(redshift_client, redshift_database, redshift_user, redshift_cluster_id, sql_statement, with_event=True):
    # execute the input SQL statement
    api_response = redshift_client.execute_statement(Database=redshift_database, DbUser=redshift_user
                                                    ,Sql=sql_statement, ClusterIdentifier=redshift_cluster_id, WithEvent=True)

    # return the query_id
    query_id = api_response["Id"]
    return query_id

def check_data_api_status(redshift_client, query_id):
    desc = redshift_client.describe_statement(Id=query_id)
    status = desc["Status"]

    if status == "FAILED":
        raise Exception('SQL query failed:' + query_id + ": " + desc["Error"])
    return status.strip('"')

def get_api_results(redshift_client, query_id):
    response = redshift_client.get_statement_result(Id=query_id)
    return response

def lambda_handler(event, context):
    redshift_cluster_id = os.environ['redshift_cluster_id']
    redshift_database = os.environ['redshift_database']
    redshift_user = os.environ['redshift_user']

    action = event['queryStringParameters'].get('action')
    try:
        if action == "execute_report":
            country = event['queryStringParameters'].get('country_name')
            # sql report query to be submitted
            sql_statement = "select c.c_mktsegment as customer_segment,sum(o.o_totalprice) as total_order_price,extract(year from o.o_orderdate) as order_year,extract(month from o.o_orderdate) as order_month,r.r_name as region,n.n_name as country,o.o_orderpriority as order_priority from public.orders o inner join public.customer c on o.o_custkey = c.c_custkey inner join public.nation n on c.c_nationkey = n.n_nationkey inner join public.region r on n.n_regionkey = r.r_regionkey where n.n_name = '"+ country +"' group by 1,3,4,5,6,7 order by 2 desc limit 10"
            api_response = call_data_api(redshift_client, redshift_database, redshift_user, redshift_cluster_id, sql_statement)
            return_status = 200
            return_body = json.dumps(api_response)

        elif action == "check_report_status":            
            # query_id to input for action check_report_status
            query_id = event['queryStringParameters'].get('query_id')            
            # check status of a previously executed query
            api_response = check_data_api_status(redshift_client, query_id)
            return_status = 200
            return_body = json.dumps(api_response)

        elif action == "get_report_results":
            # query_id to input for action get_report_results
            query_id = event['queryStringParameters'].get('query_id')
            # get results of a previously executed query
            api_response = get_api_results(redshift_client, query_id)
            return_status = 200
            return_body = json.dumps(api_response)

            # total number of rows
            nrows=api_response["TotalNumRows"]
            # number of columns
            ncols=len(api_response["ColumnMetadata"])
            print("Number of rows: %d , columns: %d" % (nrows, ncols) )

            for record in api_response["Records"]:
                print (record)
        else:
            return_status = 500
            return_body = "Invalid Action: " + action
        return_headers = {
                        "Access-Control-Allow-Headers" : "Content-Type",
                        "Access-Control-Allow-Origin": "*",
                        "Access-Control-Allow-Methods": "GET"}
        return {'statusCode' : return_status,'headers':return_headers,'body' : return_body}
    except NameError as error:
        raise NameError(error)
    except Exception as exception:
        error_message = "Encountered exeption on:" + action + ":" + str(exception)
        raise Exception(error_message)
```

我们有此 Lambda 函数支持的三种不同操作

1. 执行报告：此函数使用为 redshift 客户端、数据库、用户、集群标识符和查询语句提供的参数调用 Redshift 数据 API。 python 函数`call_data_api` 实现了这个功能。
2. 检查报告状态：该函数使用 Redshift Data API 来检查以前提交的查询语句。 请注意，与 redshift 客户端一起，需要传递查询 ID。 python 函数`check_data_api_status` 实现了这个功能。
3. 获取报告结果：该函数使用 Redshift Data API 来检索先前提交的查询语句的结果。 请注意，与 redshift 客户端一起，需要传递查询 ID。 python 函数`get_api_results` 实现了这个功能。

选择“配置”选项卡并根据您的 Redshift 集群创建环境变量[![img](https://redshift-immersion.workshop.aws/images/lab16/7_lambda_env_variables.png)](https://redshift-immersion.workshop.aws/images/lab16/7_lambda_env_variables.png)

单击“测试”并选择“配置测试事件”。 复制以下 JSON 输入以测试函数。 使用以下输入将事件名称另存为“EventExecuteReport”

```json
{
  "queryStringParameters": {
    "action": "execute_report",
    "country_name": "UNITED STATES"
  }
}
```

让我们创建第一个测试事件来触发报告查询。[![img](https://redshift-immersion.workshop.aws/images/lab16/9_l_exec_report.gif)](https://redshift-immersion.workshop.aws/images/lab16/9_l_exec_report.gif)

请注意，Lambda 函数已提交查询并完成其执行。 Redshift Data API 异步执行语句，因此我们必须返回状态和结果。

注意集群返回的 `query_id`，我们将其用作后续事件的输入，`EventCheckReport` 和 `EventReportResults` 事件我们创建如下。

准备下一个测试事件`EventCheckReport`[![img](https://redshift-immersion.workshop.aws/images/lab16/10_l_check_report.gif)](https://redshift-immersion.workshop.aws/images/lab16/10_l_check_report.gif)

使用`EventCheckReport`作为模板准备下一个测试事件`EventReportResults`并将`action`替换为`get_report_results`[![img](https://redshift-immersion.workshop.aws/images/lab16/11_l_report_results.gif)](https://redshift-immersion.workshop.aws/images/lab16/11_l_report_results.gif)

## 与 API 网关集成

现在我们已经成功测试了这些函数，让我们将此 lambda 函数添加到“API 网关”，以便我们可以通过 Web 浏览器查询报告。

在 AWS 控制台搜索栏上输入“API”并在新选项卡中打开“API Gateway”。 在 AWS Gateway 控制台中，选择 API 类型为“REST API”，然后单击“构建”[![img](https://redshift-immersion.workshop.aws/images/lab16/12_gw_create_api.gif)](https://redshift-immersion.workshop.aws/images/lab16/12_gw_create_api.gif)

现在我们为我们的 API 定义资源。[![img](https://redshift-immersion.workshop.aws/images/lab16/13_gw_resource.png)](https://redshift-immersion.workshop.aws/images/lab16/13_gw_resource.png)

让我们在我们的资源上创建一个 GET 方法，并将我们的 Lambda 函数与其关联。[![img](https://redshift-immersion.workshop.aws/images/lab16/14_gw_lambda.gif)](https://redshift-immersion.workshop.aws/images/lab16/14_gw_lambda.gif)

最后，我们将 API 部署到网关并进行测试。 阶段用于隔离 API 的测试版本和生产版本。[![img](https://redshift-immersion.workshop.aws/images/lab16/15_gw_deploy.gif)](https://redshift-immersion.workshop.aws/images/lab16/15_gw_deploy.gif)

## 在浏览器中测试 API

现在让我们通过新的浏览器选项卡或窗口通过网关测试 API。 但首先我们需要创建一个带有代码的基本 HTML 页面来调用我们的 API。 复制 - 将下面的内容粘贴到文本编辑器中，并将文件另存为“test.html”。

```html
<html>
<head><meta http-equiv="Access-Control-Allow-Origin" content="*"></head>
<script>
var endpoint;
var counter;

function updateStatus(endpoint, querystring, callback) {
  var xhttp = new XMLHttpRequest();
  xhttp.onreadystatechange = function() {
    if (this.readyState == 4 && this.status == 200) {
     callback(this.responseText);
    }
  };
  console.log(endpoint+querystring);
  xhttp.open("GET", endpoint+"/"+querystring, true);
  xhttp.send();
}

function submitQuery() {
  endpoint = document.getElementById('endpoint').value;
  var country = document.getElementById('country').value;
  querystring = "?action=execute_report&country_name="+country;

  updateStatus(endpoint, querystring, function(status){
    query_id = status.split('"').join('');
    document.getElementById("status").innerHTML = query_id;
    querystring = "?action=check_report_status&query_id="+query_id;
    counter = 1;
    checkStatus(endpoint, querystring, function(){
      querystring = "?action=get_report_results&query_id="+query_id;
      updateStatus(endpoint, querystring, function(status){
        var jsonString = "<pre>" + JSON.stringify(JSON.parse(status),null,2) + "</pre>";
        document.getElementById("status").innerHTML = jsonString;
      });
    });
  });
}

function checkStatus(endpoint, querystring, callback) {
  updateStatus(endpoint, querystring, function(status){
    if (status == "\"FINISHED\"")
      callback();
    else {
      document.getElementById("status").innerHTML = counter + ": " + status;
      setTimeout('', 1000);
      counter++;
      checkStatus(endpoint, querystring, callback)
    }
  });
}

</script>
<label for=endpoint>Endpoint:</label><input id=endpoint type=text style="width:100%"><br>
<label for=country>Country:</label><input id=country type=text style="width:100%">
<button type="button" onclick=submitQuery()>Submit</button>
<div id=status>
</div>
```

现在在浏览器中打开这个 `test.html`。 首先，在“端点”字段中输入来自 API 网关的“调用 URL”。 其次，输入国家名称，大写示例美国或加拿大或印度，然后单击提交。

注意数据 API 的异步特性

1. 看到请求已经提交到集群，返回了`query_id`
2. 注意状态显示为“PICKED”，即 Redshift 选择了要执行的报告
3. 然后，当 Redshift 执行您的查询时，状态会更改为“STARTED”
4.最后状态变为`FINISHED`并显示结果
5. `ColumnMetadata` 部分描述了返回的列
6. `Records` 部分是查询结果中的实际数据元素

1. [![img](https://redshift-immersion.workshop.aws/images/lab16/16_api_in_action.gif)](https://redshift-immersion.workshop.aws/images/lab16/16_api_in_action.gif)

这完成了通过数据 API 在 Redshift 中演示异步查询执行的实验室。

## 在你离开之前

如果您使用完集群，请考虑将其停用，以避免为未使用的资源付费。

有关更多详细信息，请参阅 https://docs.aws.amazon.com/redshift/latest/mgmt/data-api.html

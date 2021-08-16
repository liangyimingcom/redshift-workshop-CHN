---
title: "1. 创建一个集群"
chapter: false
weight: 10
---

# 创建一个集群

在本实验中，您将启动一个新的 Redshift 集群、设置连接并配置 JDBC 客户端工具。

## 内容

- [开始之前](https://redshift-immersion.workshop.aws/lab1.html#before-you-begin)
- [云编队](https://redshift-immersion.workshop.aws/lab1.html#cloud-formation)
- [配置安全](https://redshift-immersion.workshop.aws/lab1.html#configure-security)
- [启动 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html#launch-redshift-cluster)
- [配置客户端工具](https://redshift-immersion.workshop.aws/lab1.html#configure-client-tool)
- [运行示例查询](https://redshift-immersion.workshop.aws/lab1.html#run-sample-query)
- [离开前](https://redshift-immersion.workshop.aws/lab1.html#before-you-leave)

## 在你开始之前

- 登录 AWS 控制台。如果您不熟悉 AWS，则可以创建一个账户。对于本实验，您需要收集以下信息：
  - [您的 AWS_Account_Id]
  - [Your_AWS_User_Name]

## Cloud Formation

要使用云形成启动此集群并自动配置网络安全，请使用以下链接并跳至 [配置客户端工具](https://redshift-immersion.workshop.aws/lab1.html#configure-client-tool)。
[![启动](https://redshift-immersion.workshop.aws/images/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?#/stacks/new?stackName=ImmersionLab1&templateURL=https://s3-us-west-2.amazonaws.com/redshift-immersionday-labs/lab1.yaml)


{{% notice note %}}
为集群选择区域时，请考虑 *US-WEST-2（俄勒冈）*。虽然大多数这些实验室可以在任何区域完成，但一些实验室查询位于 *US-WEST-2* 的 S3 中的数据。

该模板将使用默认 CIDR 块 0.0.0.0/0，它提供从任何 IP 地址的访问。最好的做法是替换这个应该具有访问权限的 IP 地址范围。出于这些实验的目的，请将其替换为您的 IP 地址 x.x.x.x/32。
{{% /notice %}}



## 配置安全

### 专有网络

创建或确定您将在其中启动 Redshift 集群的 VPC。对于许多客户来说，默认的 VPC、子网和安全组就足够了。如果您使用默认值，您可以跳到 [子网组](https://redshift-immersion.workshop.aws/lab1.html#subnet-group)。出于我们的目的，我们将创建一个新的“VPC”来隔离流量并创建一个新的公共子网，将在其中部署 Redshift 集群。

{{% notice note %}}
如果您不打算让您的集群“可公开访问”，客户也可以部署到私有子网中。
{{% /notice %}}


```
https://console.aws.amazon.com/vpc/home?#CreateVpc：
```

[![img](https://redshift-immersion.workshop.aws/images/VPC.png)](https://redshift-immersion.workshop.aws/images/VPC.png)

默认情况下，VPC 不支持 DNS 主机名。现在通过单击“操作”->“编辑 DNS 主机名”启用它。[![img](https://redshift-immersion.workshop.aws/images/VPC_1.png)](https://redshift-immersion.Workshop.aws/images/VPC_1.png)

单击“启用”复选框。[![img](https://redshift-immersion.workshop.aws/images/VPC_2.png)](https://redshift-immersion.workshop.aws/images/VPC_2.png)

### 互联网网关

为了允许您子网中的机器访问互联网，我们将创建一个“互联网网关”。创建后，选择 Internet 网关并将其附加到之前创建的 VPC。

```
https://console.aws.amazon.com/vpc/home?#Create%20Internet%20Gateway：
```

[![img](https://redshift-immersion.workshop.aws/images/InternetGateway.png)](https://redshift-immersion.workshop.aws/images/InternetGateway.png)

[![img]( https://redshift-immersion.workshop.aws/images/InternetGatewayAttach1.png)](https://redshift-immersion.workshop.aws/images/InternetGatewayAttach1.png)

[![img](https://redshift-immersion.workshop.aws/images/InternetGatewayAttach2.png)](https://redshift-immersion.workshop.aws/images/InternetGatewayAttach2.png)

### 子网

现在，在两个不同的可用区中创建两个具有默认路由到先前创建的 VPC 的子网，以提高容错能力。

```
https://console.aws.amazon.com/vpc/home?#CreateSubnet：
```

[![img](https://redshift-immersion.workshop.aws/images/Subnet1.png)](https://redshift-immersion.workshop.aws/images/Subnet1.png)[![img]( https://redshift-immersion.workshop.aws/images/Subnet2.png)](https://redshift-immersion.workshop.aws/images/Subnet2.png)

### 路由表

为了确保子网有办法连接到互联网，创建一个“路由表”，默认路由指向互联网网关，并添加新的子网。

```
https://console.aws.amazon.com/vpc/home?#CreateRouteTable：
```

[![img](https://redshift-immersion.workshop.aws/images/Route.png)](https://redshift-immersion.workshop.aws/images/Route.png)[![img]( https://redshift-immersion.workshop.aws/images/EditRoute.png)](https://redshift-immersion.workshop.aws/images/EditRoute.png)[![img](https://redshift-immersion.workshop.aws/images/EditSubnet.png)](https://redshift-immersion.workshop.aws/images/EditSubnet.png)

### 安全组

创建与您之前创建的 VPC 关联的“安全组”。编辑安全组以创建允许从您的 IP 地址传入 Redshift 连接的规则以及所有流量的自引用规则。

```
https://console.aws.amazon.com/vpc/home#SecurityGroups:sort=groupId
```

[![img](https://redshift-immersion.workshop.aws/images/SecurityGroup.png)](https://redshift-immersion.workshop.aws/images/SecurityGroup.png)

[![img](https://redshift-immersion.workshop.aws/images/SecurityGroup_1.png)](https://redshift-immersion.workshop.aws/images/SecurityGroup_1.png)

[![img](https://redshift-immersion.workshop.aws/images/SecurityGroup_2.png)](https://redshift-immersion.workshop.aws/images/SecurityGroup_2.png)

### 子网组

现在，通过单击添加此 VPC 的所有子网按钮，创建包含您之前创建的两个子网的 Redshift `Cluster Subnet Group`。

```
https://console.aws.amazon.com/redshiftv2/home?#subnet-groups
```

[![img](https://redshift-immersion.workshop.aws/images/SubnetGroup.png)](https://redshift-immersion.workshop.aws/images/SubnetGroup.png)

### 创建 IAM 角色

为了让 Redshift 能够访问 S3 以加载数据，请创建一个类型为“Redshift”和用例为“Redshift - Customizable”的“IAM 角色”，并附加 *AmazonS3FullAccess*、*AWSGlueConsoleFullAccess* 和 *AmazonRedshiftFullAccess * 角色的政策。

```
https://console.aws.amazon.com/iam/home?#/roles$new?step=type
```

[![img](https://redshift-immersion.workshop.aws/images/Role.png)](https://redshift-immersion.workshop.aws/images/Role.png)

### 修改调度程序访问的 IAM 角色

修改您刚刚创建的 *RedshiftImmersionRole* 角色以启用调度程序和查询编辑器的使用。导航到 *信任关系* 选项卡并单击 *编辑信任关系* 修改策略如下：

确保替换以下策略中的“[Your-AWS_Account_Id]”和“[Your_AWS_User_Name]”值。如果您使用基于角色的身份验证，则可以将条目替换为您的 IAM 角色。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "events.amazonaws.com",
          "redshift.amazonaws.com",
          "scheduler.redshift.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Sid": "AssumeRole",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::[Your-AWS_Account_Id]:user/[Your_AWS_User_Name]"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

## 启动 Redshift 集群

最后，导航到“Amazon Redshift Dashboard”并单击“Create Cluster”按钮。

```
https://console.aws.amazon.com/redshiftv2/home?#clusters
```

- 集群配置 - 选择节点类型并设置节点数量。对于这些实验室，具有 2 个节点的 ra3.xlplus 节点类型将是合适的。[![img](https://redshift-immersion.workshop.aws/images/CreateCluster1.png)](https://redshift-immersion.Workshop.aws/images/CreateCluster1.png)
- 集群详细信息 - 输入适合您组织的值。请注意主用户密码，因为您以后将无法检索此值。[![img](https://redshift-immersion.workshop.aws/images/CreateCluster2.png)](https://redshift-immersion .workshop.aws/images/CreateCluster2.png)
- 集群权限 - 选择您之前确定或创建的角色以关联到集群，然后单击“添加 IAM 角色”[![img](https://redshift-immersion.workshop.aws/images/CreateCluster3.png) ](https://redshift-immersion.workshop.aws/images/CreateCluster3.png)
- 其他配置 - 禁用“使用默认值”并选择您之前确定或创建的 VPC、子网组和 VPC 安全组。[![img](https://redshift-immersion.workshop.aws/images/CreateCluster4.png )](https://redshift-immersion.workshop.aws/images/CreateCluster4.png)

保留其余设置的默认值。单击“创建集群”以启动 Redshift 集群。

## 配置客户端工具

对于这些实验室，建议使用 Redshift 提供的基于 Web 的 [查询编辑器](https://console.aws.amazon.com/redshiftv2/home?#query-editor:)。

单击“连接到数据库”->“创建新连接”。

- 如果您的 IAM 用户/角色具有权限 *“redshift:GetClusterCredentials”*，您可以使用“临时凭证”选项
- 输入“集群”、“数据库名称”和“数据库用户”。单击连接。[![img](https://redshift-immersion.workshop.aws/images/Connection.png)](https://redshift-immersion.workshop.aws/images/Connection.png)
- 如果您的 IAM 用户/角色拥有 *secretmanager* 的权限，您可以使用 `AWS Secrets Manager` 选项
- 输入“秘密名称”、“数据库名称”、“数据库用户”、“密码”和“集群”。点击连接。[![img](https://redshift-immersion.workshop.aws/images/Connection_Secret.png)](https://redshift-immersion.workshop.aws/images/Connection_Secret.png)

要使用 Redshift 查询编辑器，您的 IAM 用户还需要访问 *redshift-data:** 权限。您还可以使用 ODBC/JDBC 驱动程序连接到 Amazon Redshift。有关更多信息，请参阅以下 [文档](https://docs.aws.amazon.com/redshift/latest/mgmt/connecting-to-cluster.html)。

## 运行示例查询

- 运行以下查询以列出 redshift 集群中的用户。

```
select * from pg_user
```

- 如果您收到以下结果，则您已建立连接，本实验已完成。
  [![img](https://redshift-immersion.workshop.aws/images/Users.png)](https://redshift-immersion.workshop.aws/images/Users.png)

## 在你离开之前

如果您使用完集群，请考虑将其停用，以避免为未使用的资源付费。

---
title: "7. 运维操作"
chapter: false
weight: 70
---



# 操作

在本实验中，我们将逐步介绍 Redshift 管理员为维护其 Redhshift 环境可能必须执行的一些常见操作。

## 内容

- [开始之前](https://redshift-immersion.workshop.aws/lab7.html#before-you-begin)
- [活动订阅](https://redshift-immersion.workshop.aws/lab7.html#event-subscriptions)
- [集群加密](https://redshift-immersion.workshop.aws/lab7.html#cluster-encryption)
- [跨区域快照](https://redshift-immersion.workshop.aws/lab7.html#cross-region-snapshots)
- [弹性调整大小](https://redshift-immersion.workshop.aws/lab7.html#elastic-resize)
- [离开前](https://redshift-immersion.workshop.aws/lab7.html#before-you-leave)

## 在你开始之前

本实验假设您已启动 Redshift 集群。 如果您尚未启动集群，请参阅
[实验室 1 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab1.html).

## 事件订阅

1. 导航到您的 Redshift 事件页面。 请注意创建集群所涉及的 *Events*。

```
https://console.aws.amazon.com/redshiftv2/home?#events
```

[![img](https://redshift-immersion.workshop.aws/images/Events.png)](https://redshift-immersion.workshop.aws/images/Events.png)

1. 单击*事件订阅* 选项卡，然后单击*创建事件订阅* 按钮。[![img](https://redshift-immersion.workshop.aws/images/CreateSubscription_0.png)](https://redshift-immersion.workshop.aws/images/CreateSubscription_0.png)
2. 为*所有集群*创建一个名为*ClusterManagement*的订阅，源类型为*Cluster*，严重性为*Info, Error*。
   [![img](https://redshift-immersion.workshop.aws/images/CreateSubscription_1.png)](https://redshift-immersion.workshop.aws/images/CreateSubscription_1.png)
3. 选择订阅操作。 选择*创建新的 SNS 主题* 并将该主题命名为 *ClusterManagement*。 单击*创建主题*。[![img](https://redshift-immersion.workshop.aws/images/CreateSubscription_2.png)](https://redshift-immersion.workshop.aws/images/CreateSubscription_2.png)
4. 确保订阅已启用，然后单击*创建事件订阅*。›[![img](https://redshift-immersion.workshop.aws/images/CreateSubscription_3.png)](https://redshift-immersion.workshop.aws/images/CreateSubscription_3.png)
5. 导航到 SNS 控制台并单击新创建的主题 *ClusterManagement*。 点击*创建订阅*。

```
https://console.aws.amazon.com/sns/v3/home?#/topics
```

[![img](https://redshift-immersion.workshop.aws/images/CreateSubscription_4.png)](https://redshift-immersion.workshop.aws/images/CreateSubscription_4.png)

1. 输入协议 *Email* 并输入您有权访问的电子邮件地址的端点。 单击*创建订阅*。[![img](https://redshift-immersion.workshop.aws/images/CreateSubscription_5.png)](https://redshift-immersion.workshop.aws/images/CreateSubscription_5.png)
2. 您将很快收到一封电子邮件。 单击电子邮件中的 *确认订阅* 链接。[![img](https://redshift-immersion.workshop.aws/images/ConfirmSubscriptionEmail.png)](https://redshift-immersion.workshop.aws/images/ConfirmSubscriptionEmail.png)
3. 该链接应将您带到确认订阅的最终确认页面。[![img](https://redshift-immersion.workshop.aws/images/SubscriptionConfirmed.png)](https://redshift-immersion.workshop.aws/images/SubscriptionConfirmed.png)

## 集群加密

注意：根据 [LAB 2 - 创建 Redshift 集群](https://redshift-immersion.workshop.aws/lab2.html) 中加载的数据，完成本部分实验大约需要 45 分钟。 请相应地计划。

1. 导航到您的 Redshift 集群列表。 选择您的集群并单击 *Actions* -> *Modify*。

```
https://console.aws.amazon.com/redshiftv2/home?#clusters
```

[![img](https://redshift-immersion.workshop.aws/images/ModifyCluster.png)](https://redshift-immersion.workshop.aws/images/ModifyCluster.png)

1. 在*数据库配置* 下，启用*使用 AWS 密钥管理服务 (AWS KMS)* 单选选项。 单击*修改集群*。[![img](https://redshift-immersion.workshop.aws/images/EnableKMS.png)](https://redshift-immersion.workshop.aws/images/EnableKMS.png)
2. 注意您的集群进入 *resizing* 状态。 加密集群的过程类似于使用经典调整大小方法调整集群大小。 所有数据都被读取、加密和重写。 在此期间，集群仍可用于读查询，但不能用于写查询。[![img](https://redshift-immersion.workshop.aws/images/Resizing.png)](https://redshift-immersion.workshop.aws/images/Resizing.png)
3. 由于我们之前设置的事件订阅，您还应该收到有关集群大小调整的电子邮件通知。[![img](https://redshift-immersion.workshop.aws/images/ResizeNotification.png)](https://redshift-immersion.workshop.aws/images/ResizeNotification.png)

## 跨区域快照

1. 导航到您的 Redshift 集群列表。 选择您的集群并单击 *Actinos* -> *配置跨区域快照*。

```
https://console.aws.amazon.com/redshiftv2/home?#clusters
```

[![img](https://redshift-immersion.workshop.aws/images/ConfigureCRR_0.png)](https://redshift-immersion.workshop.aws/images/ConfigureCRR_0.png)

2. 选择 *Yes* 单选按钮以启用复制。 选择*us-east-2* 的目标区域。 由于集群已加密，因此您必须在其他区域中建立授权以允许重新加密快照。 为选择快照副本授权选择*创建新授权*。 将 Snapshot Copy Grant 命名为值 *snapshotgrant*。 点击*保存*。[![img](https://redshift-immersion.workshop.aws/images/ConfigureCRR_1.png)](https://redshift-immersion.workshop.aws/images/ConfigureCRR_1.png)
3. 为了演示跨区域复制，启动手动备份。 单击*操作* -> *创建快照*。[![img](https://redshift-immersion.workshop.aws/images/Snapshot_0.png)](https://redshift-immersion.workshop.aws/images/Snapshot_0.png)
4. 将快照命名为 *CRRBackup*，然后单击 *Create snapshot*。[![img](https://redshift-immersion.workshop.aws/images/Snapshot_1.png)](https://redshift-immersion.workshop.aws/images/Snapshot_1.png)
5. 导航到您的快照列表并注意正在创建快照。

```
https://console.aws.amazon.com/redshiftv2/home?#snapshots
```

[![img](https://redshift-immersion.workshop.aws/images/Snapshot_2.png)](https://redshift-immersion.workshop.aws/images/Snapshot_2.png)

1. 等待快照创建完成。 状态将是*可用*。[![img](https://redshift-immersion.workshop.aws/images/Snapshot_3.png)](https://redshift-immersion.workshop.aws/images/Snapshot_3.png)
2. 通过从区域下拉菜单中选择 *Ohio* 导航至 us-east-2 区域，或导航至以下链接。

```
https://us-east-2.console.aws.amazon.com/redshiftv2/home?region=us-east-2#snapshots
```

[![img](https://redshift-immersion.workshop.aws/images/Snapshot_4.png)](https://redshift-immersion.workshop.aws/images/Snapshot_4.png)

## 弹性调整大小

注意：实验室的这一部分将需要大约 15 分钟才能完成，请相应地进行计划。

1. 导航到您的 Redshift 集群列表。 选择您的集群并单击 *Actions* -> *Resize*。 请注意，如果您没有看到集群，则可能需要更改 *Region* 下拉列表。[![img](https://redshift-immersion.workshop.aws/images/Resize_0.png)](https://redshift-immersion.workshop.aws/images/Resize_0.png)
2. 确保选择了 *Elastic Resize* 单选。 选择*新节点数*，然后单击*立即调整大小*。[![img](https://redshift-immersion.workshop.aws/images/Resize_1.png)](https://redshift-immersion.workshop.aws/images/Resize_1.png)
3.当resize操作开始时，你会看到*Preparing for resize*的集群状态。[![img](https://redshift-immersion.workshop.aws/images/Resize_2.png)](https://redshift-immersion.workshop.aws/images/Resize_2.png)
4. 操作完成后，您将再次看到*Available* 的集群状态。[![img](https://redshift-immersion.workshop.aws/images/Resize_3.png)](https://redshift-immersion.workshop.aws/images/Resize_3.png)

## 在你离开之前

如果您使用完集群，请考虑将其停用，以避免为未使用的资源付费。
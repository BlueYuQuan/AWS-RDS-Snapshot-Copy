# AWS RDS快照复制
## 将RDS快照复制到第二个帐户以保持安全

背景
灾难恢复（DR）通常被认为是处理基础设施的大规模故障 - 例如整个数据中心的损失。在AWS中，通常会通过构建跨多个可用区域和区域的解决方案来缓解这类故障。但是，还有其他类型的“灾难”，包括合法访问AWS账户的人员意外或彻底删除资源。

这种'灾难'可以通过将关键资源的副本（AMI，CloudFormation模板，加密密钥和实例以及RDS快照）保存到第二个“故障保护”帐户中进行缓解，该帐户中的访问受到有限且严格控制。这种控制可以通过将密码和MFA设备分成多个员工来维护，需要两个人才可以访问。

这里提供的Lambda函数提供了一种方法，可以将最新的自动快照的每日副本从Live帐户中的一个或多个RDS实例复制到Failsafe帐户，并可用作上述策略的一部分。

概观
提供了两个Lambda Python函数。

第一个，rdscopysnappshots.py，每天运行在Live帐户中。它会删除前一天运行中剩余的任何快照副本，为每个RDS实例创建最新快照的新手动副本，与Failsafe帐户共享并发送SNS警报以指示快照可用。

第二个函数rdssavesnapshot.py运行在Failsafe帐户中，并在接收到SNS警报时触发。它会手动复制新共享的快照，并删除比设置的时间早的任何现有快照。

组态
使用前应配置以下变量：

rdscopysnapshot.py
变量	描述
INSTANCES	要获取快照副本的RDS实例标识符的列表，例如[“db-name1”，“db-name2”]
地区	RDS实例存在的AWS区域，例如“eu-west-1”
与某人分享	快照将被共享的故障安全帐户，例如“012345678901”
SNSARN	SNS主题ARN用于宣布手动快照副本的可用性，例如“arn：aws：sns：eu-west-1：012345678901：rds-copy-snapshots”
rdssavesnapshot.py
变量	描述
地区	存在数据库实例的AWS区域，例如“eu-west-1”
保留	快照保留期以天为单位，例如31
用法
在Live帐户中，使用rdscopysnapshots-role-policy.json JSON角色策略创建新的IAM角色。使用rdscopysnapshoy.py创建新的Lambda函数，并使用每日CloudWatch事件触发它。将新角色与新的Lambda函数关联起来。如上所述配置新的Lambda函数。

请注意，多次测试新功能是安全的。

在Failsafe帐户中，使用rdssavesnapshot-role-policy.json JSON角色策略创建一个新的IAM角色。使用rdssavesnapshot.py创建新的Lambda函数，并使用上面配置的主题从SNS中触发它。将新角色与新的Lambda函数关联起来。如上所述配置新的Lambda函数。

在这两种情况下，新的Lambda函数都会在创建任何快照副本时等待。这可能需要几分钟的时间。将新的Failsafe Lambda函数的超时设置为5分钟。将新的Live Lambda函数的超时设置为至少相同的数量 - 如果您配置了多个RDS实例，则可能会更长。

加密
这些功能已经使用未加密的RDS实例进行了测试。他们也应该使用加密的RDS实例。但是，如http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ShareSnapshot.html中所述，您需要在使用前从Live帐户共享适当的KMS加密密钥到Failsafe帐户。

致谢
对这些功能的初步思考大多来自Matt Johnson（mhj@amazon.com）。

这些函数本身部分基于https://github.com/xombiemp/rds-copy-snapshots-lambda中的函数。

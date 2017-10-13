title: EC2自动备份
date: 2017-10-13 09:37:12
tags:
- AWS
- EBS
- 自动备份

categories:
- 运维
---

昨儿我的一个同事不幸把AWS上的EC2搞坏了，幸而只是由于错误的权限设置导致服务器无法登录，上面运行的服务还可以正常访问。然后花了一下午时间的折腾，总算搞定了这个问题。方法很麻烦，但至少有效，其实就是再创建一个EC2，挂载有问题的EBS，然后更改设置...

今天的主题，并不是讲如何恢复，而是主要来看一下备份相关的设置。AWS的EC2管理后台，并没有明确的菜单来提供自动备份的设置，我搜了一圈，发现很多文章都是教你使用AWS CLI + 脚本来做这个事儿，甚至还有专门收费的帮你自动备份EC2的商用服务（每个月1刀）。

不可思议啊，全球最屌的云服务商竟然没有提供简化的自动备份配置？这不科学！后来我找到一个github项目[aws-missing-tools](https://github.com/colinbjohnson/aws-missing-tools/tree/master/ec2-automate-backup)，我差点就要这么做了。不过这个项目文档里缺少了配置AWS CLI的内容，所以我又去AWS官方文档里找，突然让我发现了一个[例子](http://docs.aws.amazon.com/AmazonCloudWatch/latest/events/TakeScheduledSnapshot.html)，尴尬了，这不是明明提供了自动化备份的功能么？只是设置的位置不是很直接而已嘛。

简单说，就是依靠AWS CLOUDWATCH提供的事件机制，结合你设置的规则来触发备份操作，反正你只要根据官网例子的步骤，1分钟之内就搞定了。我就不多啰嗦了。
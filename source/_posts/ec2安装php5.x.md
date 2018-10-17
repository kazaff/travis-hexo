title: ec2安装php5.x
date: 2018-10-18 09:37:12
tags:
- php
- ec2
- linux

categories:
- 运维
---


最近在aws上拼命装服务器，使用的镜像是：Amazon Linux 2 AMI 2.0.20181008 x86_64 HVM gp2。
它上面默认提供了aws的源，提供了很多常用的软件，但版本都比较新。我们的项目代码依赖php5.6，所以，尴尬了。

疯狂的搜了一遍，找到个可以安装的源：

```
sudo yum -y update

sudo yum install –y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

sudo wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo wget https://centos7.iuscommunity.org/ius-release.rpm

sudo rpm -Uvh ius-release*.rpm

sudo yum -y update

sudo yum -y install php56u php56u-opcache php56u-xml \
  php56u-mcrypt php56u-gd php56u-devel php56u-mysql \
  php56u-intl php56u-mbstring php56u-bcmath php56u-soap
```

如果你用的是nginx，那么你还需要装一下php-fpm管理组件：

```
sudo yum -y install php-fpm-nginx
```

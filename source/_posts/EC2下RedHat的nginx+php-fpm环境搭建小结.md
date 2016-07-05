title: EC2下RedHat的nginx+php-fpm环境搭建小结
date: 2016-07-05 09:37:12
tags:
- aws
- ec2
- php-fpm
- nginx
categories: 运维
---

这几天在折腾AWS上的环境，免费套餐由于配置太低，原本打算放的项目临时调整，所以现在需要搭建wordpress环境（nginx，php），原本so easy的事儿，结果折腾了一天半，恶心坏了啊~

### nginx

由于ubuntu下安装php-fpm死活失败，查了一下竟然说官方仓库中的版本有问题，原本我就对ubuntu不太熟，一看又来个官方问题，玩儿蛋去吧，果断更换成RedHat系统。

结果谁知道，安装nginx时，默认的AWS下的RedHat源里没有啊。使用[这里](http://www.if-not-true-then-false.com/2011/install-nginx-php-fpm-on-fedora-centos-red-hat-rhel/)提供的方法添加对应的源，就可以顺利安装nginx。


### php-fpm

记得安装之前，先执行：
```
yum remove php*
```

然后，我没有安装7.0版本的php，所以，上面给的那个文章中的php安装部分不适合我，
```
yum install php56w php56w-fpm php56w-mysql php56w-gd
```

然后按照之前的一篇[文章](http://blog.kazaff.me/2016/05/21/centos%E4%B8%8B%E5%AE%89%E8%A3%85nginx+php-fpm/)来配置nginx和php-fpm即可，最后执行：
```
service nginx restart
service php-fpm restart
```
开启对应服务即可。


### mysql

这里我们不需要在EC2上安装mysql，直接连接RDS即可。不过这里还会碰到一个问题：**数据库创建连接失败**。

排除了rds地址，端口，用户名密码填写失败的问题后，我们只能把思路放在系统级别了，[这里](http://stackoverflow.com/questions/4078205/php-cant-connect-to-mysql-with-error-13-but-command-line-can)提供了解决方案：
```
setsebool -P httpd_can_network_connect=1
```
至少，我是靠这个办法搞定的，别问我为啥，不造啊~


### 文件上传失败

wordpress下上传文件，系统会提示下面这个报错：
> the uploaded file could not be moved to wp-content/uploads/2016/05.

就是这个问题，让我折腾了一整天，阿西吧！一直以为是因为nginx和php-fpm所使用的权限导致的，试了各种设置，完全不行啊~

最后老思路，既然不是软件环境的事儿，那就来看看操作系统呗，[这篇文章](http://abtechnologytalk.blogspot.com/2016/06/aws-ec2-apache-file-upload-and-move.html)就是解决方案：
```
/usr/sbin/sestatus
```
若该命令返回的结果中提示`SELinux enabled`，你就需要:
```
vi /etc/selinux/config
```
将**SELinux**的值从`enforcing`改为`disabled`,然后重启EC2即可。


### 总结

学艺不精啊，半吊子运维果然不给力啊~

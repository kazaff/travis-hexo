title: RedHat7的apache+php-fpm环境搭建小结
date: 2016-08-01 09:37:12
tags:
- php-fpm
- apache
categories: 运维
---

之前写过一篇[《EC2下RedHat的nginx+php-fpm环境搭建小结》](http://blog.kazaff.me/2016/07/05/EC2%E4%B8%8BRedHat%E7%9A%84nginx+php-fpm%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA%E5%B0%8F%E7%BB%93/)。

由于种种原因，这次要把nginx替换成apache，所以再总结一篇~~

要在RedHat上安装我们的目标环境，先得找到合适的源：

```
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
```

安装php环境，由于php依赖httpd，所以下面的命令会安装好apache2：

```
yum install php56w php56w-fpm php56w-opcache php56w-gd php56w-pdo php56w-mysql php56w-common
```

最后，我们还需要让apache感知到php-fpm，新增`/etc/httpd/conf.d/fpm.conf`文件：

```
ProxyPassMatch ^(.*\.php)$ fcgi://127.0.0.1:9000/var/www/html/
```

启动对应服务即可：
```
service httpd restart
service php-fpm restart
```

PS:
如果你也是在用AWS的话，建议初始化完EC2后创建个快照，以便日后快速恢复系统。

参考：
- https://webtatic.com/packages/php56/
- http://developers.redhat.com/blog/2014/04/08/apache-with-various-php-versions-using-scl/
- https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Managing_Confined_Services/chap-Managing_Confined_Services-The_Apache_HTTP_Server.html

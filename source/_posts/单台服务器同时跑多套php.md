title: 单台服务器同时跑多套php
date: 2018-10-21 09:37:12
tags:
- php-fpm
- nginx
- linux

categories:
- 运维
---

一套web系统，后台往往比前台功能要复杂丰满许多，但后台的访问权限往往更容易控制，毕竟前台是给宇宙中所有的生物在使用的。
为了安全起见，我们公司的一套web电商系统开始尝试做前后台功能隔离，根据整理出来的报告，后台功能需要大量的风险高的php内置函数。
前台基本上db的存存取取，没有什么图片管理，报表导出，文件打包等等功能，所以我们需要让它们分别跑在不同的php安全配置下。

服务器是一台AWS EC2，跑着默认提供的免费linux镜像，webserver用的是nginx，它使用fastcgi的方式把php交给php-fpm来处理。
所以，我们的方案就很直观了：

1. 为前后台搭建独立的虚拟主机；
2. 前后台各自的虚拟主机将php请求分配给各自的php-fpm管理器；
3. 运行两套不同php配置的php-fpm进程。


前两步关于nginx虚拟主机的内容这里就不多讲了，我们直奔主题：**同时运行两套php-fpm进程**

clone一套已有的php-fpm配置（`/lib/systemd/system/php-fpm.service`）:

```

[Unit]
Description=The PHP FastCGI Process Manager For Honmi Admin System
After=syslog.target network.target

[Service]
Type=notify
PIDFile=/run/php-fpm/php-fpm-admin.pid
ExecStart=/usr/sbin/php-fpm --nodaemonize -c /etc/php-admin.ini -y /etc/php-fpm-admin.conf
ExecReload=/bin/kill -USR2 $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

注意上面我们在`ExecStart`项中设置了php-admin.ini和php-fpm-admin.conf，这样新运行的这套php-fpm进程将会使用独立的php设置和php-fpm设置。

打开`php-fpm-admin.conf`文件，我们需要配置它使用的独立的端口号：

```
...
[www]
listen = 127.0.0.1:9001
...
```

默认那套跑的在9000端口上，这样我们现在就完成了第三步。

### 知识点

- 高版本的linux，不再使用`service`命令管理服务，所以使用它是总会看到`Redirecting to /bin/systemctl start xxxx.service`提示，因为系统默认会帮你转发到`systemctl`命令下

- 执行`systemctl enable xxxxx.service`来将服务设置成开机启动


### 参考资料

[添加服务到开机自动启动（centos7开机自启动nginx，php-fpm）](https://www.jianshu.com/p/b5fa86d54685)
[linux下多版本php共存的原理、方法](https://www.cnblogs.com/ningskyer/articles/5639276.html)

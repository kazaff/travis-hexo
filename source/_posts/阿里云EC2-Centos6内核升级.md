title: 阿里云EC2-Centos6内核升级
date: 2017-09-07 09:37:12
tags:
- 项目管理

categories:
- 运维
---

这几天收到了阿里云发来的短信，内容十分惊悚，吓得我直抽抽~~于是立刻登录控制台看看到底咋了！

原来是阿里云的云盾系统检测出我们公司的EC2上存在系统和软件漏洞。不过现在它提供的安骑士开始收费了，所以漏洞的详细信息无法查看，幸好给了7天的免费使用时间。

解决系统软件漏洞的方式简单粗暴，就是先来一波升级：

```
yum update
```

这样，基本上常见的软件版本过时导致的漏洞都可以修复。不过如果涉及到系统内核升级，我觉得还是要慎重一些，查到一个[博主的文章](https://www.szyhf.org/2017/01/07/%E9%98%BF%E9%87%8C%E4%BA%91%E4%B8%8Ecentos%E5%86%85%E6%A0%B8%E9%97%AE%E9%A2%98/)，建议还是根据自己的实际情况酌情对待。

如果你觉得可以升级内核，那么执行完上面的命令后，不要忘记reboot系统哦~

服务器重启完毕后，在阿里云控制台对漏洞信息进行验证即可~

不过，我发现，还是存在一些无法修复的漏洞，提示如下：

> 软件: kernel-devel 2.6.32-431.23.3.el6
> 命中: kernel-devel version less than 0:2.6.32-573.22.1.el6

查了一下，原来是因为系统存在多个kernel-devel导致的，可以执行:

```
yum info kernel-devel
```

把不需要的版本直接卸载掉：

```
yum remove kernel-devel-{version}-{release}
```

不需要重启哟，再去阿里云后台验证一次就可以了。
title: RedHat的cron配置BAD FILE MODE问题
date: 2016-07-20 09:37:12
tags:
- RedHat
- 计划任务
- crontab
categories: 运维
---

最近在AWS提供的免费EC2上跑了个Wordpress，用的是nginx+php-fpm。可能是由于免费ec2的配额是在无法容纳庞大的wp，也可能是别的原因，总之一个几乎没有流量的网站也会出现周期性的卡顿。
<!--more-->
由于没有思路进行代码方面的排查，所以准备写个计划任务来定期重启php-fpm的连接池来简单粗暴的规避上述问题。不过，竟然发现红帽子linux默认的crontab状态竟然提示：

```
crond[510]: (root) BAD FILE MODE (/etc/crontab)
```

其实原本不需要写这篇文章，因为上述错误信息你放在gg或bd上搜会找到大量的解决方案（其实中文帖子都是源自一个源，一个字都没改动）。我照着试了试，还是不行，看了一下一个英文论坛上的描述，其实差别只是一个`0`，但是对于我这种半吊子来说，也是很致命的：

```
chmod 0644 /etc/crontab
service crond restart
```

`644`和`0644`的差别：

先来看一个[提问](http://serverfault.com/questions/344544/what-is-the-first-digit-for-in-4-digit-octal-unix-file-permission-notation)，下面跟的答案其实说的不是太明确，但应该意思没错。

我们来看一个[国产贴](http://www.gwalker.cn/article-213.html)。这篇文章写的很屌，而且举了一个很好的例子。虽然我看的还是有点晕，但相信有经验的运维人员已经搞明白了。

不负责任的说，那个`0`表示的就是关闭掉`setUID`权限设置（避免过多的开放权限给普通用户），这应该是crond对配置文件的要求吧~

title: https证书申请
date: 2019-03-19 09:37:12
tags:
- https
- openssl
- ACME
- certbot
categories: 运维
---

时间飞逝，现在https已经是web项目必选项了。
早在很久之前，就有过一篇文章提[https](https://blog.kazaff.me/2015/06/01/https%E6%83%85%E7%BB%93/)，里面提供的证书申请方案已经过时。

其实，一直以来都有机构或组织提供openssl证书服务，最接地气的就应该是[Let’s Encrypt](https://letsencrypt.org/)。
去年年初，我们公司搭建环境的时候，就是用的是它提供的[ACME.sh](https://github.com/Neilpang/acme.sh/wiki/%E8%AF%B4%E6%98%8E)，可以看到它还是比较远古的，很多步骤，而且我们公司后来出现过它证书没有自动创建的问题~~

在一个夜黑风高的晚上，马上钻被窝的我，被领导电话夺命追魂，因为公司官网无法访问了，吓得我瞬间睡意全无。
打开电脑才发现是证书过期了~~

慌忙中我怎么有耐心执行完acme.sh的所有步骤~~不能够的！
google一查，原来已经有更智（lan）能（duo）的方案了：[certbot](https://certbot.eff.org/)

有了它，我们只需要两步即可完成证书替换：

```
$ wget https://dl.eff.org/certbot-auto
$ chmod a+x certbot-auto

$ sudo /path/to/certbot-auto --nginx
```

然后顺着导航提示回车即可。是不是很爽？

睡觉~

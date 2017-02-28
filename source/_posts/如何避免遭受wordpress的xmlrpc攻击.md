title: 如何避免遭受wordpress的xmlrpc攻击
date: 2017-02-28 09:37:12
tags:
- xmlrpc
- wordpress
- nginx
- php-fpm

categories:
- 运维
---

早上收到领导简讯，说公司官网无法访问了。omg，这怎么可能？第一反应是为啥360监控没有第一时间给我邮件提醒类？

然后立刻登录到服务器上，发现所有的配置都没有变更过，运行了几个月就突然挂了吗？而且从表象上来看，nginx依然坚挺，只是返回403错误而已。
这里需要注意的是，**打开配置文件之前一定要先查看一下核心配置文件的最后修改日期**，这样可以知道是否有其他人登录服务器修改过。

我毫无头绪的尝试了一些配置的修改后放弃了，因为一切都是那么的自然，毫无理由的就403了，nginx在，php-fpm也在。不过，nginx只能提供静态资源的请求，
所有对php的请求都直接403了。

最终我只能去翻看一下nginx的日志文件，发现了线索：

 > ... connect() to unix:/var/run/php5-fpm.sock failed (11: Resource temporarily unavailable) while connecting to upstream, client...

日志中还显示出了一个ip地址，查了一下是一个芬兰的地址，不停的请求`xmlrpc.php`，引起了我的关注。

将该地址加入黑名单后，瞬间网站就恢复了。

由于服务器配置不高，所有我之前给php-fpm分配的配额很低，所以大量的请求一下子就让php-fpm饱和了。所以就有了前面的403返回。

有趣的是，对方使用了一个固定的ip不停的发来请求，如果是大量不同的ip，我可能到现在都无法定位问题。这里还是要提醒大家：**线上系统突然出现问题，一定要第一时间去翻看日志**。这样可以省去大量的时间来猜疑，像我，一开始就认为是https配置的问题。

接下来我查了一下wordpress的xmlrpc攻击相关的文章，发现这个问题对于使用wp的网站来说似乎普遍存在。
[这篇文章](https://www.digitalocean.com/community/tutorials/how-to-protect-wordpress-from-xml-rpc-attacks-on-ubuntu-14-04) 有非常细致的剖析问题和提供了多种的解决方案，非常推荐看一下。

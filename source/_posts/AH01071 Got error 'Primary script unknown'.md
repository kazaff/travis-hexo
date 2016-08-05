title: AH01071 Got error 'Primary script unknown'
date: 2016-08-04 09:37:12
tags:
- php-fpm
- apache
categories: 运维
---

这几天有点神经衰弱了，被apache搞的！之前的一篇[日志](http://blog.kazaff.me/2016/08/01/RedHat7%E7%9A%84apache+php-fpm%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA%E5%B0%8F%E7%BB%93/)完成了ec2上的apache+php-fpm的环境搭建，一切都是那么的自然。

可谁知道，环境相当的稳定，间歇性的会导致apache或php-fpm服务挂掉，真是哔了狗了！在apache的error日志文件中看到了大量的报错：

>　AH01071 Got error 'Primary script unknown'

<!--more-->

GG了一大圈，感觉没有一个对口的解决方案，只能自己来猜！最后发现可能导致该报错的原因是：由于我们把php请求交给php-fpm进程来处理，那apache就不应该再管这些请求了，所以在尝试把默认的apache对应的php模块都关闭后，就再也看不到该报错了。

怎么关？我的环境下，apache的相关配置文件在`/etc/httpd/`，只需要就将下面两个文件备份删除，重启服务即可：

- /etc/httpd/conf.d/php.conf
- /etc/httpd/conf.modules.d/10-php.conf

目前已经感觉不会再报该错了，留院观察几天吧~

##补充

根据本人的观察，发现还是会有上述的报错，证明我们的思路是错的！不过奇怪的是上面的设置确实会导致这个报错大量减少，是我的错觉么？
为了避免大量的错误信息淹没我的磁盘，我默默的设置了清理日志文件的定时任务，这算是什么？掩耳盗铃么？娃哈哈~

推荐一下360的[网站监控](http://jk.cloud.360.cn/)服务，很贴心，免费套餐的配额也是很给力的。根据两天的观察，确实已经稳定了。但不知道是为啥会随机的服务挂掉。

继续留院观察吧~

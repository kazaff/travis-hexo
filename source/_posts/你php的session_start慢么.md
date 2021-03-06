title: 你php的session_start慢么
date: 2018-12-18 09:37:12
tags:
- session
- 文件锁
categories: php
---

事情发生在一个晴朗的早上，客户投诉说公司运营的一个web电商系统出现偶发的504问题。“偶发”这种事儿，真心不好排查啊。
我们检查了一遍所有的相关配置，包括nginx，php，php-fpm，mysql等等等等，调整了一些参数后情况并没有变好~~
不仅如此，还有相当多的其它因素干扰着问题的排查，这其中就包含今天的这个主题。

实在没辙了，只能开启所有的log来追查了。

经过对日志的分析，发现“偶发”的504都会伴随着几次php的slow log记录，其中包含了两个比较惹人注目的函数调用：curl和session_start。
前者很好理解，肯定是curl的目标地址出现了响应问题，经过确认也证实了如此，原来是之前依赖的一个第三方接口停止服务了。
而之所以“偶尔”，是因为这个服务有时候响应null，有时候直接阻塞到超时。。。

这不是今天的重点，我们来看session_start慢的问题。在用户浏览器上表现的行为是，当出现了某个页面504，接下来的一个时间段内，
该浏览器访问同一个二级域名下的所有页面都会返回504，但立即切换浏览器或更换二级域名即可立即正常访问。

同事提出一种看法：所有的同一个会话的请求都会最终交给同一个php-fpm的worker进程来处理，所以会出现这种现象。
这和我的认知有点不同，毕竟要想实现session粘度处理，一般往往是要在整个请求链路的各个节点都要做设置的，
不可能默认就提供这样的“高级功能”才对。可问题表现出来的情况恰恰又和同事的看法吻合，一时间很苦恼。

那就按照这个观点，我们调整了php-fpm的一个参数：request_terminate_timeout。
将它开启并设置一个较短的等待时间，确实发现上面提到的那个浏览器的504等待时间缩短了，
这仿佛又一次验证了同事的观点。

一直到排查的尾声，我也一直没有特别接受这个观点，直到我们看到了session_start的慢日志。
第一感觉是，一个内置函数怎么可能会慢呢？GG了一下，发现大量的相关文章，原来是文件锁导致的。
简单的说，php默认的session机制是靠文件，浏览器携带的cookie中指明的session_id到服务端后，
我们的php脚本会在入口统一调用session_start来开启会话，**假如一个请求开启session后卡住了，后面的同一个会话的请求就会在
session_start调用上等待**，直到第一个请求释放文件锁。对应我们的问题场景就是处理第一个请求的worker进程被kill。

这对这种问题，大家给出的方案是，依靠php提供的session handler机制把session存储在memcache或db里。
或者使用`session_write_close()`方法来释放写锁，在session_start后直接调用该方法，可以化解锁冲突，
而且你依然可以继续读取session中的值，只是无法编辑，若想编辑，再次调用`@session_start()`函数即可。

长远来看，还是推荐大家讲session存储在独立的节点上，方便日后的集群扩展。

今天的分享就到这里~

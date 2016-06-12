title:  gmailWatcher之node
date: 2016-05-01 09:37:00
tags:
- gmail
- imap
- async
- mailparser
categories: nodejs
---

Email是一个可能比很多人岁数都大的“上古神器”，尽管和它同期出生的小伙伴很多都已经寿终正寝，但它却依然坚挺，可见其代表了多大的用户刚需！

最近热映的《北京遇上西雅图2》讲的就是邮件的爱情故事，正巧又碰上项目新需求包含mail的收发要求，就借此良机好好学习一下这个陪伴我们很久的大伙伴。
<!--more-->
### 理论知识

如果你是在不知道什么是Email，那我只能说：看看这篇生动的[微小说](http://mp.weixin.qq.com/s?__biz=MzAxOTc0NzExNg==&mid=2665513086&idx=1&sn=ad45995d6c6e355d4dc7cf90815560a8#rd)吧。

好，我们来先聊聊几个关键词：

- [邮件协议](http://baike.baidu.com/view/2367542.htm)：LDAP、SMTP、POP、IMAP
- 邮件服务器：商用厂商（gmail.com、163.com）、自建邮件服务
- 邮件客户端：原生客户端（outlook）、网页邮箱系统、浏览器插件、自建邮件客户端

提到“自建邮件服务器”，勾起了我痛苦的回忆，之前在前任公司为了在centos上手动搭建邮件服务，让我反复安装操作系统N次（不要问我为什么）。所以这篇文章并不是讲如何搭建邮件服务器的，我们会使用gmail作为我们的邮件服务器。

由于我们的主题是邮件获取，所以POP和IMAP协议是我们主要需要了解的内容，这里我使用的是 **IMAP**，因为[它新](https://zh.wikipedia.org/wiki/IMAP)。

### 需求分析

我们要做的是这么一件事儿：

> 监控我们指定的一个gmail帐号，一旦该邮箱收到邮件，就解析邮件内容（包括附件）,并将该邮件标识为已读（确保不会重复处理该邮件）。

目标明确，可以开始写代码了！等等，其实如此常见的需求一定已经有大量的实现方案，不管是php，java，还是python，或是今天我们要使用的nodejs，都已经有现成的库或框架来供我们使用了。

不过我们上面的理论知识可以辅助我们知道，要实现这么一个需求，我们的思路是：

1. 使用对应协议和目标邮箱建立链接
2. 获取满足条件的邮件数据
3. 根据协议解析数据，获取我们需要的属性（发件人，标题，内容以及附件）
4. 更新该邮件的状态为已读
5. 持续监听“新邮件”事件，重复第2步

这里面我们忽略了一些细节复杂问题，如：

- 邮箱安全认证问题
- 邮箱链接中断处理
- 邮件解析失败
- 大邮件解析资源损耗（主要是大附件导致的内存损耗）
- 海量邮件的并发处理模型
- 其它

### 现有库

相信现有的解决方案可以帮我们完成上述的各个方面，那么nodejs为我们提供了怎样的现成代码呢？经过一番搜索，锁定了这个库：[mail-listener2](https://github.com/kazaff/mail-listener2)(注意这里给的链接是我已经fork后的，并加了一些中文注释和小修改)。

我已经在其中的关键部分添加了近乎啰嗦的注释，所以这里就不重复制造垃圾了。这里需要特别说明一下的是，该库使用了[aysnc并发库](http://blog.fens.me/nodejs-async/)，依赖其提供的并发模型让实现代码简化了不少，推荐大家在自己的项目中也大胆使用。

还有就是，如果你针对附件解析开启了流处理，那你需要自己来处理附件流，该库的并没有在开启流处理后帮你把附件保存在指定位置，不过扩展起来也很简单，只需要在对应的位置添加：

```javascript
//替换该行fs.writeFile(self.attachmentOptions.directory + attachment.generatedFileName, attachment.content, function(err)
attachment.stream.pipe(fs.createWriteStream(self.attachmentOptions.directory + attachment.generatedFileName+attachment.generatedFileName));
```

除此之外，该库并没有保留"node-imap"中的关于超时的相关参数，这会导致网速不太好的环境无法使用，建议后期加上下面俩参数：

- connTimeout 创建连接的超时时间，node-imap默认是10000毫秒
- authTimeout 连接创建完毕后认证的超时时间，node-imap默认是5000毫秒

这个库其实只是简单的封装了一下[node-imap](https://github.com/mscdex/node-imap)和[mailparser](https://github.com/andris9/mailparser)。

这俩库才是真正完成我们前面说的步骤，其中"node-imap"学习起来成本较高，主要是你得先了解imap协议本身。在协议之上该作者增加了nodejs的一些语言特性，靠事件来组织业务，所以需要静下来好好看看。

### 异常处理方案

目前为止，基本上除了异常处理，其他工作都被库做完了，娃哈哈，生活就是如此美好！那，我们应该如何处理异常才比较合适呢？

这还是要看业务需求，如果你开发的是一个用户参与度很高的客户端，那你只需要在异常发生后上报给用户即可，例如弹窗。

但如果是一个后台常驻服务，就不仅要考虑上报异常的方法，还需要考虑如何保证碰到无法恢复的异常后应用线程会自动重启，听起来很高大上，但其实[PM2](http://pm2.keymetrics.io/)已经帮你实现了。除此之外，国内BAT好像也都有自己的nodejs运维方案供我们选择。

### 结尾

相信已经说的够详细了，剩下的就靠大家留言了！

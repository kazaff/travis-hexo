title: Request使用proxy导致的socket连接数泄漏问题
date: 2016-12-30 09:37:12
tags:
- socket
- proxy
- Request
categories: nodejs
---

这是一篇快餐文章，源于前天接到了服务器预警邮件，提示：连接数超过阀值。

快速定位问题，发现是因为之前跑的一个nodejs开发的小程序，该程序用于定时去指定网站采集相关数据的。

由于要采集的网站需要设置翻墙代理，很麻烦的。之前直接使用Request来发http请求也没碰到这个问题。所以归结为是由于使用proxy不稳定，导致大量的socket异常。

在进入主题之前，先吐槽一下[Request](https://github.com/request/request)这个库，它的文档看似详实，但如果你仔细阅读就会发现很简陋。很多地方感觉欲言又止，很是让人烦躁。

尝试去看它的源码，发现它返回异常的方式采用了2种不同的方法（一般异步方法都如此处理）：

- 沿用nodejs的回调返回异常风格
- 发布异常事件

这也是一开始总是被无法捕获的异常导致程序crash的原因。

去github上的issues翻到别人的异常捕获异常方法，也是醉了：

```javascript
Request
.get({/*some config*/}, function(err, response){/*回调逻辑，第一个参数为error*/})
.on('error', function(error){
  // 再次绑定error事件
  console.error('Request on error');
  console.error(error.stack);
});
```

按道理说，这两种方式同时只需要用一种，同时使用两种则会发生重复调用问题。但如果我们没有绑定error事件（只采用回调风格），socket级别的异常是无法捕获得到的，这也就导致了上面的方案。不过经过绑定了这个error事件后，发现几乎Request触发的所有异常都被其捕获到了，不会再因为无法捕获的异常而导致程序crash了（还存在一些异常，我建议`process.on('uncaughtException')`还是不能省）。

接下来主题，先看一下一个[issue](https://github.com/request/request/issues/2440)，题主已经详细描述了问题也给了解决方案。
实测了一天，感觉确实解决了，至少从图标中观测正常许多了：

![](http://pic.yupoo.com/kazaff/G7tod4Cy/medish.jpg)

如果去看源码，会发现很绕，毕竟到处可见的回调和事件流。不过从解决方案上来看，[很直观](https://github.com/koichik/node-tunnel/pull/21/commits/6218c612d005d0cbab3cf133ccd20b9a21a2aa4d)。思路就是在: **确保在链接出现异常后也触发回调，在回调中来处理善后工作，最终会回收相关资源，而不是直接清除掉必要的事件监听器**。

去看源码吧，你将会收获更多。

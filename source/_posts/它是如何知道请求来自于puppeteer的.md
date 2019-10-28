title: 它是如何知道请求来自于puppeteer的
date: 2019-10-28 09:37:12
tags:
- puppeteer
categories: nodejs
---

一直以来，在采集的道路上，都是puppeteer与我相伴，感觉无往不利，无坚不摧。
但内心是知道总有一天，随着越来越规范，你使用puppeteer的目的会被限缩到固定范围的，毕竟它存在的意义是端到端测试。
而我们拿来作为采集数据的工具来用，总不算是正路~哇哈

闲话不多说，我们快入主题，我最近发现之前一直work的采集程序最近突然失败了，然后经过一番排查，发现目标网站识别出
请求是来自于非人类的，而拒绝登录了。好家伙，没想到这一天来得这么快~好歹等我交接出去啊~~😂

期初以为是对方识别user-agent来做出判断，但我设置了各种常规user-agent，并使用抓包工具确认设置成功，但依然无法突破！
这就奇怪了，也确实没有什么服务器可以用来识别的请求头了啊~

查了一圈，最后还是在puppeteer的社区求得了大神的帮忙。原来确实这种基于浏览器编程接口的模式下，浏览器会默认设置一个标识位：
`navigator.webdriver`，如果目标系统用js脚本判断这个变量是否被设置，就可以识别出本次访问到底是人类还是程序了。
而这种机制，就是我前面提到的标准。puppeteer官方也不希望这个工具未来会被滥用以至于被唾弃。

不过进一步查了一下，`navigator.webdriver`这个标识位不仅仅用于puppeteer，其它界面测试的套件也都会设置这个标识位。
下面说一下暂时如何突破这一点的：

```javascript
const browser = await puppeteer.launch({headless: false, ignoreDefaultArgs: ["--enable-automation"],});
```
目前，我们可以在启动puppeteer的时候，忽略`--enable-automation`这个设置来避免`navigator.webdriver`标识位被初始化。
但我觉得未来可能就不会再有效了~

除非，我们基于chrome源码自己开发一个浏览器来做我们想做的事儿~~

### 参考文献

[一行js代码识别Selenium+Webdriver及其应对方案](https://www.kingname.info/2019/02/12/hide-webdriver/)
[puppeteer slack](https://app.slack.com/client/T8WQY2F8Q/C8XEP1718)

title: 谈puppeteer碰到的bug
date: 2019-08-22 09:37:12
tags:
- puppeteer
categories: nodejs
---

什么是puppeteer？直观点说，就是一个提供以编程的方式控制浏览器（chromium）的nodejs库，非常的彪悍。
有了它，你可以用来做端到端测试，也可用来采集一些比较复杂的网站。我之前文章有涉及到这个神器，所以就不再啰嗦了~

这篇文章只是记录在使用中几个碰到的坑~

### page.click 无限阻塞

从官网手册上看，`click`函数在发现页面上没有对应的元素的时候，会报错。但我在实际使用的时候发现，`puppeteer 1.12.x`版本下程序可能会无限阻塞在这个函数上，不抛异常，也不会返回。而且发生这种情况的时候，页面上确实是有对应的元素的，手动点击该按钮也是可以正常触发页面响应的。

此外，注意到一点是，当出现这个情况的时候，我切换成下面的写法:

```javascript
await page.evaluate(() => {
  jQuery('button').click();
});
```

程序会直接报错，提示 **执行期间，上下文被销毁** 的异常信息。正常情况下，该方式和直接`page.click`都应该是可以触发按钮点击的，现在突然都失灵了~
更增加排查难度的是，在循环执行时，一开始是成功的，平均到第三次时，发生这个bug的可能性就非常大。同样的代码，使用`puppeteer 1.19.x`则完全正常！

和同事讨论了一下，他之前也碰到过这个问题，后来在网站找到另外一个解决方案：

```javascript
// 直接click方法，会导致莫名其妙的阻塞在click调用上
let submitBtn = await page.$('button');
let submitBox = await submitBtn.boundingBox();
let boxX = submitBox.x + (submitBox.width / 2);
let boxY = submitBox.y + (submitBox.height / 2);
await page.mouse.click(boxX, boxY, {delay: 0});
```

这段代码就完全可以解决问题，它使用了更底层的一套接口，直接操作鼠标。。。不过实际使用的时候需要注意，**该方式必须要将headless设置成false**。

所以，目前的结论是，如果你不想升级版本的话，就是用鼠标接口，否则可以升级sdk版本来解决。

### page.click 点击无效

依然是和 `page.click` 函数相关，在我们项目的某个页面（日历插件），发现`puppeteer 1.19.x`下，点击后无法影响该输入项的当前值。
但降低版本到`puppeteer 1.18.x`后，问题就解决了。

目前怀疑是最新版本的puppeteer可能存在bug（我们测试的时候，1.19.0才刚更新25天）。


以后有新的坑，再更新这个篇文章吧~
祝大家顺利~

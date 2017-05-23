title: gmail模拟登录失败处理
date: 2017-05-22 09:37:12
tags:
- gmail
- 安全认证

categories:
- nodejs
---

不管任何开发语言，收发邮件相关的库都是有的，而且用起来都非常简单。不过在实际项目中，还是会因为所使用的邮箱平台而出现一些问题，比较常见的就是安全认证相关的问题。

拿Gmail为例，它会提醒你一系列的安全问题，例如：

```
Error: Please log in via your web browser: https://support.google.com/mail/accounts/answer/78754 (Failure)

{"message":"connect ECONNREFUSED","name":"Error","stack":"Error: connect ECONNREFUSED\n    at exports._errnoException (util.js:742:11)\n    at Object.afterConnect [as oncomplete] (net.js:982:19)","code":"ECONNREFUSED"}

```

按照网上提供的解决方案，我们可以先在chrome中登录我们的目标gmail帐号，然后访问[https://myaccount.google.com/lesssecureapps](https://myaccount.google.com/lesssecureapps)，开启该设置后，很多人这个问题就搞定了。

但，我们并没有，可能是google更新了安全方面的限制，当我把项目部署上线后（放到了海外的vpn上），就又开始无法登录了。

随后我们还需要访问[https://accounts.google.com/DisplayUnlockCaptcha](https://accounts.google.com/DisplayUnlockCaptcha), 授权通过后应该就搞定了。

但，我并没有，还需要在[https://myaccount.google.com/notifications](https://myaccount.google.com/notifications)找到对应ip地址的请求拦截信息，点击“通过”，才算最终完成所有需要的配置。


### 赠送知识点

jQuery如何操作页面iframe中的dom节点：

- 父窗口操作IFRAME: window.frames["iframeSon"].document
- IFRAME操作父窗口: window.parent.document

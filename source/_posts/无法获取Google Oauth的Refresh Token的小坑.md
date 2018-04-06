title: 无法获取Google Oauth的Refresh Token的小坑
date: 2018-4-6 10:37:12
tags:
- google
- Oauth
- refresh token

categories:
- API
---

最近在项目需要和google的服务对接，google开放了大量的rest api来供第三方使用。而要使用其接口，第一步就是认证授权。

Oauth相关的内容老早就分享过，不过那时候应该是针对qq和微信，差别不大~~可以通过[这篇文章](https://blog.csdn.net/wangshubo1989/article/details/77980316)看到是如何快速简单的对接google Oauth2的。

现在分享东西，都免不了吐槽。google官方提供的文档很详实，但它引用的第三方类库就不那么给力了，[golang语言的类库](https://github.com/google/google-api-go-client)基本上就没有文档，用法全靠自己在网上摸索~~

不过，今天要说的这个注意事项和类库无关，是google oauth的一个设计特性。我在测试的时候，按照上面给的那篇文章的代码run了一遍，起初是卡在了“用code换取token”的这一步，总是提醒 **tcp io time out**。一开始以为是使用的这个类库过时了，毕竟官方文档例子中使用的接口url和类库源码中看到的不一样~~

后来才发现，原来是请求被墙了导致的，而之所以不是在用户授权页面就卡住的原因是因为我本地机器是基于sss代理来访问googl的，而这个代理并不能被程序使用。不过这个问题属于golang开发者的老生常谈问题了。推荐装一个 **Proxifier**，这样不用写一行代码就可以做到让电脑任何的请求都达到翻墙的效果了。

这算是个题外话。我们今天主要说的是，一旦你以上面的代码那样配置，并首次获取到了token后，你在改变一些配置时会发现并没有拿到期望结果~~总感觉你的代码和google之间有一层“缓存”的存在。**原因就在`prompt`这个参数**。官方文档里有对这个参数的详细解释，只是新手很难拿它和这个问题关联起来。

我碰到的表象是，我首次运行测试用例时，并没有设置`access_type=offline`，这就意味着我只能获取到access_token，而没有refresh_token。而前者的有效期一般只有1小时，而我们的项目需要长久的授权来达到数据同步的目的。

虽然我修改代码，增加了对应的设置：

```golang
url := googleOauthConfig.AuthCodeURL(oauthStateString, oauth2.AccessTypeOffline)
```

但谁曾想到重新运行后依然拿不到refresh_token。这就是我今天打算分享的坑。

我们显然已经解释过原因了：

> prompt
> Optional. A space-delimited, case-sensitive list of prompts to present the user. If you don't specify this parameter, the user will be prompted only the first time your app requests access.

该参数的作用用来设置是否重复提醒用户进行授权的，如果我们没有设置，它默认就是只在第一次尝试授权时提醒用户。之后你修改了参数，google还是基于第一次授权时的配置进行响应。

网上找到了另外一个解决方案，不过需要用户配合：用户访问[https://myaccount.google.com/u/0/permissions](https://myaccount.google.com/u/0/permissions)页面，该页面时google提供给用户用来管理自己授权信息的，用户可以在页面列表中找到我们的project，然后删除掉现有的这个授权即可。

这样，再次运行实例，用户就需要重新授权，我们也就能提供新的配置参数，从而获取到我们想要的值了。意不意外？惊不惊喜？真不真实？

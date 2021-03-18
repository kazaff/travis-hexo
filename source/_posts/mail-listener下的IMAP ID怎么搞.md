title:  mail-listener下的IMAP ID怎么搞？
date: 2021-03-18 09:37:00
tags:
- 163邮箱
- unsafe_login
- mail-listener

categories: nodejs
---

今天发现一直使用的nodejs的一个IMAP库，无法完成国内163邮箱的登录授权，会提示：

> Unsafe Login. Please contact kefu@188.com for help

太讨厌了。然后一顿乱搜，原来是163邮箱在登录认证的时候，需要客户端携带特定的身份参数。
文章后面的参考资料里提供了官方的解释以及另一个博主的分享。

不过可惜，这些资料里提供的demo都不是基于nodejs的。所以还是只能靠自己来解决~~
经过进行源码阅读，发现其实nodejs的底层`imap`库也是提供了对应的方法的，只是奇怪的是无法直接使用。相关代码如下：

```javascript

let { MailListener } = require("mail-listener5");
let mailWatcher = new MailListener({
	username: "eyusei@163.com",
	password: "MSZOYXXOTHNTEERI",
	host: "imap.163.com",
	port: 993,
	tls: true,
    //debug: console.log,
	tlsOptions: {rejectUnauthorized: false},
	mailbox: "INBOX",
	searchFilter: ["UNSEEN"],
	markSeen: true,
	fetchUnreadOnStart: true,
	mailParserOptions: {streamAttachments: false},
	attachments: false,
	attachmentOptions: {directory: "./attachments/"}
});

mailWatcher.start();
mailWatcher.imap.id({"name": "kazaffClient", "version": "1.0"});

```

运行后会直接报错：

> Server does not support ID

进行debug后发现，原来在调用`imap.id`方法时，直接就被拒绝了：

```javascript
Connection.prototype.id = function(identification, cb) {
   if (!this.serverSupports('ID'))
     throw new Error('Server does not support ID');
  ......
```

[完整代码](https://sourcegraph.com/github.com/mscdex/node-imap/-/blob/lib/Connection.js#L385)看这里。

尝试将385~386两行注释掉后，就完全ok了。
分析后果感觉是因为163邮件服务那边要求身份认证的时机和`imap`库认为的发送认证时机不匹配导致的。
直接注释掉这个校验，由于163邮件服务器肯定支持`ID`校验，所以不会有任何问题。

唯一不开心的问题是直接修改源码导致后期升级麻烦。不过至少可以先跑起来了~~

不知道有朋友知道更好的办法没有？请教了~

### 参考资料

[解决网易163邮箱Unsafe Login.错误](https://blog.yrpang.com/posts/45207/)
[imap连接提示Unsafe Login，被阻止的收信行为](https://help.mail.163.com/faqDetail.do?code=d7a5dc8471cd0c0e8b4b8f4f8e49998b374173cfe9171305fa1ce630d7f67ac211b1978002df8b23)
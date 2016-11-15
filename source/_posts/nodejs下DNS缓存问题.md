title: nodejs下DNS缓存问题
date: 2016-11-15 09:37:12
tags:
- dnscache
- request
- window
categories: nodejs
---

无意间看到一个[文章](http://www.madhur.co.in/blog/2016/05/28/nodejs-dns-cache.html)，是关于nodejs下发送http请求不会缓存dns结果的。这意味着，如果你基于nodejs写了一个http采集程序，不提供dns缓存则会让每次请求都傻傻的重复解析域名为ip地址。听起来会非常影响性能不是么？

我的项目中，发送http请求并不是使用的node原生的http库，而是依赖一个常用的`Request`库。我查阅了一下该库的相关文档和github issue，也发现了一些和dns相关的帖子。不过多数说的是，关于dns问题，本身并不是`Request`库的范畴，而归结于nodejs的内核问题。omg，感觉好深奥啊！

幸好，上面提到的那篇文章中也提出了两个解决方案：

- 应用级别：[dnscache](https://github.com/yahoo/dnscache)
- 操作系统级别：[Bind](https://www.isc.org/downloads/bind/), [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) 和 [unbound](http://unbound.net/)

不论是哪个方案，看起来似乎都很简单，只是安装并初始化即可。但问题是，我们怎么来验证它们真实有效？由于我本地的开发机操作系统环境是win7 64bit，所以上文提到的操作系统级别的方案我无法测试。那我们就来看一下应用级别方案到底是否有效吧~~

首先，我们需要让win能追踪dns请求，这里我找到了一个[软件](http://www.nirsoft.net/utils/dns_query_sniffer.html)，下载后不需要安装直接运行即可。然后，我们还需要一个清除缓存的方法，可以看[这里](http://www.nenew.net/windows-dns-cache-clean.html)，简单说就是在终端中执行：

> ipconfig /flushdns

工具就准备完毕了，我们创建一个测试脚本：

```javascript

const Request = require('request');

function fetch(url, callback){
	Request.head({
		url: url,
		timeout: 10000,
		tunnel: true,
		gzip: true,
		proxy: false,
		followRedirect: false
	}, callback);
}

let now = Date.now();
fetch('http://blog.kazaff.me', function(err, response, body){
	console.log('lookup time without cache: ', Date.now() - now);
});

```

好的，现在打开`DNSQuerySniffer`，然后先清理一下本地DNS缓存，一切就绪后执行我们的测试脚本`node test.js`。你会在`DNSQuerySniffer`中看到一次DNS请求及其相关信息。在一定的时间间隔内，反复运行我们的测试脚本你会发现并不会再次触发DNS请求，这说明什么？**我的win7环境本身就自带操作系统级别的DNS缓存**（只是缓存时间很短）。

修改我们的测试脚本如下：

```javascript

const dnscache = require('dnscache')({
	"enable": true
});
const Request = require('request');

function fetch(url, callback){
	Request.head({
		url: url,
		timeout: 10000,
		tunnel: true,
		gzip: true,
		proxy: false,
		followRedirect: false
	}, callback);
}

let now = Date.now();
fetch('http://priceline.com', function(err, response, body){
	console.log('lookup time without cache: ', Date.now() - now);

	setTimeout(function(){
			now = Date.now();
			fetch('http://priceline.com', function(err, response, body){
				console.log('lookup time with cache: ', Date.now() - now);
			});
	}, 2000);
});

```

这次我们在执行测试脚本后，快速清空本地DNS缓存（如果你手速不快，可以适当延长setTimeout的触发间隔），你会发现，两秒后的http请求并没有重新查询DNS，这说明什么？很明显，**我们的应用自己维护了DNS缓存**，所以第二次请求根本就不会关心操作系统本地是否存在对应的DNS缓存记录。

SO，我们如果不想让自己的程序依赖操作系统环境，那现在就可以自己来维护DNS解析记录了，哇啦~

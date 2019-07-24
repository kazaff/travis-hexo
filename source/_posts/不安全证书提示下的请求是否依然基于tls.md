title: 不安全证书提示下的请求是否依然基于tls
date: 2019-07-23 09:37:12
tags:
- TLS
- HTTPS
- 证书
categories: 运维
---

好久没有更新Blog了，惭愧惭愧~

今天咱们来聊一个之前总是忽略的问题：

![](http://pic.yupoo.com/kazaff_v/c80e0b52/1eb41ad3.png)

我记得是好几年前，当时https风初见势头，很多大一些的互联网平台都https化的时候，我也跟风的研究并搭建过基于https的网站。不过那个时候证书还是需要花钱买的，所以跟着网上的教材我只能在本地搭建测试环境~

不过这几年随着https证书贫民化，几乎所有网站都切换到了https下，什么？你还不知道如何免费https化？强力推荐使用[Certbot](https://certbot.eff.org/)。

有点跑题了，拉回来拉回来。假如我们使用自签名的证书，用浏览器打开的时候就会碰到上面截图的提示。这其实是浏览器的保护措施，不过它的提示感觉有点“过度”保护了~之前没有深究过，突然想知道，在浏览器提示不安全的私密连接的基础下执意访问呢？是否会退化到http？

不光是自签名证书会碰到这个问题，只要是浏览器验证证书有效性的时候发现问题都会提示不安全，例如证书过期，域名不匹配等。

所以我本地搭建了个测试环境，然后利用抓包工具进行了分析，得到了最终的结论：`不安全的私密连接，依然是基于https的，只是使用的证书由于没有通过验证，所以很可能是中间人的恶意证书`。

本文章不会展开SSL和TLS的历史，也不会科普TLS握手流程，这些都可以在文章底部的参考链接中的文章里找到答案。

得到这个结论并非没有任何意义，假如我们使用`curl`命令或某个语言的http库，请求一个rest服务或基于http的api接口，在服务使用自签名证书的前提下，本次调用到底是否使用证书呢？

```shell
$ curl https://192.168.1.142
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.

```
可以看到，`curl`默认由于证书验证问题会最终放弃请求。所以我们必须明确告知`curl`忽略证书问题，可以使用这个参数`-k`。

那么如果是某种编程语言呢？我们来看几种语言的http库是什么情况：

### php
```php
curl_setopt($cHandler, CURLOPT_SSL_VERIFYHOST, false);
curl_setopt($cHandler, CURLOPT_SSL_VERIFYPEER, false);
```

### golang

```golang
transCfg := &http.Transport{
  TLSClientConfig: &tls.Config{InsecureSkipVerify: true}, // ignore expired SSL certificates
}
client := &http.Client{Transport: transCfg}
```

### nodejs

```nodejs
process.env["NODE_TLS_REJECT_UNAUTHORIZED"] = 0;
```

### java
```java
try {
    TrustManager[] trustAllCerts = new TrustManager[] {
       new X509TrustManager() {
    public java.security.cert.X509Certificate[] getAcceptedIssuers() {
        return null;
    }
    public void checkClientTrusted(X509Certificate[] certs, String authType) {  }

    public void checkServerTrusted(X509Certificate[] certs, String authType) {  }
    }
    };

    SSLContext sc = SSLContext.getInstance("SSL");
    sc.init(null, trustAllCerts, new SecureRandom());
    CloseableHttpClient httpClient = HttpClients.custom().setSSLHostnameVerifier(NoopHostnameVerifier.INSTANCE).setSslcontext(sc).build();

    String output = Executor.newInstance(httpClient).execute(Request.Get("https://127.0.0.1:3000/something")
                                      .connectTimeout(1000)
                                      .socketTimeout(1000)).returnContent().asString();
    } catch (Exception e) {
    }


    # source: http://stackoverflow.com/questions/2703161/how-to-ignore-ssl-certificate-errors-in-apache-httpclient-4-0
```

目前，小弟我只实际抓包过`curl -k`的流程，稍后我会尝试抓一下`golang`的请求看看是否符合预期。

## 抓包工具

如果你只是在浏览器端或postman测试，使用`fiddler`这款工具即可，简单够用~ 但我没有办法使用它抓到`curl`的包，所以只能拿神器`wireshark`来解决问题了。

不过友情提示，一定要去官方下载`wireshark`最新版（3.03+），安装的时候选择支持监听回环网卡（npcap loopback adapter），否则是无法抓住你本机回环的请求的。

更有意思的是，`wireshark`把我本机的内网ip（192.168.1.142）也当做回环地址来处理了，我本以为只要我不是用localhost或127.0.0.1，就应该可以监听到数据包的，结果花了好多时间才。。不说了，都是泪。

## 参考链接

- [SSL/TLS协议运行机制的概述](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)
- [TLS整理（下）：TLS如何保证安全](https://www.jianshu.com/p/24af67c40e8d)
- [TLS 详解](https://juejin.im/post/5b88a93df265da43231f1451)
- [如何用 wireshark 抓包 TLS 封包](https://cloud.tencent.com/developer/article/1416948)
- [细说 CA 和证书](https://www.barretlee.com/blog/2016/04/24/detail-about-ca-and-certs/)
- [HTTPS加密过程和TLS证书验证](https://juejin.im/post/5a4f4884518825732b19a3ce)

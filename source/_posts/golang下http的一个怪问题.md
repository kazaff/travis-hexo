title: golang下http的一个怪问题
date: 2018-3-19 10:37:12
tags:
- http
- rewrite
- body

categories:
- golang
---

最近用golang的`github.com/gorilla/mux`库来实现一个简单的http server。在实际测试的时候发现了一些问题，大多数都是因为自己不熟悉导致的，不过其中一个我觉得值得拿来分享一下。

大概情况是这样的，我计划实现一个简单的服务，接受post请求，并解析请求携带的json数据。一切都是这么的自然，和简单。不过部署测试的时候蒙了，客户端代码提交上来的请求，虽然正确的触发了服务端绑定的函数，但请求携带的json数据丢了。。更费解的是，用postman模拟提交相同的数据，就完全正常？！

这就很奇怪了，debug了一下，发现原来客户端代码和postman提交请求存在一个不同点，因为客户端代码请求时url是拼接出来的，导致了一个问题：

```
代码用的url： http://127.0.0.1//api/hola (注意这里出现了两个//)
postman用的url：http://127.0.0.1/api/hola
```
当然，postman的url才是我们希望使用的。修正这个后就一切恢复了正常！

这一点如果放在其它语言写的服务端中，是完全没有问题的。勾起了我的好奇心~
翻了一下源码，发现在`mux.go`定义的方法里有一个逻辑，注意下面代码中的第三个`if`判断，它会格式化请求时使用的地址，并尝试格式化，一旦它对url进行了任何形式的格式化，代码就会返回客户端一个301重定向响应，而这次重定向就导致了 **为何触发了服务端绑定的正确函数，但请求数据丢了** 。

```

...
func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	if !r.skipClean {
		path := req.URL.Path
		if r.useEncodedPath {
			path = req.URL.EscapedPath()
		}

		if p := cleanPath(path); p != path {
			url := *req.URL
			url.Path = p
			p = url.String()

			w.Header().Set("Location", p)
			w.WriteHeader(http.StatusMovedPermanently)
			return
		}
	}
  ...

```

说了这么多，为了避免这个问题，其实可以在初始化`router`时，设置一个开关，即可避免这个坑：

```go
router := mux.NewRouter().StrictSlash(true).SkipClean(true)
```

我不确定这个逻辑是不是还用于匹配其它场景，如果不能简单的设置这个标识位，那也可以在客户端代码中拼接url时做足够多的逻辑判断避免发生在我身上的问题。

祝好运~

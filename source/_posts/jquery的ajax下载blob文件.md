title: jquery的ajax下载blob文件
date: 2016-07-20 16:37:12
tags:
- jquery
- ajax
- blob
categories: 前端
---

好久没有解决前端问题了，下午同事问了我一个问题：**jquery的ajax怎么下载文件？**

乍一听有点蒙，之前用ng和react时也写过类似的功能，但是很顺利（所以忘记具体细节了）。jquery为啥会不行呢？看了一下具体场景，发现原来jq的ajax回调已经把response的数据傻瓜式的以字符串的方式解析了。

查了一下gg，发现国内的解决方案就是在该场景下不实用jq，而是自己手动创建`XMLHttpRequest`。虽然这个方法很可靠，但之前封装的jq的ajax就不能使用了。

查了查jq的文档，本打算自己根据jq提供的`jQuery.ajaxSetup()`接口来拓展数据类型，但怎么都搞不定。后来，在[github](https://gist.github.com/SaneMethod/7548768)上找到了一个大牛封装好的jq插件。

然后我们就可以这么写了：
```html
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8">
		<title>blob demo</title>
	</head>
	<body>
		<img id="img" src="" />
		<script src="//cdn.bootcss.com/jquery/2.2.1/jquery.js" charset="utf-8"></script>
		<script src="jquery-ajax-blob-arraybuffer.js"></script>
		<script type="text/javascript">
			$.ajax({
				url: "./face.jpg",
				type: "get",
				dataType: "blob",	//扩展出了blob类型
			}).done(function(data, status, jqXHR){
				var reader = new window.FileReader();
				reader.readAsDataURL(data);
			 	reader.onloadend = function() {
	        document.getElementById("img").src=reader.result;
			  }
			}).fail(function(jqXHR, textStatus) {
			  console.warn(textStatus);
			});
		</script>
	</body>
</html>
```

不过，从该插件的源码上来看，它也是手动构建了一个`XMLHttpRequest`对象来发送ajax，不过[兼容性](http://caniuse.com/#search=XMLHttpRequest)可能会成为问题。想深究的可以看[这里](https://segmentfault.com/a/1190000004322487)。

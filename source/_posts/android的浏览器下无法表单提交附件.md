title: android的浏览器下无法表单提交附件
date: 2015-12-04 09:37:12
tags: 
- 微信浏览器
- android
- 附件上传
categories: 移动端
---

今天刚发现一个bug，很小，但是很恶心：用android的内置浏览器无法上传表单附件。

gg了一下，发现这里讲的[方法](http://stackoverflow.com/questions/6569919/does-the-android-web-browser-allow-uploading-photos-just-taken-from-camera)貌似可行：

	<input type="file" name="photo" accept="image/*" capture="camera">
	
注意添加的那个属性：**capture="camera"**

本以为大功告成，谁知道微信内部浏览器还是不行，妈蛋，一搜才知道，好多人都反应这个问题，从13年就有相关提问，我也是日了狗了，怎么还是没修复？而且android下的微信内置浏览器连个刷新功能都没有，这尼玛是不是在耍贱呢？

好，那大家都是怎么解决微信下的这个问题呢？看到几个方案貌似大家比较赞同：

[方案一](http://www.zhihu.com/question/21452742)：大概意思就是一旦判断出用户所在的是微信环境，就引导用户切换到系统浏览器下，这算是成本最低方案了。

[方案二](https://segmentfault.com/q/1010000002479491)：使用的是微信提供的js-sdk，相当于让用户把图片先提交给微信服务器，然后在让自己系统的后台去下载，听起来就麻烦，而且貌似这个上传接口是需要去认证公众号，还有调用频次限制！

我觉得吧，这就是作！微信中明明看到了其使用的内置浏览器是基于qq浏览器的，单独使用qq浏览器就可以上传附件，怎么就在微信里就不行？！如果是基于安全考虑的，那为啥ios的微信就可以上传图片呢？！

好了不说了，睡觉……

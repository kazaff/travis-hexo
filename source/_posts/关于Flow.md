title: 关于Flow
date: 2015-05-27 09:37:12
tags: 
- flow
- facebook
- 类型检查
categories: 前端
---

最近感觉又back2js了，仅仅是几个月的暂别，就感觉js又tmd翻天覆地了￥Q@#@#$～

这次要科普的是[Flow](http://flowtype.org)，这玩意儿是FaceBook开源的一个**类型检查**库，从此写js就再也不"自由"了～
<!--more-->
都说FB的天才多，现在终于感觉到了，人家玩的都是语言扩展，顿时感觉高端大气档次高了啊！其实为若类型语言增加类型检查，JS并不是第一个语言，早先FB就给PHP改造过了……之所以我突然想起来科普这个，是因为最近在看React的代码，发现有很多看似眼熟又总觉得怪怪的代码风格，一开始以为是ES6的新特性，但是查了一下却没找到对应介绍，谁知道是Flow提供的，真是让我大开眼界啊！

看一个官方的demo：
	
	function add(num1, num2) {
  		return num1 + num2;
  	}
	var x = add(3, '0');
	console.log(x);
	
你说，`x`的值是啥？3？30？undefined？不管是啥，其实这都是动态类型带来的连带伤害，我们在以前的js/php编程的日子里上面这种场景屡见不鲜。但是FB的大大们坐不住了，非得改到爽才行：

	
	/* @flow */
	function add(num1: number, num2: number): number {
  		return num1 + num2;
	}
	var x: number = add(3, '0');
	console.log(x);
	
在Flow的世界里，你就得这么老实的写，这样，是个人都应该知道x应该是啥了吧？

这里我就好奇了，Flow提供的既然不是标准的js语法，那浏览器怎么可能理解？确实不能理解，别说浏览器，我的WebStorm都报错，怎么办？

我们需要在交给客户端之前转换为标准的js代码，官方提供了对应的[转换工具](http://flowtype.org/docs/running.html)，这样我们的js代码就可以在编写时拥有完善的类型检查，又可以直接在产品环境下运行了，这种思想和**less**如出一辙。

再往下聊，就需要我们同时对js这门语言和Flow提供的适配规则有很全面的了解了。所以我就不多说了，免得误人子弟～～
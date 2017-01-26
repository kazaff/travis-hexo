title: 看我如何搞定nodejs内存泄漏问题
date: 2017-01-04 09:37:12
tags:
- 内存泄漏
- memory leak
categories: nodejs
---

最近又用node写了一个小工具，需要常驻进程，经过几天的观察，发现内存占用有持续增加的趋势（虽然不明显，但还是让我察觉到了，我真屌）。突然发现，
我竟然不知道怎么排查nodejs的内存泄漏，吓死宝宝了！
<!--more-->
花时间看了一下相关资料（google真好，外果仁真屌），看来这部分也已经有比较完善的方法论+工具了。所以这篇文章记录一下自己从不懂到入门的经历~~
我希望这篇文章不仅能提供具体的工具供大家使用，还提供足够的理论知识来辅助大家思考，当然，也可能是我自己想多了~~哇哈

### 发现问题

由于没有太多运维经验，也不知道啥逆天的工具来帮我一键式监控所需要的指标，如果你和我情况一样，那我们只能手动来造个简陋的但够用的监控脚本了。

别告诉我你和我一样shell也不熟，直接就node吧。少废话~

先装上pm2，然后写一个脚本，来定时打印目标应用的内存使用率，当然，前提是目标应用也都放在pm2中管理。

```JavaScript
const exec = require('child_process').exec;
var Later = require('later');
var schedule = Later.parse.text('every 5 mins');	// 每5分钟正点触发
Later.setInterval(function(){
	exec('pm2 jlist', {	// 打印出pm2中应用的基本状态信息，输出是json字符串
		timeout: 2000
	}, (err, data, stderr)=>{
		if (err) {
			console.error(err, err.stack);	// an error occurred
			return;
		}

		//将结果写入日志
		data = JSON.parse(data);
		if(data[0]){	// 这里取0是因为我希望监控的应用在pm2中的顺序是第一位
			console.log(data[0].monit.memory/(1024*1024));	// 直接输出到pm2的log中
		}
	});
}, schedule);
```

然后就等一段时间，就会在对应的log文件中拿到相关的内存数据，然后只需要用电子表格生成一个图标即可，我推荐使用google drive的spreadsheet：

![](http://pic.yupoo.com/kazaff/G6ReFVIp/medish.jpg)

上面的图是我收集了大概2天的内存数据绘制成的图标，可以看出内存使用量成上升趋势。没错，就是泄漏了！！

![](http://pic.yupoo.com/kazaff/G6RfSV4G/medium.jpg)

**友情提醒：** 修复内存泄漏可能会耗时很久，你最好先找一个临时方案来维持生计，例如定期重启程序。

### 搭建环境

本着实战为主的策略，我们先从搭建内存泄漏监控环境开始。刚开始参考[node-memory-leak-tutorial](https://github.com/felixge/node-memory-leak-tutorial/)，以为会很顺利搭建好的，不过碰到了[这个error](https://github.com/node-inspector/node-inspector/issues/905#issuecomment-240930864)。看Issus应该是个很常见的错误，按照别人的解决方案，尝试[切换成`nodejs 6.3.1`版本](https://github.com/coreybutler/nvm-windows)进行了测试，确实可以绕过那个错误：

```
// 在项目目录下
node-debug leak.js
```

然后终端会启动你的chrome，并停在代码的断点位置，深吸一口气你就可以点击执行了。

**备注**：若遇到无法创建快照问题，需要多刷新几次哟~

其它工具我也顺便试了一下：

1. [node-memwatch](https://github.com/marcominetti/node-memwatch)
2. [node-webkit-agent](https://github.com/c4milo/node-webkit-agent)
3. [node-heapdump](https://github.com/bnoordhuis/node-heapdump)

因为它们都需要根据操作系统进行编译，我的本地环境是 **win7 64bit**，这并不是一个理想的nodejs环境，至少我这么认为，否则也不会碰到恶心的“.net framework”问题。我劝大家千万别学我轻易的就删除了 **.net framework 3.5** 这个安装包，因为这是win7自带的，删了以后就装不上了，而装更新的4.0+版本的话我这边很重要的一个软件就无法运行了（翻墙你懂的）。[在Windows 7系统上安装.NET Framework 3.5框架](http://blog.csdn.net/jasonsong2008/article/details/17248115)很不容易的！建议可以用上docker来搭建一个专门用来分析用的容器，这里我就不折腾下去了，its your turn~~

### nodejs内存分析的理论姿势

在开始听我正儿八经胡说八道之前，推荐你先看几个文档：

- [Memory Terminology](https://developers.google.com/web/tools/chrome-devtools/memory-problems/memory-101#show-last-Point)
- [How to Record Heap Snapshots](https://developers.google.com/web/tools/chrome-devtools/memory-problems/heap-snapshots#show-last-Point)
- [Debugging Memory Leaks in Node.js Applications](https://www.toptal.com/nodejs/debugging-memory-leaks-node-js-applications#show-last-Point)
- [Simple Guide to Finding a JavaScript Memory Leak in Node.js](https://www.alexkras.com/simple-guide-to-finding-a-javascript-memory-leak-in-node-js/)
- [Understanding Garbage Collection and hunting Memory Leaks in Node.js](https://www.dynatrace.com/blog/understanding-garbage-collection-and-hunting-memory-leaks-in-node-js/#show-last-Point)
- [writing fast memory efficient javascript](https://www.smashingmagazine.com/2012/11/writing-fast-memory-efficient-javascript/#show-last-Point)
- [Error Handling in Node.js](https://www.joyent.com/node-js/production/design/errors#show-last-Point)

一次性看完这些，可能要花很久，如此贴心的我已经帮你看过了，根据我的理解，总结如下：

- javascript的v8内存管理和java jvm类似，都有新生代（To-Space and From-Space），老年代等；
- 排查内存泄漏需要分析内存快照，可以使用已有的工具以devtool的profile面板或代码的方式创建snapshot；
- 创建的快照文件可以导入devtool的profile进行分析；
- 快照生成的最佳实践是：先保证程序已经预热，然后进行快照1（先触发GC），然后对程序进行一些交互（例如：对于web服务即http请求），再次创建快照2，如此循环来生成多个版本的快照；
- 合理的利用devtool的profile提供的功能，正确的选择视图；
- 理解profie中的字段含义：
	- 对象上的黄色标识表示的是javascript直接引用，红色表示间接依赖引用，不太需要关注的是无底色对象，其代表被其它资源引用（如：natvie code）；
	- profile会根据对象的构造方法对对象进行分组归类，每个组对应的“Shallow Size”表示的是该组对象的直接内存占用大小（例如：该类对象自身的原始类型数据的内存占用），对应的“Retained Size”表示的是该组对象依赖的其它对象而造成的内存占用总数（等于自身的Shallow Size + 依赖对象的Shallow Size [ + 依赖对象的依赖对象的Shallow Size [ + 递归下去]]）;
	- 由于性能原因，profile中不会显示对象的整型类型的属性，但是它们并没有丢失，仅仅是工具没有显示出来而已。
- 应该警惕“distance”比较大或比较小的对象，总之和其它同类型对象的distance不一样就意味着可能有问题；
- 尽量不要用匿名函数，函数有名字会让分析更容易，其实更推荐的是使用OOP，这样会最容易定位需要追踪的变量，毕竟都是构造器创建出来的嘛；
- 闭包（匿名函数，定时器等）创建的上下文引用很容易造成不易察觉的内存泄漏；
- console的相关函数(log, error等)在实际分析中发现其引用的变量无法释放，可以参考[#1741](https://github.com/nodejs/node/issues/1741)，所以你可以在测试代码中替换掉console的相关函数（这样你就不需要改动被测代码逻辑了）；
- 对象上的事件监听器的闭包最容易造成泄漏，即便是使用once，也可能一次都没有触发而导致该回调函数无限期引用数据。

ok，一大堆姿势足够你花很久时间阅读了。不过并不是说你看了这些内容，就可以轻松战胜困难了。还有一个环节我们没有讨论：**若你的项目足够复杂（大），那要怎么搭建项目的测试环境呢？**

这里我认为，大概需要按照下面的步骤来做：

1. 将完整的项目拆解成独立的不同块，并为每个拆解后的小模块写测试代码
2. 针对定时器相关的逻辑，最好改成手动触发，或利用测试库（sinonjs）模拟时间片段
3. 初期可以先尽可能排除依赖的第三方库，最后酌情去测试它们（如果你怀疑是它们的问题的话）
4. 低级别异常伪造（例如socket，file等）要靠伪造对应方法（不推荐使用sinonjs.stubs，因为它保存每次调用时的参数数据，影响你观察，不妨试试[mockery](https://www.npmjs.com/package/mockery)）
5. 最终还是有必要放在线上环境实测一段时间来观测问题是否真的修复了

我们主要来说一下第5条，其意味着你要在线上环境想办法导出快照到本地来分析。下面来看看怎么做：

首先，你给线上环境中安装`v8-profiler`库，它用来提供创建快照的功能。

然后，看一下下面的这段样板代码，其意义在于在你的项目中加载`v8-profiler`库，并提供一个对外指令用来通知它创建快照文件。

```javascript
var fs = require('fs');
var profiler = require('v8-profiler');

// ---------------
// 测试目标
function LeakingClass(){}
var leaks = [];
setInterval(function(){
	for(var i = 0; i < 100; i++){
		leaks.push(new LeakingClass);
	}

	console.error('Leaks: %d', leaks.length);
}, 1000);
// ---------------

// 指令服务
var koa = require('koa');
var route = require('koa-route');
var service = koa();
var snapshotNum = 1;	// 用于为生成的快照进行编号
service.use(route.get('/snapshot', function *(){
	var response = this;
	var snapshot = profiler.takeSnapshot();
	snapshot.export(function(error, result) {
  	fs.writeFileSync((snapshotNum++) + '.heapsnapshot', result);
  	snapshot.delete();

		response.body = 'done';
	});
}));

service.listen(2333, '127.0.0.1');	// 推荐绑定内网ip，不要允许外网访问该服务

```

每次请求`http://127.0.0.1:2333/snapshot`，你都会在项目根目录生成一个快照文件，然后把它下载到本地磁盘就可以在chrome里随时进行分析了。

### 总结

在实际排查过程中，发现最难测试的还是依赖的第三方库的泄漏问题。毕竟你无法理解它们的实现。但不可能所有逻辑都自己来完成，所以面对各种各样的第三方类库，还是建议选择尽可能权威的，主流的。剩下那些很小的功能模块，就只能花时间研读其实现代码了。

如果你的业务采用了生产消费者模式，**你的测试脚本一定要保证任务的生产和消费的速率保持同速率（或者干脆确保消费者处理完一批次的任务耗时一定要小于批次创建的间隔时间）**，不然由于任务得不到处理，必然会产生任务积累，看起来就好像有内存泄漏一样，但其实这种情况其实是合理的，只是说明你的消费者太少了而已。

另外，一定要最大频度的，尽可能长时间的运行测试代码，才能明显的暴露出问题，例如：

```javascript
setInterval(function memoryleakBlock(){
	// 待测试的代码块
}, 100);
```

注意在上面`memoryleakBlock`中避免引用全局变量哟，这样你运行一夜，第二天上班来看结果（如果它还跑着的话）。

![](http://pic.yupoo.com/kazaff/G7D9yicX/d6nGL.png)

title: casperjs+mocha+chai搭建E2E测试环境
date: 2016-08-24 09:37:12
tags:
- 端到端测试
- casperjs
- mocha
- chai
categories: 前端
---

紧接着上一篇[《小团队玩不转的测试》](http://blog.kazaff.me/2016/08/18/%E5%B0%8F%E5%9B%A2%E9%98%9F%E7%8E%A9%E4%B8%8D%E8%BD%AC%E7%9A%84%E6%B5%8B%E8%AF%95/)文章，我们继续来看看如何搭建一个端到端测试环境。

废话不多说，直接开始动手，省的你说没干货:-)
<!--more-->
## mocha-casperjs

前一篇文章分别介绍过每个用到的插件，而此刻提到的这个组合，也是有前辈整合好了能开箱即用的：[mocha-casperjs](https://github.com/nathanboktae/mocha-casperjs)。

这里需要注意的是，根据官方提供的安装方法，推荐所有使用到的组件都使用`-g`全局安装，这样方便日后在任何项目中跑测试，而且如果有的采用全局安装有的采用非全局的话是跑不起来的，这一点官方也有说明：

> Note that mocha-casperjs has peer dependencies on casper and mocha, and will be installed ajacent to where you are installing mocha-casperjs (e.g. if you install mocha-casperjs globally, you'll have mocha and casperjs also installed globally).

如果你和我一样使用的是window开发环境，那你可能还需要自己手动安装`phantomjs`，并将其添加到系统环境变量中，否则会被`casperjs`提示缺少依赖。

还有就是，win自带的终端对中文编码支持不够好，所以我推荐使用`Git Bash`，前提是你机器上安装了git环境，其实就是一个`MINGW`环境。这样就能方便的使用大部分shell命令了。

对了，如果你win环境下没有python也不行，所以安装一个吧，并且将其加到系统环境变量中，一切就绪就可以运行官方提供的测试代码了：

> mocha-casperjs

在你的测试项目路径下执行上面的这个命令，会自动查找test或tests文件夹下所有的测试脚本并自动执行，最终的测试报告默认会直接打印在终端中~


## casper-chai

看了官方提供的测试例子，是不是蒙圈了？我对比了一下chai的文档，其实大多数chai提供的api都没有用到，毕竟都是针对单元测试的。而这个组件将casper的一些测试api用chai的方式提供给我们，感觉写起来更舒服一些。

所以推荐你先来看一下[官方手册](https://github.com/brianmhunt/casper-chai/blob/master/docs/casper-chai.md)，看看都有哪些断言api可以用来写测试项。

除此之外，你还应该看一下casperjs的[api文档](http://docs.casperjs.org/en/latest/modules/index.html)，这样才能玩得转端到端测试的各种场景。

## 测试用例

接下来的内容我们就来尝试对一些有代表性的页面编写端到端测试用例。下面的测试项目中，`json`文件用来模拟接口响应，这里只是为了演示方便，正常情况下端到端测试应该是在公司内部测试服务器上进行的，要求测试服务器上的项目版本必须和线上完全一致，包括初始化数据在内。

### 表单页面

##### 登录表单

该页面的特点是：

- 表单验证
- 登录后跳转
- 会话数据


##### 复杂表单

该页面的特点是：

- ajax输入校验
- 头像上传
- 复杂的输入类型


### 列表页面

该类型页面的特点是：

- 弹窗表单
- 翻页
- 自动完成、日历等特殊功能输入框

测试链接如下：

>　http://localhost/E2ETestDemo/pages/list/list.html?page=1

注意，这里之所以要提一下访问链接格式，是因为现有前端逻辑需要依赖地址栏中的`?page=1`参数。这一点正是我想说的，我们在写测试项时，必须要完全理解代码的含义和场景所需要的输入输出数据结构。有时候可能是因为我们模拟的测试环境丢失了一些数据导致测试不通过，而在线上产品中并没有重现该bug，这是最头疼，也是最无奈的。


### 拖拽功能页面

该类型页面的特点是：

- 可拖拽区块
- 拖拽区限制

---

上面的各个被测场景我已经放在[github](https://github.com/kazaff/E2ETestDemo)了，大家可以部署在自己本机的web环境里看一下页面的模样。对应页面的测试用例放在`tests`中，端到端测试用例脚本推荐以`e2e.js`结尾，这样未来当你还有单元测试等其他类型的测试脚本时容易区分。同时建议每个测试脚本对应一个页面，这样日后对应页面产生修改后也可以快速定位需要同步修改的测试脚本。

### 截图

![](http://pic.yupoo.com/kazaff/FOXhai4h/QcXZm.png)

## 坑

### 页面的逻辑js没有被执行

这个坑是真的坑爹，坑的我差点崩溃！从表象上来看，你会发现你根据官方提供的接口无论如何都无法触发目标页面的js执行，拿我们上面那个登录页面来说，你会发现不论是使用`sendKeys`，还是`evaluate`里使用`__utils__`都无济于事！你可以通过`capture`生成截屏来验证！

哥哥我当时就蒙圈了，说好的幸福呢？这尼玛让我如何完成对领导的承诺？！就在我万念俱灰准备离职之际，突然看到了一个歪果仁的解答：

> I find that this happens when casperJS is too fast for the browser and a casper.wait() will solve the issue.

突然就醍醐灌顶了，一下子就好像打通了任督二脉！回味一下确实如此，内置的浏览器加载html所依赖的资源需要时间，而我们的测试脚本在html加载完毕后就会立即执行，所以就给我们感觉仿佛世界观崩塌了！

解决方案很简单，只需要让测试脚本执行前`wait`几秒钟即可。


### 页面默认是手机尺寸

如果你的被测网站是响应式的，你会发现`capture`生成截屏默认是适配的手机样式。我起初以为和`UserAgent`有关：

```
casper.userAgent('Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.106 Safari/537.36');
```
事实证明没鸟用，后来我又试图修改截图的尺寸，结果也是毫无意义。后来才发现官方提供了`viewport`接口，而且也明确表明，默认`phantomjs`采用的是`400 X 300`的设置。

坑爹！

### 提示"0 passing"

不知道为何，当我们的测试脚本存在语法错误时，执行测试命令后就会看到`0 passing`提示，这对我们调试造成了非常大的困难，尤其是当测试脚本行数非常大时！


### 灵异问题

在实际使用时，由于ui所使用的一些js组件，或其他一些不明真相的原因，会导致你写不出正确的测试用例，我在demo中有注解，仔细看的话会发现不少小问题，虽然都最终解决了，但不知道为何如此，不甘心啊~

## 心得总结

经历了上述的几种常见场景，我觉得基本上只要有足够精力和时间，大部分项目都可以完成端到端测试。不过再次提醒，维护测试脚本也是要花时间和精力的，建议针对项目中相对稳定的界面和功能进行端到端测试，而处于待定阶段的页面还是人肉先行吧。

测试用例之间最好不要有顺序上的依赖，除非你能保证日后该流程会被整体执行，否则当你想单独执行某个子测试用例时会非常尴尬！还有就是，你的项目一定会使用很多第三方成熟的ui组件，而针对这些组件你不需要写太多测试用例，你的测试关注点应该更多的集中在自己实现的逻辑中。

推荐优先使用`thenClick`而非`click`接口，因为如果你连续调用（和两次调用代码位置无关），很可能会造成casperjs执行太快而导致早于你的业务逻辑js代码的执行，截屏上会让你感觉你的click调用未生效，但其实是太快了~~~

最终，想要写出好的测试用例，排除测试代码自身质量因素外，还需要编写者对被测页面有足够的了解，不仅仅是页面上的显示元素，还包括该页面的加载细节（最头疼的**网络延迟**问题，不过可以使用casper.waitXXX相关api来优雅的解决这个问题），初始化细节等等。这样看来，其实最合适写测试用例的还是这个页面的实现者本尊，但同时这就要求你必须同时掌握测试相关的背景知识。现在来看，“测试先行”就显得非常合理了，根据设计书，开发人员完全可以先完成测试项（这里指的并非测试脚本，因为端到端测试不同于单元测试，它重度依赖UI，所以提前写端到端测试脚本不太现实，我这里指的测试项指的是**文字描述的测试条目**，有助于开发人员更好的分析页面逻辑），只不过这种做法的效率目前没办法评估，还要扭转开发人员现有的思维模式，是一个不小的挑战！

这里我们也回避了一个较为重要的问题：到底一个页面需要多少个测试项才合理？前一篇文章和其参考文献中也有对该问题的相关描述，在考虑到诸多因素后如何权衡才能取得到一个平衡点，这是一门科学，等我日后有感悟了再回来表。

这篇比上一篇干货要多不少吧，希望大家满意:-)


## 参考手册

[http://docs.casperjs.org/en/latest/modules/index.html](http://docs.casperjs.org/en/latest/modules/index.html)

[https://github.com/brianmhunt/casper-chai/blob/master/docs/casper-chai.md](https://github.com/brianmhunt/casper-chai/blob/master/docs/casper-chai.md)

[http://chaijs.com/api/bdd/](http://chaijs.com/api/bdd/)

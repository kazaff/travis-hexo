title: git hook帮你维护前端代码规范
date: 2016-08-17 09:37:12
tags:
- FECS
- 代码格式
- git
- hook
- nodejs
- pre-commit
- review
categories: 前端
---

只要不是一个人在战斗，你都一定会碰到很多工程问题。我们今天来说的，就是代码格式问题。这不是个什么有意思的话题，这个话题讲的就是条条框框，就是枯燥，就是没意思。但是，如果你的团队缺少代码格式规范的话，当你review组员的代码时，你就会感觉在吃屎，没错，不夸张！

我不怀疑团队组员的积极性，因为条条框框本来就不是程序员的调调，而且人类和机器的最大差别就是遵守规范的程度。所以，你不能要求你口头上说代码格式要怎样怎么，所有同事就会立刻写出符合要求的完美代码。

我也从来没活在真空中，项目的期限和需求变动累加在一起，但凡是个活人，都尼玛会被折腾的精疲力尽，谁有能包票自己无时无刻都会遵守代码规范？反正我做不到！既然我自己都做不到，就更不应该要求别人。

不过，如果可以让机器来帮我们检查代码格式问题，那无疑是一个不错的注意。这里的哲学思想是：习惯成自然！你可以一开始记不住规范，但随着你在项目组内待的时间，慢慢你就会养成习惯，最终所有人的代码都会符合既定规范。

ok，扯了这么多废话，我们开始搭建代码检测环境吧。


## git hook

用git的人，肯定知道这个东西，但可能没怎么用过，例如我。这可不是个小玩意儿，它厉害着呢！大家可以从[这里](https://segmentfault.com/a/1190000000356485)大概了解一下相关的知识。而我们这次主要目标是：pre-commit。

pre-commit钩子是在git本地提交前会执行的一个回调，通过我们自己定义的脚本可以达到前面我们提到的代码格式检查的目的，但凡不符合规范的提交，一律驳回。


## nodejs

如果你shell和我一样弱鸡，那你也可以和我一样考虑使用nodejs作为脚本引擎。庆幸的是不少前辈已经为我们铺平道路，老省心了：

- [husky](https://github.com/typicode/husky)
- [node-hooks](https://github.com/mcwhittemore/node-hooks)

今天我要介绍的，是我们国内百度提供的一套解决方案：[fecs](http://fecs.baidu.com/)，之前从没说过，今天也是在搜索相关主题的时候发现的好东西。我们主要使用它的插件：[fecs-git-hooks](https://github.com/cxtom/fecs-git-hooks)，有了它，我们只需要一行命令就可以创建好所需的pre-commit：
```
//项目根目录下执行
npm i fecs-git-hooks
```
执行完后，你会在项目根目录下的`.git/hooks`文件夹下看到配置好的pre-commit，一切就已经完成了。你可以创建一个测试项目来测试一下，现在不满足fecs定义好的规范的提交都会被驳回。

那么，到底有哪些规范会被检查呢？

- [html](https://github.com/ecomfe/spec/blob/master/html-style-guide.md)
- [js](https://github.com/ecomfe/spec/blob/master/javascript-style-guide.md)
- [css](https://github.com/ecomfe/spec/blob/master/css-style-guide.md)

## 规则自定义

每个公司甚至每个团队都可能有自己的代码风格规范，这个不能强求，毕竟没有最好的规范，只有最合适的。

那如果fecs默认的规范和我们的不匹配呢？也很简单，fecs提供了可配置文件来让我们设置是否开启某一项检查。

那我的场景来说，我修复了所有的ERROR级别的错误，但是WARN级别的我个人觉得就无所谓了，但机器不这么认为，所以只要还存在一条不满足规范的代码就无法提交！

根据[官方文档](https://github.com/ecomfe/fecs/wiki/Configuration)所述，我们可以在项目根目录添加一个`.fecsrc`配置文件。但官方并没有解释清楚配置项要怎么写，故意的吗？

其实并不难，只需要根据git提交时报错的日志，每一行的最后一个`()`内就是这个配置项的名字，只需要分别出它到底是html，css还是js的配置项即可。

然后在`.fecsrc`中这么写：
```
{
  "htmlcs": {
		"max-len": false,
		"asset-type": false,
		"style-disabled": false,
		"lowercase-id-with-hyphen": false,
		"lowercase-class-with-hyphen": false,
		"indent-char": false,
		"self-close": false,
		"img-width-height": false,
		"attr-lowercase": false,
		"img-src": false,
		"label-for-input": false,
		"bool-attribute-value": false,
		"attr-no-duplication": false
	},
	"csshint": {
		"disallow-important": false
	},
	"eslint": {
    "env": {
        "es6": true
    },
    "rules": {
        "no-console": 0
    }
  }
}
```

你也可以查看都有哪些可配置的项目：

- [csshint](https://github.com/ecomfe/fecs/blob/master/lib/css/csshint.yml)
- [htmlcs](https://github.com/ecomfe/fecs/blob/master/lib/html/htmlcs.yml)
- [eslint](https://github.com/ecomfe/fecs/blob/master/lib/js/eslint.yml)


## 不足

这种方案相当于是本地化git hooks，要求项目组员必须在各自的开发环境下统一配置一致的fecs环境，日后如果有调整，也需要所有人同步更新。

git hook其实也有服务器端的，日后我们再学习一下~

## 注意

一旦在组内推广，初期肯定会碰到大量的代码格式报错，所以应该选择一个合适的时间段进行推广，不然会带来额外的繁重工作影响团队气氛，甚至是项目进度。

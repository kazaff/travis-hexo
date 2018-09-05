title: GraphQL的前提
date: 2018-9-5 10:37:12
tags:
- GraphQL
- DDD
- pagination

categories:
- nodejs
---

很有可能，这会是最后一篇关于这个主题的文章了，呃，这算是真的从入门到放弃么？啊哈哈哈哈哈~

![](http://pic.yupoo.com/kazaff/HCj1vqRd/medish.jpg)

东西是好东西，思想是好思想，不过感觉用在项目中的成本和风险有些不好把握，怂了怂了，这次是真怂了。

即便如此，我们还是需要讨论一下几个相关问题：

- 如何设计schema（edges, nodes）
- 1+N问题解决原理(batch, cache)
- 学习曲线和资料

### schema

一开始接触Graphql（后称：GQL），一定是学习关于schema相关的知识，我想对于前端的同学这方面的东西也是最有新鲜感的。
资料上给的例子，都非常的简化，不管你读起来感觉多么的实战，但不得不承认依然十分的简单。
但实际项目中，你会遇到复杂得多的场景，并不是说GQL无法支持复杂场景，而是想说场景复杂后，会要求开发者把控schema的能力要十分的强。

这让我不禁想到了领域模型的概念，不过话说回来，即便是现在早已普世的RESTful，真的用在项目里，也依然要搞清楚它本质的概念（面向资源）。
殊途同归，最终都是要先消化业务，抽象模型，再来谈最基本的语法。
只不过，GQL的schema中为模型里的很多对应概念都提供了对应的元素：edges, nodes。

不知道你有深究过GQL名字的含义么？其中`graph`到底表达了什么含义？

![](http://pic.yupoo.com/kazaff/HCpPoIyY/medish.jpg)

了解`Neo4j`这种基于图理论的数据库，就很容易找到对应~所以，我们的GQL也基于图的概念哟~上图中每个圆点就代表了nodes，每个灰色的线条就代表了edges。
延续这种思路，套用在真实项目中，就需要你有深厚的领域建模功力，甚至也要了解服务拆分和部署相关的知识，没错，要求你是架构师啦。
呃，至少是业务专家吧。
当然，如果你并不完全遵守规则也可以（就好像大多数生成自己是Rest的接口，其实仅仅是http+json的rpc而已）~~

### 1+N

在之前的一篇[翻译文章](https://blog.kazaff.me/2018/03/25/%E5%86%8D%E5%93%81GraphQL/#1-N%E6%9F%A5%E8%AF%A2%E9%97%AE%E9%A2%98)中就提到过这个问题，不过现在看来，作者撒了个谎，或者说至少他并没有提到使用dataloader必须要遵守的规则，恰恰是这两个规则会导致你方案的复杂性（尤其是对于计划让GQL代理后端服务的项目）。

如果我们仔细看过dataloader开发者提供的[视频](https://www.youtube.com/watch?v=OQTnXNCDywA&feature=youtu.be)，你会很明确的知道，要想正常的使用dataloader，你就不得不要确保：

1. 数据源（可以是db，也可能是service）返回的数据条目数必须完全匹配dataloader受理的请求个数
2. 请求的顺序和数据源响应的结果的顺序也要保持一致

这两点在你研读dataloader的源码后就会完全明白~

那官方文档的例子来说，假如你在一个时间循环中多次调用`load`方法，dataloader打包了你的请求，打包后的请求为`[ 2, 9, 6, 1 ]`，那么意味着数据源返回的数据必须如下：

```
[
  { id: 2, name: 'San Francisco' },
  { id: 9, name: 'Chicago' },
  null,
  { id: 1, name: 'New York' }
]
```
请注意条目数和顺序的一致性。

大家可以思考一下，我们要如何保证这两点呢？是否会造成已有的服务必须调整呢？（老实说，这是我最头疼的一点）。

### 学习曲线

除了学习GQL外，我的观点已经很明确，你还需要很多其他知识来辅佐，否则很难讲你会在实际项目里把GQL发挥到最大功效。
而目前为止，互联网上关于QGL的实战资料很少，我能找到的都还停留在概念讲解和语法分析上，这意味着我们这样的小团队想开箱即用基本上是不太可能的。
不过还是长期看好这个东西，详细它会成为下一代API的趋势。


### 扩展阅读

[https://blog.apollographql.com/explaining-graphql-connections-c48b7c3d6976](https://blog.apollographql.com/explaining-graphql-connections-c48b7c3d6976)
[https://github.com/facebook/dataloader#batching](https://github.com/facebook/dataloader#batching)
[https://platform.github.community/t/difference-between-using-edges-node-nodes/1883/3](https://platform.github.community/t/difference-between-using-edges-node-nodes/1883/3)
[https://medium.com/@gajus/using-dataloader-to-batch-requests-c345f4b23433](https://medium.com/@gajus/using-dataloader-to-batch-requests-c345f4b23433)

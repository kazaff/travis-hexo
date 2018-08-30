title: GraphQL运行原理
date: 2018-8-29 10:37:12
tags:
- GraphQL

categories:
- nodejs
---

最近依然在关注GraphQL这个技术，感觉它非常有前景。结合现有的业界关于网关服务的实践和总结，再融合GraphQL的思想，瞬间一个先进无比的网关服务出现在了眼前。

之前翻译过一篇[文章](https://blog.kazaff.me/2018/03/25/%E5%86%8D%E5%93%81GraphQL/)，感觉是读过的最完整的一篇深入浅出graphql的实战文章。今天打算继续消化一下文章中引用的一份文档：[Execution](https://graphql.org/learn/execution/)。这篇文档解释了GraphQL是如何根据定义的schema来完成数据的组装和聚合的，应该算是整个GraphQL架构中最核心的设计之一了，值得了解。
下面废话不多说，正文走起。

### Execution

在必要的数据校验环境后，GraphQL（译：后面简称GQL）服务端会根据每一个query的实际要求来剪裁恰如其分的数据结构以进行响应，通常来说是JSON格式。

GQL非常依赖其类型模型（type system），我们来看一个实际的例子来演示如何执行一个query。这个例子和文档中其他部分是一致的：

```
type Query {
  human(id: ID!): Human
}

type Human {
  name: String
  appearsIn: [Episode]
  starships: [Starship]
}

enum Episode {
  NEWHOPE
  EMPIRE
  JEDI
}

type Starship {
  name: String
}
```

为了了解query执行的细节，我们来看一个例子：

```
// 请求
{
  human(id: 1002) {
    name
    appearsIn
    starships {
      name
    }
  }
}

// 响应
{
  "data": {
    "human": {
      "name": "Han Solo",
      "appearsIn": [
        "NEWHOPE",
        "EMPIRE",
        "JEDI"
      ],
      "starships": [
        {
          "name": "Millenium Falcon"
        },
        {
          "name": "Imperial shuttle"
        }
      ]
    }
  }
}
```

你可以将在GQL查询中每一个字段（field）看做是前一种类型的函数或方法，它返回下一种类型。事实上，这就是GQL的工作原理。在GQL服务端，每一个类型的每一个字段背后都是靠一个被称为`resolver`的函数来支撑的（译：提供数据）。每当一个字段被要求返回，与其对应的`resolver`函数就会被执行，并产生下一个值（译：进入新一轮`resolver`执行）。

如果发现字段返回的是一个标量值，如字符串或数字，此时执行就算告一段落了。
然而如果该字段执行后返回的是一个对象值，那么执行器会继续试图获取该对象值包含的字段，一直到最终得到一个标量值为止。

#### Root fields & resolvers

每个GQL服务的最顶层类型是一个包含一切的统一API入口类型，通常我们称之为`Roottype`或`Querytype`。
在我们的例子中，`Querytype`提供了一个字段叫`human`，它接受一个参数`id`。这个字段对应的`resolver`函数通过操作数据库并构建和返回`human`对象。

```
Query: {
  human(obj, args, context, info) {
    return context.db.loadHumanByID(args.id).then(
      userData => new Human(userData)
    )
  }
}
```

这个例子是用JavaScript写的，但GQL服务端可以用[多种语言](https://graphql.org/code/)来实现。`resolver`函数接受4个参数：

- obj: 前一个对象（译：触发该`resolver`的字段所在的对象），这个参数对最顶层字段没有意义（译：当然啊，不然嘞~）
- args: 来自GQL query
- context: 提供所有`resolver`依赖的资源，例如数据库连接对象，当前登录的用户信息等
- info: 包含与当前查询相关的特定于字段的信息的值，以及模式细节，可以参考[graphqlobjecttype](https://graphql.org/graphql-js/type/#graphqlobjecttype)

#### Asynchronous resolvers

我们来近距离看一下这个`resolver`函数的细节：

```
human(obj, args, context, info) {
  return context.db.loadHumanByID(args.id).then(
    userData => new Human(userData)
  )
}
```

`context`参数中包含了数据库连接对象，用于数据查询来得到GQL query中要求的`id`数据。查询数据库是一个异步调用，所以返回一个`promise`。`promise`是javascript中的异步调用概念，不过其他很多语言也有对应的实现方式，通常被称之为`futures`，`tasks`或者`Deferred`。当数据库操作返回后，我们就可以构建和返回一个新的`Human`对象啦。

需要注意的是尽管`resolver`函数返回的是`promise`，但GQL query并不是异步的，它会期望`human`携带了所有要求返回的数据。在执行过程中，GQL会一直等到`Promise`,`Futures`或`Tasks`完结后才会继续并最大化保持并发度（译：这一点很重要）。

#### Trivial resolvers

现在我们已经得到了一个`Human`对象，接下来GQL执行器将继续处理其下的字段。

```
Human: {
  name(obj, args, context, info) {
    return obj.name
  }
}
```

GQL服务依靠类型系统来决定如何继续执行下去。甚至是在`human`返回任何结果之前，GQL就可以根据类型系统要求提供的`human`类型声明得到下一步应该处理的字段有哪些。

在这个例子里`name`字段的处理是非常简单明了的。传入`name resolver`函数的`obj`参数就是前一步返回的那个`new Human`对象。例子中我们期望得到的`human`对象包含`name`字段，已经如愿以偿。

事实上，很多GQL类库都不需要你提供这种简单的`resolver`，它们会默认自动从`obj`中读取并返回对应名字的属性（译：默认解析器规则）。

#### Scalar coercion

当`name`字段被处理过后，`appearsIn`和`starships`字段会被同时处理。`appearsIn`字段也有一个`trivial resolver`，我们来仔细看一下：

```
Human: {
  appearsIn(obj) {
    return obj.appearsIn // returns [ 4, 5, 6 ]
  }
}
```

注意，我们的类型系统声明`appearsIn`将返回一个枚举类型，然而这个函数却返回的是number数组！实际上，如果我们查看结果，我们将看到相应的Enum值被归还。发生了什么？

这就是一个`Scalar coercion`的例子。类型系统知道应该返回什么，并将解析器返回的数据转换成了API声明要求的类型。在这个例子中，在服务的其他位置应该存在一个枚举类型的定义来标识`4,5,6`对应的枚举值。

#### List resolvers

通过`appearsIn`，我们已经看到了当一个字段需要一个返回多条数据时的细节。它返回了一个枚举值数组，然后类型系统将每个值转换成了对应的枚举值。那`starships`字段解析的细节有是什么呢？

```
Human: {
  starships(obj, args, context, info) {
    return obj.starshipIDs.map(
      id => context.db.loadStarshipByID(id).then(
        shipData => new Starship(shipData)
      )
    )
  }
}
```

这个字段的`resolver`不只是返回一个`promise`，它返回了一个`promise`数组（译：屌不屌）。`human`对象拥有一个`starships`的`id`数组，但我们需要加载所有这些`id`关联的`starship`对象。

GQL会等待所有的`promise`并发的完成后才会继续，当所有`starship`对象都得到后，GQL会继续并发的尝试获取这些对象的`name`字段。

#### Producing the result

当所有字段都处理完毕后，结果值构建成一个从叶子节点到根节点全链路的键值对映射，键为字段名，值为`resolver`返回的结果，最终按照请求的结构返回给客户端对应的数据结构（JSON结构）。

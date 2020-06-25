title: mapping set to strict, dynamic introduction of [_class] within [_doc] is not allowed
date: 2020-06-26 09:37:12
tags:
- elascticsearch7+
- spring-data-elasticsearch
categories: spring
---

这个异常非常的“有趣”，因为我GG了一圈，竟然一个相关的结果都没有。。。一开始以为自己碰到疑难杂症了，深呼吸一口打算去翻源码了。
结果“非常不小心”被我在官方文档上以“_class”作为关键字检索到了相关的内容，好尴尬啊~

官方在[Mapping Rules](https://docs.spring.io/spring-data/elasticsearch/docs/4.0.1.RELEASE/reference/html/#elasticsearch.mapping.meta-model.rules)这一章节中，提到了`Type Hints`的概念：

> Mapping uses type hints embedded in the document sent to the server to allow generic type mapping. Those type hints are represented as _class attributes within the document and are written for each aggregate root.

在我这个新手看来，应该是Spring-Data-Elasticsearch库对将Elasticsearch返回的结果自动封装成POJO时需要的一个标识。很自然也很直观，对吧~

恶心就恶心在，如果你和我一样是初次使用，并且在Elasticsearch中将Index的Mapping设置成严格的静态类型，就会导致本文标题的这个异常。然后你就会开始发蒙，`_class`这个属性是什么时候产生的。。。

解决这个异常的方法很简单，在我们的Mapping中增加_class属性的定义即可：

```JSON
...
"mappings": {
        "dynamic": "strict",
        "properties": {
          "_class" : {
              "type" : "keyword"
            },
....
```

奇怪的是，全世界的开发者好像都没有碰到这个问题。。。是我确实太菜了吧可能。。
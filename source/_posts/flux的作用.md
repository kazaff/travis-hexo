title: flux的作用
date: 2017-04-30 09:37:12
tags:
- redux
- vuex
- 扁平化
- 不可变
- 数据表概念

categories:
- 前端
---

很久很久之前，应该是fb刚提出flux的初期，我就花时间在学习如何使用它了。不过断断续续的工作经验，很多东西的记忆已经模糊了，亦或者是从来没有悟出什么深层次的东西，导致前段时间另外一个团队的同事在聊到vuex时，我脑子里除了文档中教条式的使用方法，一片空白。问题大了啊~~

今天无意间逛到了一篇文章，其中扯到了redux的意义，使我受益匪浅。为了避免再次忘记，我赶紧记录下了这篇文章。

关于flux、redux和vuex，我们不谈它们之间具体的用法，这篇文章的关注点在于这种思想的意义（比较空，但很重要）。
对这种思想的一种粗浅理解是，**通过增加这一中间层，让整个项目对数据的操作（CURD）由原先的混乱与随意改善为统一和明确**，做大的好处应该就是增加了代码维护性。

而仅仅这一种维度的解释，实在无法说服别人来引入这个玩意儿，毕竟如果仅仅想做到这样，自己封装一个数据操作类即可啊。

另外一个模糊的认识，是来自于早期官方的一个示例，通过这个中间层，我们可以很轻易的就让应用实现“后退”功能。不过这依然可以简单的将历史版本数据保存在localstorage中而轻易实现。那么，flux还有什么深意么？

直到看到了[这篇文章](http://redux.js.org/docs/recipes/reducers/NormalizingStateShape.html#show-last-Point)，我们来总结一下文章中提到的几个要点：

- 如果数据出现在多个不同的地方，势必存在重复和嵌套层次，这会让更新数据的逻辑变得复杂
- 过深的嵌套和严重的重复会加剧操作数据的复杂性，降低处理性能
- 伴随着引入不可变数据类型，这种嵌套和重复还会丧失不可变数据类型的优势，进一步降低渲染性能

这样，flux的作用就基本上已经变得不可或缺，不再是可有可无了。

那么，具体flux提出了哪些概念来解决问题呢？

文章中提到了“data table”，其实就是数据库概念，由于种种原因，数据库往往都被我们设计成尽量扁平的（关系型数据库只有行列概念），除了范式外，上面提到的要点也都自然而言的考虑在内了。

按照flux的最佳实践，我们需要：

- 为每一个概念实体都创建独立的存储单元
- 实体之间依赖id来进行关联
- 多对多关系也抽象成一种概念实体（和关系型数据库处理多对多关系完全一致）

其实完全是借助数据库思想来解决问题的，只是这种想法在当时还是很超前的。

### 实例

嵌套结构：

```javascript

const blogPosts = [
    {
        id : "post1",
        author : {username : "user1", name : "User 1"},
        body : "......",
        comments : [
            {
                id : "comment1",
                author : {username : "user2", name : "User 2"},
                comment : ".....",
            },
            {
                id : "comment2",
                author : {username : "user3", name : "User 3"},
                comment : ".....",
            }
        ]    
    },
    {
        id : "post2",
        author : {username : "user2", name : "User 2"},
        body : "......",
        comments : [
            {
                id : "comment3",
                author : {username : "user3", name : "User 3"},
                comment : ".....",
            },
            {
                id : "comment4",
                author : {username : "user1", name : "User 1"},
                comment : ".....",
            },
            {
                id : "comment5",
                author : {username : "user3", name : "User 3"},
                comment : ".....",
            }
        ]    
    }
    // and repeat many times
]

```

扁平化结构：

```javascript

{
    posts : {
        byId : {
            "post1" : {
                id : "post1",
                author : "user1",
                body : "......",
                comments : ["comment1", "comment2"]    
            },
            "post2" : {
                id : "post2",
                author : "user2",
                body : "......",
                comments : ["comment3", "comment4", "comment5"]    
            }
        }
        allIds : ["post1", "post2"]
    },
    comments : {
        byId : {
            "comment1" : {
                id : "comment1",
                author : "user2",
                comment : ".....",
            },
            "comment2" : {
                id : "comment2",
                author : "user3",
                comment : ".....",
            },
            "comment3" : {
                id : "comment3",
                author : "user3",
                comment : ".....",
            },
            "comment4" : {
                id : "comment4",
                author : "user1",
                comment : ".....",
            },
            "comment5" : {
                id : "comment5",
                author : "user3",
                comment : ".....",
            },
        },
        allIds : ["comment1", "comment2", "comment3", "commment4", "comment5"]
    },
    users : {
        byId : {
            "user1" : {
                username : "user1",
                name : "User 1",
            }
            "user2" : {
                username : "user2",
                name : "User 2",
            }
            "user3" : {
                username : "user3",
                name : "User 3",
            }
        },
        allIds : ["user1", "user2", "user3"]
    }
}

```


实体关系处理：

```javascript

{
    entities: {
        authors : { byId : {}, allIds : [] },
        books : { byId : {}, allIds : [] },
        authorBook : {
            byId : {
                1 : {
                    id : 1,
                    authorId : 5,
                    bookId : 22
                },
                2 : {
                    id : 2,
                    authorId : 5,
                    bookId : 15,
                }
                3 : {
                    id : 3,
                    authorId : 42,
                    bookId : 12
                }
            },
            allIds : [1, 2, 3]

        }
    }
}

```

### 工具

思想有了，工具也不能落后，[normalizr](https://github.com/paularmstrong/normalizr)就是来帮我们来快速实现不同结构之间的转化的。

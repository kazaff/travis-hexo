title: 关于mysql的insert for update在多条插入记录下的使用
date: 2019-03-18 09:37:12
tags:
- mysql
- insert for update
categories: 数据库
---

今天我们来讨论一个比较大众的问题：关于当在mysql中insert记录时违反了表的唯一键后要怎么办？
其实一般而言，越来越多的系统在数据一致性和有效性问题上，都放在应用逻辑层来维护合法性。
到底哪种方式好，在这篇文章中不做评论，我个人觉得尽量都在代码中体现才是王道。

回到问题，一般遇到这种情况，第一反应是使用`insert for update`语法，例如：

```
INSERT INTO t1 (a,b,c) VALUES (1,2,3)
  ON DUPLICATE KEY UPDATE c=c+1;
```

这句sql很直观，基本上算是语义化了。但如果是批量insert呢，我们希望如果遇到冲突，update块使用对应插入记录的值来更新，
而不是固定值或db中当前值，该怎么办呢？

```
INSERT INTO t1 (a,b,c) VALUES (1,2,3),(4,5,6)
  ON DUPLICATE KEY UPDATE c=VALUES(c), b=VALUES(b);
```
对，没错，mysql提供的内置函数`VALUES()`就是针对这种需求的。


#### 参考文献

[https://dev.mysql.com/doc/refman/8.0/en/insert-on-duplicate.html](https://dev.mysql.com/doc/refman/8.0/en/insert-on-duplicate.html)

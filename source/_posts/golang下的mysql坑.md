title: golang下的mysql坑
date: 2018-3-17 10:37:12
tags:
- mysql
- timeout
- query-placeholder

categories:
- golang
---

初学golang，感觉一切都很新鲜。直入正题，简单分享一下项目中碰到的几个小坑。

### 关于sql语句占位符

按照大量文档上的样板代码，一般我们查询mysql，会这么写：

```go
name := "kz"
row := db.QueryRow("SELECT * FROM users WHERE name=?", name)
```

需要注意的是，`name`的类型为`string`，所以golang的mysql库在替换占位符`?`时，会自动增加引号在值两边（防止注入）。所以你是不需要在sql语句中自己增加引号的。详情可以看源码`github.com\go-sql-driver\mysql\connection.go`的大概`287`行：

```go
...
case string:
			buf = append(buf, '\'')
			if mc.status&statusNoBackslashEscapes == 0 {
				buf = escapeStringBackslash(buf, v)
			} else {
				buf = escapeStringQuotes(buf, v)
			}
			buf = append(buf, '\'')
...
```

但是，如果你想这么查询mysql，就有点悲剧了：`"SELECT * FROM users WHERE id NOT IN(?)`，我们希望用一个拼接的字符串来替换占位符，例如“1,2,3”，由于我们拼接的字符串是动态的，此时只能放弃使用占位符了，老老实实的字符串拼接sql吧。。


### sql执行的timeout问题

在我的场景里，是存在写锁（`SELECT FOR UPDATE`）的，实现的过程中发现要想让查询能快速从阻塞中退出，首先想到的是修改session的`innodb_lock_wait_timeout`外，但由于golang的mysql库是自带连接池的，这就意味着我们要记得修改回默认值，这件事太麻烦了。其次，打算自己来实现一个timeout channel来辅助恢复阻塞，但这样的话需要了解一下mysql库内部的机制，因为我担心虽然我靠自己的通道拿到了控制权，但数据库链接依然阻塞，这样后续的针对这个链接的操作依然无法执行（未求证）。

在寻找答案的过程中，发现了[Context](https://segmentfault.com/a/1190000006744213)概念，而mysql库目前也支持了利用这个机制来实现timeout，代码如：
```go
newCtx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()
row := tx.QueryRowContext(newCtx,"SELECT * FROM users WHERE name=? FOR UPDATE", "kz")
if err != nil {
  // 锁申请超时，放弃本次执行，稍后重新尝试
}
```

不过，在实际使用的时候，由于库设计者的哲学，该超时的链接还是会放回连接池中，这会导致下次从池中获取可用连接时，可能会取出这个已经超时的链接，不过mysql库会自从重新创建一个新的可用链接，并在终端打印出一条警示：`driver: bad connection`，源码如下：
```go
func (db *DB) BeginTx(ctx context.Context, opts *TxOptions) (*Tx, error) {
	var tx *Tx
	var err error
	for i := 0; i < maxBadConnRetries; i++ {
		tx, err = db.begin(ctx, opts, cachedOrNewConn)
		if err != driver.ErrBadConn {
			break
		}
	}
	if err == driver.ErrBadConn {
		return db.begin(ctx, opts, alwaysNewConn)
	}
	return tx, err
}
```
可以看到，该警示无害，库总会无条件申请一个新的可用链接~


今天的分享就到这里，88

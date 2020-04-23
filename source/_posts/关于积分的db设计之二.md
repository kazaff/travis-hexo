title: 关于积分的db设计之二
date: 2020-4-23 10:37:12
tags:
- 积分设计
- 有效期
categories: 数据库
---

继续积分这个话题，我们接着之前的[那篇文章](https://blog.kazaff.me/2020/04/23/%E5%85%B3%E4%BA%8E%E7%A7%AF%E5%88%86%E7%9A%84db%E8%AE%BE%E8%AE%A1%E4%B9%8B%E4%B8%80/)往下说。上一篇给了一个db的设计方案，也简单讨论了一下它的精妙与不足。
那么，是否还可以有更好一些的db设计呢？

别着急，我们还是先来说业务。
其实除了上一篇文章提到的那些规则外，我们的电商系统里，还有另外一个额外的小规则，由于比较另类，所以我放在这里才说，因为它也会对上面的那个db设计造成一定的问题。

我们的电商系统，在客户结算时并不提供一个输入框来给客户输入打算用使用的积分数量，而是只提供了一个开关选项，如果客户选择使用积分（默认），则意味着会**尽可能使用积分来完成结算**。这一点应该算是比较另类了，这意味着，合理的情况下会尽可能避免用户的积分到期，除非他很久才回归。也意味着客户只要选择使用积分，就可能会关联一大批的积分记录（请结合之前的points表来理解这句话）。这也是我对之前的db设计直觉上总感到不完美的切入点。

我假设的是，客户每次下单都积累积分（假设10次），之后他想要使用积分了，就可能会影响10条记录，并且在这10条记录上做写锁以及扣减逻辑，未来退款还要再逆向做一次。而且，其实每次计算客户当前可用积分，`where`条件也很难很好的利用索引（`expire > NOW`很难有效的过滤掉足够多的记录）。

那么结合这些因素，我们试着改良一下前面的db设计：

```sql
CREATE TABLE points_status(
    id INT NOT NULL AUTO_INCREMENT  COMMENT '主键' ,
    user_id INT NOT NULL   COMMENT '用户id' ,
    points INT NOT NULL  DEFAULT 0 COMMENT '可用积分总值' ,
    points_status TEXT NOT NULL  DEFAULT [] COMMENT '积分状态明细' ,
    PRIMARY KEY (id)
)ENGINE=InnoDB CHARSET=utf8 COMMENT = '积分状态表 ';

CREATE TABLE points_logs(
    id INT NOT NULL AUTO_INCREMENT  COMMENT '主键' ,
    user_id INT NOT NULL   COMMENT '用户id' ,
    order_id INT NOT NULL  DEFAULT 0 COMMENT '订单id' ,
    type CHAR(1) NOT NULL  DEFAULT 0 COMMENT '操作类型' ,
    points INT NOT NULL   COMMENT '积分变更值' ,
    expire INT NOT NULL  DEFAULT 0 COMMENT '到期时间批次' ,
    PRIMARY KEY (id)
)ENGINE=MyISAM CHARSET=utf8  COMMENT = '积分流水表 ';

CREATE INDEX idx1 ON points_logs (user_id, type, expire, order_id);
```

这次拆分成了两张表，也比较符合习惯，一张数据聚合表（points_status），另一张数据流水表（points_logs）。

其中**积分状态表**中直接将当前用户可用的积分总值存入`points`字段，方便获取，而为了记录每一批次积分的到期时间，该表的`points_status`字段中存放的也是json结构的数据字符串：

```javascript
[{"expire": "精确到日期的时间戳", "val": "该批次的积分数值"}, ...]
```

这里注意一个细节，`expire`键中存放的是精确到日期的时间戳，这样同一天到期的多个积分会自动汇总到一起。我们姑且称为**积分批次**。而在表`points_logs`中，每一条积分的流水记录也都有相同颗粒度的到期批次字段`expire`，这样当出现退款时，可以从对应流水记录中的批次得到每个批次应该退还多少数值。

我们还是举个具体的例子吧~~依然假设目前就只有一个客户，他通过下单，已经挣到了200积分，那么在db中会保存对应的记录：

**points_status表**
| id | user_id | points | points_status |
| :--| :--     | :--    | :--           |
| 1  | 1       | 200    | [{"expire":"2020-05-01", "val":100},{"expire":"2020-06-01", "val":100}] | 

**points_logs**
| id | user_id | points      | expire    | type       | order_id |
| :--| :--     | :--         | :--       | :--        | :--      |
| 1  | 1       | +100        | 2020-05-01| earn       | 1        |
| 2  | 1       | +100        | 2020-06-01| earn       | 2        |

每次客户挣得积分，业务代码操作时只需要锁定`points_status`表中对应`user_id`的那一条数据。至于积分流水表，以插入为主。
每当业务需要更新客户的积分状态时，都可以“顺便”做一件事儿：将`points_status.points_status`字段中的过期值清理一下，清理出得过期积分也要插入到`points_logs`表中，如下面这样的记录：

**points_logs**
| id | user_id | points      | expire    | type       | order_id |
| :--| :--     | :--         | :--       | :--        | :--      |
| 3  | 1       | -100        | 1984-07-23| expire     |          |

> 注意：该记录的`type`为expire，我们可以暂定`type`字段供有三种类型值：earn(得取)，cost(消费)，expire(过期)

我们也可以在每天的凌晨执行一个定时任务，用来把系统中所有客户的过期积分都处理一遍。当然如果系统的客户数据量巨大，也可以根据客户的活跃度分批次进行处理。即便是不处理，在客户决定使用积分的时候，也优先从`points_status.points_status`字段中动态计算积分的有效性，听起来好像也没有彻底解决实时计算的复杂度，但至少不会造成并发锁的冲突。

而且如果考虑到定时任务，那我们则可以认为`points_status.points`值就是精确的。即便是前面提到的数据量大而选择分批执行，我们也可以通过增加一个字段来存储是否精确，以此来最大程度减少频繁实时计算带来的性能问题。

我们再来看看之前提到的业务指标是否得到了满足：

1. 积分存在有效期，过期作废
2. 用户可以查看当前可用积分总数
3. 允许用户看到累积过期的积分总数
4. 结算时，若用户选择使用积分，需要优先使用快要到期的积分部分
5. 退款时，若用户之前有使用积分，需要按照积分的有效期，优先退还到期时间晚的积分部分

第1、2，4点，直接在`points_status`表单条记录中就可以得到答案；
第3点，可以从`points_logs`表中按照`type==expire`的条件拿到总数；
第5点，需要借助`points_logs`中`type==cost && order_id==退款订单id`的条件得到需要退款的积分数值和对应的批次，再去`points_status`表中进行具体查找修改即可（若在对应的记录中已经找不到对应的积分批次了，则说明该批次积分已经过期了，此时不需要其它操作，直接将需要退还的积分直接以`type=expire`类型插入积分流水表即可）。

这样的设计，解决了一部分问题。
可是，还有更好一些的设计方案吗？

我相信答案一定是肯定的，请留下你的看法，谢谢~~
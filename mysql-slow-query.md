### 记一次线上Mysql慢查询问题排查

#### 背景：

上午公司群报警， 表象是消息数据库连接数过多， 提示报错如下：

```
| mysqli_connect(): (08004/1040): Too many connections in 
```


### 确实问题，线上解决问题：

0. 后端查询监控， 有小量流量高峰， 但不至于导致消息数据库持续报警， 排除是高峰流量打垮。

1.  DBA排查慢查询日志， 找到慢查询SQL：

```
SELECT * FROM notice  WHERE user_id in ('7143915',0) AND notice_id>'334241216' AND type !=5 AND have_read=0  ORDER BY notice_id DESC LIMIT 3
```

2. 线上持续报警， DBA猜测是该慢查询导致， 线上杀死查询进程。 

3. 线上报警停止， 基本确认是该慢查询SQL导致。


### 追查根本原因


1 . 根据报错SQL， 我们用Mysql的`explain`工具分析。

```
explain SELECT * FROM notice  WHERE user_id in ('25855',0) AND notice_id>'12222222' AND type !=5 AND have_read=0  ORDER BY notice_id DESC LIMIT 3
```

发现是使用的`PRIMARY KEY`, 命中rows：18308552。 命中条目过多，导致查询效率降低

2 .  到这里我们需要看下数据表结构 和数据表总量 

```
表结构：
CREATE TABLE `notice` (
 `notice_id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
 `user_id` bigint(20) NOT NULL DEFAULT '0' COMMENT '被通知的用户id',
 `message` text NOT NULL COMMENT '通知正文',
 `have_read` tinyint(4) NOT NULL DEFAULT '0' COMMENT '是否是新消息 0：是新消息',
 `creation_date` datetime DEFAULT NULL,
 `type` tinyint(4) NOT NULL DEFAULT '1' COMMENT '类型',
 PRIMARY KEY (`notice_id`),
 KEY `user_id_combine` (`user_id`,`have_read`) USING BTREE,
 KEY `user_id_type` (`user_id`,`type`)
) ENGINE=InnoDB AUTO_INCREMENT=138635402 DEFAULT CHARSET=utf8 COMMENT='公告表'

数据总条目：
34219964
```

所以我们发现查询出的数据太多。召回太多

观察表结构，发现联合索引

```
KEY `user_id_combine` (`user_id`,`have_read`)
```
比较适合我们， 查询WHERE中有对user_id, have_read做操作 
即

```
user_id in ('25855',0) AND have_read = 0
```

3 . 引入强制索引。

在这里我们引入强制索引， 强制使用`user_id_combine`索引

查看更新后sql

```
explain SELECT * FROM notice force index(user_id_combine) WHERE user_id in ('25855',0) AND notice_id>'12222222' AND type !=5 AND have_read=0 ORDER BY notice_id DESC LIMIT 3
```

结果是没有使用到索引， 查询rows总数：36617105

居然没有用到指定的`user_id_combine`

DBA猜测:
> 组合索引，前面那个字段，只能用等于查询，不能用in

觉得不太靠谱， 组合索引第一个字段和其他普通索引一样， in的话， 也是可以使用索引的

尝试将`user_id in ('25855',0)` 中 `'25855'` 写成数字，试一下

```
explain SELECT * FROM notice force index(user_id_combine) WHERE user_id in (25855,0) AND notice_id>'12222222' AND type !=5 AND have_read=0 ORDER BY notice_id DESC LIMIT 3
```

命中`user_id_combine`索引。

猜测是类型转换导致， 修改代码，完成修复

4 . 再追溯根本原因

网上查阅资料， 数据表字段类型 和WHERE查询类型如果不一样， 会导致Mysql的隐式类型转换。 即mysql自动帮我们转换数据类型。

在mysql in 中， 如果有多种数据类型， 则不进行隐式类型转换。 如果是同类型， 则可以使用索引。

`user_id in ('25855',0)` 写成

`user_id in ('25855','0')`

`user_id in (25855,0)`

都可以正常使用索引


参考资料：

- https://www.jianshu.com/p/6f34e9708a80
- https://dev.mysql.com/doc/refman/5.6/en/type-conversion.html
- http://seanlook.com/2016/05/05/mysql-type-conversion/
- https://yuerblog.cc/2017/05/22/mysql-implicit-type-cast/







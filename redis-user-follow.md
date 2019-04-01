## 用户关注业务Redis报错问题记录

### 背景

最近几天存储用户关注关系的Redis实例，通过监控，发现报错相对频繁， 特此提前分析。 在高峰流量到来时，顺利化解性能瓶颈。

### 当前现状

用户关注关系， 即存在「我的关注」「我的粉丝」两种列表结构。

直观操作是，当用户A对用户B进行关注时：

1. 将用户B加入到用户A的关注列表。
2. 将用户A加上到用户B的粉丝列表。

为了记录用户关注动作执行的 当前时间，并且为了在列表展示时，通过该时间进行排序。

我们选择使用Redis的`有序集合`结构来存储用户关注源数据


1. 我的关注列表key设计

```
key名: "u_follow:{user_id}"
member: {following_user_id}
score: {follow_time}
```

2. 我的粉丝列表key设计

```
key名: "u_fans:{user_id}"
member: {fans_user_id}
score: {fans_follow_time}
```

#### 具体业务场景

* 获取用户A的关注总数：

```
ZCARD "u_follow:{A_user_id}"
```

* 获取用户A的粉丝总数:

```
ZCARD "u_fans:{A_user_id}"
```

* 用户A关注用户B:

```
ZADD "u_follow:{A_user_id}" {unixtime} {B_user_id}
ZADD "u_fans:{B_user_id}" {unixtime} {A_user_id}

ZADD命令(时间复杂度O(M*log(N)))
```

* 校验用户A和用户B的关注关系:

```
判断用户A是否在用户B的关注列表中， 并且用户A是否在用户B的粉丝列表。
通过ZRANK(时间复杂度O(log(N)))命令来实现

ZRANK "u_follow:{B_user_id}" {A_user_id}
ZRANK "u_fans:{B_user_id}" {A_user_id}

```

* 获取用户B的关注列表， 粉丝列表

```
通过ZREVRANGE(O(log(N)+M))命令来实现

ZREVRANGE "u_follow:{B_user_id}" 0 20 WITHSCORES
ZREVRANGE "u_fans:{B_user_id}" 0 20 WITHSCORES
```

* 业务上为了防刷， 单用户关注上限为500人


### 问题分析

+ 通过上面表结构设计， 我们可以发现：

1. 数据有冗余， 即进行一次关注动作，进行两次写操作
2. 粉丝列表没有限制， 可能会无限增大，最终出现大KEY

+ 通过业务监控发现：

1. 读请求 集中在判断用户A和用户B的关注关系(ZRANK)。
2. 写请求较少


### 着手优化

#### 阶段1

通过业务监控发现的问题(1) 存在请求浪费情况

业务方是希望获取 用户A是否关注用户B，但现存接口返回的是 用户A和用户B的关系(用户B是否在A的 关注列表中， 用户B是否在用户A的粉丝列表中)。

针对性优化接口，按需所取： 即请求什么则返回什么. 不额外返回。

#### 阶段2

猜测可能是因为线上存在大KEY, 拖累Redis

线上数据观察， 用户总关注关系日志为千万级， 单用户粉丝列表最多20万级。ZRANK时间复杂度是O(log(N)), 20万数据最多计算21次， 不至于太慢，log(N)复杂度可接受。

排除掉是因为线上存在大KEY导致的问题。但是需要解决粉丝列表大key情况

我们看其请求量， 已经快接近redis单机上限。 因此再一步猜测， 是因为达到单机Redis上限，引起的报错

#### 阶段3

根据目前业务上主要查询，发现， 粉丝列表设计不合理。 完全可以只存储用户最近1000或者固定粉丝数。没必要完全存储全部粉丝列表

+ 用户A和用户B的关注关系调整为

1. 校验用户A是否在用户B的关注列表里: 
2. 校验用户B是否在用户A的关注列表里:

```

redis中操作为
ZRANK "u_follow:{B_user_id}" {A_user_id}
ZRANK "u_follow:{A_user_id}" {B_user_id}

```

+ 用户A的粉丝列表只存储最新的1000个粉丝， 获取用户粉丝列表，如果超过限制，则从mysql中获取

```
ZCARD "u_fans:{A_user_id}" <= 1000

ZREVRANGE "u_fans:{A_user_id}" 1000 20 WITHSCORES

```


+ 用户A的实际粉丝总数，调整为key value方式存储：

```
 //增加用户A粉丝数
 INCR "u_fans_num:{A_user_id} 1
 //减少用户A粉丝数
 INCR "u_fans_num:{A_user_id} -1
 //获取用户A粉丝数
 GET "u_fans_num:{A_user_id}

 //每天凌晨， 通过查询用户关注log, 获取活跃用户ID， 定期从数据库刷新粉丝数到redis中， 做到每天校正
 
```

`总结`

+ 解决大key问题：
 
 * 限制用户最新粉丝总数总数<1000

+ 提高Redis实例并发:
   * 增加Redis实例从库
   * 将用户ID分段，不同段用户ID放入不同的redis实例中

最终上线：

//todo 阶段3优化等待上线验证。

参考：

https://infoq.cn/article/weibo-relation-service-with-redis














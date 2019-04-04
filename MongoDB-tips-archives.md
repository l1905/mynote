# MongoDB 使用tips(归档日志)

组内使用MongoDb 记录日志， 估算了下， 日志太大， 因此期望归档时间较久的日志。

调研发现有两种策略

1. 定期过期数据
2. 固定集合大小


## 1. 定期过期数据：

### 主动过期
MongoDB支持文档带过期时间。即新建时间索引， 字段新增`expireAfterSeconds`属性

支持两种过期方式:

1.指定过期时间， 类似Redis的expire命令

    expireAfterSeconds: 过期秒数
2.自定义过期时间点， 类似Redis的expireat

    expireAfterSeconds: 0
时间必须是 isodate格式，， 不支持 Y-m-d H:i:s字符串格式

PHP类库调用方式为:

```
new \MongoDB\BSON\UTCDateTime($utc_time)
```

### 删除

但是当我们从MongoDB删除文档或集合后， 并不会将已经占用的磁盘空间释放， 会一直维护已经占用的磁盘空间数据文件。

为了更加有效的使用磁盘空间， 我们需要对mongodb数据文件做碎片整理以及未使用空间的回收。

回收主要使用`db.repairDatabase()`命令，但会阻塞进程，线上慎用。

## 1. 固定集合大小：

固定集合文档大小，如果超出，则覆盖最久文档。

### 创建方式

```
db.createCollection( "log", { capped: true, size: 100000, max : 5000 } )
```

size: 指定字节大小， 最小字节是 4096

max: 主要指定最大文档条目。

判断规则是：size, 和max最先到触发删除， 类似汽车保修三年或者10万公里


### 其他特点：

1. 检查是否是固定集合 `db.collection.isCapped()`



参考文章：

https://yq.aliyun.com/articles/606187

https://docs.mongodb.com/manual/tutorial/expire-data/

https://docs.mongodb.com/manual/core/capped-collections/
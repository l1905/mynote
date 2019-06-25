# 关注分析


用户关注， 用户粉丝， 相对于中台服务，大部分社区网站都会使用该服务， 我们以此为出发点，来分析B站关注关系的设计

## 关注操作

我们看下当用户A对用户B，进行关注操作时， 会经过哪些逻辑处理。

1. 首先对该接口进行限速(这里引入中间件rate服务)
2. 判断被关注者是否在黑名单中， 判断关注着是否有绑定手机号
3. 获取关注摘要表(`user_relation_stat_%d`按当前用户ID分50张表), 不存在，则创建
	主要字段有如下：
	* mid： 用户ID
	* following： 关注总数
	* whisper： 悄悄关注总数
	* black：拉黑总数
	* follower: 粉丝总数
	* ctime： 创建时间
	* mtime： 修改时间
4. 开始db事务， 进行关注操作, 利用defer操作， 最后提交或者回滚事务
	* 查询摘要信息表，获取摘要
	* 查询用户A我的关注表，判断关注属性
		用户我的关注表`user_relation_mid_%d` 按用户A的ID分500张表， 主要有以下字段
		1. mid: 用户ID
		2. fid： 关注者用户ID
		3. attribute： 属性，bit位标记 0位悄悄关注 1位正在关注，2位互粉 7账号拉黑
		4. source： 来源
		5. status： 关注状态 0正常，1取消关注
		6. ctime： 创建时间
		7. mtime： 修改时间
	   
	   用户我的粉丝表`user_relation_fid_%d`， 按用户B的ID分为500张表，字段和`relation_mid`功能一样
	* 更新用户A， 用户B的摘要信息表
	
5. 更新缓存
	* 用户B的最近关注用户列表
		1. 存储方式为redis的有序集合， 即最近关注的用户列表
	* 存储最近是否有新关注提醒
		1. 存储方式 redis hash， key：mid， value： 已读状态
		2. 对用户ID取模10000， 分为1000个小hash

## 查看用户A与一组用户的关注关系

1. 用户关注列表， 放在 redis的 hash中
	* `key`的格式为`at_%d`， `sub_key`: `fid`, `sub_value`: `json_encode(属性)`, 先从缓存中获取，缓存不存在，从数据库中获取

## 查看用户A的关注列表 和粉丝列表

1. 数据存储在memcache 中， protobuf压缩
2. 粉丝列表只存储最新的1000条数据


## 其他

attribute位字段封装的相对优雅，接口封装明确， 使用`<<`，明确语意， 后期自身设计位运算，可以做参考， 代码摘选

```
//属性预定义
AttrNoRelation = uint32(0) //
AttrWhisper    = uint32(1) //0位 是悄悄关注
AttrFollowing  = uint32(1) << 1 //1位是正在关注
AttrFriend     = uint32(1) << 2 //2位是 互粉
AttrBlack      = uint32(1) << 7 // 7位是拉黑

// SetAttr set attribute. 设置属性
func SetAttr(attribute uint32, mask uint32) uint32 {
	return attribute | mask
}

// UnsetAttr unset attribute. 撤销属性
func UnsetAttr(attribute uint32, mask uint32) uint32 {
	return attribute & ^mask // ^ 按位取反
}
```


## 点评

用户关注表冗余出两张表的优点：

1. 将表拆小
2. 拆表后，不影响 我的粉丝 列表获取

缺点：

1. 事务使用较多，逻辑相对较复杂， 容易出现死锁 并且并发性较低

    	

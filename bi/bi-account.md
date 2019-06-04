# 账号体系分析----001


从源码中，我们可以看到，其将账号体系拆分出众多微服务， 各自高内聚，将功能提供最小化， 我们先以`passport-login`服务为切入点，探究其内在实现


## passport-login 服务


主要提供以下三种接口类型

1. 通用接口： 比如获取加密rsa的公钥
2. 用户接口： 用户账号密码登录接口
3. 用户cookie凭证管理： cookie的增删改查接口, web端使用
4. 用户token凭证管理：token的增删改查接口，客户端使用


通过以上我们列举的接口类型，基本可以确认， 此服务主要用于管理用户登录凭证， 以及账号密码登录。 
下面我们分析具体接口实现。 这里没有存放`用户手机号短信登录``三方登录(微信等)`等这些校验逻辑

### 1. 获取加密rsa公钥接口

从`service`发现， 其加密公钥是放到数据库中存储的， 在项目运行时， 从数据库中获取所有加密类型，放到内存中， 以实现密钥和代码隔离。

但是可能出现的问题， 如果数据库中密钥调整， 则必须要重启项目。 

存放密钥数据库为 独立数据库`passport_secret`, 对应表名为 `user_secret`, 表结构如下

1. key_type : 密钥类型， 0rsa加密公钥,1 rsa加密私钥, 2.aes加密key,3. hash的salt盐
2. key： 密钥值

### 2. 账号密码登录校验接口

输入为：账号， input密码

首先检查账号类型是属于手机号，还是邮箱， 内部user_id

以手机号登录：

1. 先加密手机号（AES）
2. 通过加密的手机号 查询对应的用户ID，即mid
3. 根据mid获取用户基础信息，拿到db中密码
4. 将input密码 和db密码做校验比对， 获取最终结果


业务逻辑没什么可以说的， 这里我们挑选感兴趣的来讲。

#### 数据表设计

1. 用户基础表

	mid： 用户ID
	
	userid： 内部用户ID(怀疑是兼容老系统遗留)
	
	pwd: 密码
	
	status： 账号状态
	
	deleted： 是否删除 
2. 手机号关联mid表

	mid:用户ID
	
	tel：手机号
	
3. 邮箱关联mid表

    mid: 用户ID
    
    tel: 手机号

这里不是太理解，建立手机号-mid， 邮箱-mid关联表的意义， 何不直接都放到用户基础表，这样还能减少一次sql查询。

#### 密码加密解密校验方式

1. 前端通过获取的RSA加密公钥对「用户输入的密码+当前客户端时间」一块加密 传给服务端
2. 服务端先通过私钥进行 解密， 获取当前客户端时间 + 用户输入密码
3. 校验客户端时间是否合法， 即是否是最近加密生成
4. md5(输入密码， db中盐) == db中密码 


### 3. 用户cookie凭证管理(web端)

这里主要对cookie有以下操作

1. 新增cookie， 即登录成功后，生成用户cookie
2. 删除cookie， 即退出登录后，删除用户cookie
3. 删除单用户全部cookie, 即将该用户强制退出登录

我们先来看下cookie的生成方式

1. 根据mid,时间戳 生成session
2. 生成`csrf`值, 这里是随机数
3. 设置cookie类型， 比如0是登录cookie
4. 设置有效时间点, 即从生成cookie时间起， 1个月后过期
5. 先插入数据表. `user_cookie_%s`
	数据表以年月来分表，比如 `user_cookie_201905`
	表结构如下 mid,session,csrf,type,expires
	
	* mid :用户ID
	* session： 返回web浏览器
	* csrf： csrf值
	* type: 类型
	* expires： 过期时间点
	
6. 更新到memcache缓存中(memcache失效时间从配置中获取， 可能是24h，即一天)
7. 返回客户端session

优点：

1. 合理管理cookie, 避免cookie表过大
2. 合理使用memcache缓存， 仅存储活跃用户的cookie信息， 节约memcache资源
3. 方便强制使用户退出

缺点：

1. cookie过期逻辑相对较复杂， 如果出现跨年情况，或者需要延长过期时间，比如过期时间从1个月，延长到2个月， 需要合理使用。


### 2. 用户token凭证管理（app端)

这里主要对token有以下操作

1. 新增token， 即登录成功后，生成用户token，以及refresh_token
2. 删除token， 即退出登录后，删除用户token
3. 删除单用户全部token, 即将该用户强制退出登录


生成token的流程 和生成cookie非常类似， 有主要以下不同:

1. 引入`refresh_token`比较像 `auth2.0`的概念。当`access_token`将要过期时， 可以拿着`refresh_token`来换新的`access_token`, 此后`refresh_token`失效, 即只能用一次
2. 生成token 时， 不仅要传`mid`, 还要传`appid`， 这里猜测 `appid`和客户端版本，或者应用调用版本有关， 即用来区分开放平台 和自身客户端来登录。
3. token有效期为一个月， refresh_token有效期为2个月， 即token通过refresh_token延长有效期最多为2个月。


主要表结构：

1. 表`user_cookie_%s`,存放token数据  按年月进行分表, 比如 `user_cookie_201905`
   主要字段有以下：
   
   * mid： 用户ID
   * appid: 应用ID
   * token： token值
   * expires： token有效时间点
   * type： token类型

2. 表`user_refresh_%s`, 存放`refresh_token`数据，按年+奇数月进行分表, 比如`user_refresh_201905` 或者 `user_refresh_201903`， 主要字段有以下：

	* mid: 用户ID
	* appid: 关联应用ID
	* refresh: refresh_token
	* token: 关联token值
	* expires: 过期时间点


目前章节没有注册+ 三方登录逻辑， 我们下面分析继续分析
















    





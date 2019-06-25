# B站代码分析流程

主要解析发后端实现过程, 以评论下项目为例， 来主要看

1. 对外接口： 提供内部服务
2. 后台bgm服务： 后台管理员操作服务
3. job服务： 评论作业服务

## 对外接口 
简单介绍b站代码结构

对外网关服务 主要放在 `go-common-master/app/interface/`目录中， 基本涵盖了B站所有对外代码。

评论项目目录则在 `go-common-master/app/interface/main/reply/`目录

评论目录结构有以下：

```
cmd: 服务的main函数入口
conf： 配置文件初始化
dao： 对应PHP的model层
http： route路由部分+controler部分
model： 对应数据结构体层
service: 业务主要处理层，类似我们的biz层
```

部署评论方式：


1. 通过cmd中的main 拉起服务，初始化配置文件(conf), redis, memcache, rpcclient, mysql等
2. 请求进来，找到对应路由函数(http)， 映射对应的service层处理函数, 初始化数据，调用service层。 service层调用dao层， dao层调用model层。 处理完毕，返回最终处理结果




发评论接口

url: `https://api.bilibili.com/x/v2/reply/add`

```
method: post
request参数含义：
oid: 文章ID
type: 文章类型
message: 发布消息
plat: 平台
jsonp: jsonp
csrf: 4dd143e7ec6c2136b9438db496fee120
```

我们首选在 http目录的http.go中找到路由map方法
`/Users/litong/Downloads/go-common-master/app/interface/main/reply/http/http.go`

`group.POST("/add", authSvc.User, addReply)`

知道请求被分发dispatch到http的 `addReply`方法。 我们去看下`addReply`方法实现。

对应方法中 

1. 首先是初始化请求参数,
2. 其次 检查参数合法性， 校验参数, 
3. 获取登录凭证内部用户ID mid， 校验用户合法状态(小黑屋 是否被禁用)

```
root = 0 && parent = 0 
则判断为是 直接回复评论 
rp, captchaURL, err = rpSvr.AddReply(c, mid, oid, int8(tp), int8(plat), ats, ak, c.Request.Header.Get("Cookie"), captcha, msg, device, version, platform, build, buvid)

否则 认为是回复 某一条评论

rp, captchaURL, err = rpSvr.AddReplyReply(c, mid, oid, root, parent, int8(tp), int8(plat), ats, ak, c.Request.Header.Get("Cookie"), captcha, msg, device, version, platform, build, buvid)

```


先看直接回复评论方法 
rpSvr.AddReply()
对应文件路径： `/Users/litong/Downloads/go-common-master/app/interface/main/reply/service/reply.go`

```
还是先分析方法参数

c : 上下文环境
mid : 用户ID
oid ：文章ID
tp： 文章类型
pat: 平台， 比如是web，还是安卓
ats: @用户ID列表
accessKey: 等待确认
cookie： cookie内容，为啥需要传进来
captcha： 验证码code
msg： 消息内容
dev: 设备信息
version： 版本信息
build： 打包信息
buvid： 这个不清楚具体作用

```

缺点：
参数平铺在一起， 建议是不重要参数可以收敛到一个map中处理

查看方法中具体业务逻辑

```
1. validateReply 校验状态
    比如用户积分是否达到可以发评论
    检查发表评论用户是否被 up主拉黑
    校验subject， 获取不到，则初始化
        subject源数据： 数据表按文章ID分50片
            oid： 文章ID
            type 文章类型
            mid: upper主信息
            count, 一级评论数
            rcount, 未知,恢复总数， 不清楚具体用在哪里
            mcount, 监控总数
            acount, 全部评论总数
            state, 状态
            attr, 锁定属性
            ctime, 创建时间
            mtime,修改时间
            meta， 未知

2. 处理emoji表情
    比如[xxx] 替换为【xxx】 不清楚用意
3. 监管检查 SuperviseReply
    某一时间段内，某一文章类型 或者海外用户禁止发布评论
4. 持久化数据，即打队列 persistReply

```

这里详细看下持久化数据的业务逻辑  `persistReply`

```
1. 通过发号器获取评论ID
    rpID, err := s.nextID(c)
2. 初始化数据
    RpID:   rpID,
        Oid:    subject.Oid,
        Type:   tp,
        Mid:    mid,
        Root:   root,
        State:  reply.ReplyStateNormal,
        Parent: parent,
        CTime:  cTime,
        Dialog: dialog,
        Content: &reply.Content{
            RpID:    rpID,
            Message: msg,
            Ats:     ats,
            IP:      xip.InetAtoN(ip),
            Plat:    plat,
            Device:  dev,
            Version: ver,
            CTime:   cTime,
        },
3. 大数据机器学习过滤bigDataFilter
4. 内容过滤 checkContentFilter2
5. 校验 subject 属性, 
6. 打到数据总线 Databus.AddReply 即kafka数据队列中

```

前台发布评论流程走完。


## 后台bgm服务


bgm服务 主要放在`go-common-master/app/admin/`目录中， 基本涵盖了B站所有对外代码。

评论项目放在`go-common-master/app/admin/main/reply/`目录中。

目录代码结构 和API项目类似

我们查看路由方法
`/Users/litong/Downloads/go-common-master/app/admin/main/reply/http/http.go`

主要分为两块：

```
1. 管理员后台操作 接口
    需要鉴权(即管理员是否有权限)
    例如：
    group.GET("/search", auther.Permit("REPLY_READONLY"), replySearch)     
2. 提供内部接口
    供内部接口使用
    例如：
    group.GET("/log", logByRpID)
```


我们先看下后台搜索列表接口：

replySearch
`/Users/litong/Downloads/go-common-master/app/admin/main/reply/http/reply.go`

```
搜索参数如下：
type： 频道ID
oid: 文章ID
uid：用户ID
adminid： 管理员ID
start_time： 开始时间
end_time： 结束时间
ip： 发布评论IP
page：页码
pagesize： 限制返回总数
typeids： 频道ID数组
Keyword： 关键词，估计是搜索内容
keyword_high： 未知， 关键词相关
nickname：昵称
states： 文章状态
sort： 排序方式 asc, desc
order： 具体排序字段，比如oid, uid
attr： 属性筛选
adminname： 管理员名称
```

查看dao层 我们可以看到， 后台搜索直接走的ES内部服务

`/Users/litong/Downloads/go-common-master/app/admin/main/reply/dao/http_search.go`

评论信息 单独走ES索引表
管理员操作信息，单独走ES索引表

思考：

这里的ES不单是后台使用， 还为用户个人中心 查看自己的评论时使用。

下面我们看个 评论审核通过接口：

adminPassReply

```
传参列表：
oid： 文章ID
type： 文章类型
rpid： 评论ID
adid： 管理员ID
remark： 扩展参数
```

```
删除具体逻辑有如下
1. 获取要删除的评论列表
2. 遍历评论列表， 对每个评论 开启协程，单独处理业务逻辑， 并发提高速度
3. 如果 目前评论状态是 先审后发，则通过 recReply，修改评论状态， 更新前台缓存， 从等待审核中删除缓存
4. 否则的话，更新评论状态， 如果是受监控状态， 则更新状态
5. 加入kafka消费队列中： 不清楚这里report的含义，是哪里用到 //todo
6. 加入管理员操作日志
    1. 将评论设置为非新评论, 不明白业务逻辑
    2. 插入管理员操作日志
        INSERT INTO reply_admin_log (oid,type,rpid,adminid,result,remark,isnew,isreport,state,ctime,mtime) VALUES

7. 更新ES索引文件，使用的是 upsert(存在就更新，不存在就插入)
```

## JOB作业服务

job服务 主要放在 go-common-master/app/job/目录中， 基本涵盖了B站所有对外代码。

评论项目放在 go-common-master/app/job/main/reply/目录

```
启动评论job方式：
1. 通过cmd中的main 拉起服务，初始化配置文件(conf), redis, memcache, rpcclient, mysql等
2. main中初始化service New方法， 拉起消费协程(kafka-->channel)，拉取实际消费协程(channel) 执行业务逻辑

    1. 所有的评论处理信息 都打到一个kafaka队列里， 一个总消费协程负责消费
        dataConsume
    2. 根据文章ID分片， 将不同消息，打到8个channel中 ?疑问进程挂掉后， 没有真正消费的怎么办
    3. 8个协程 开始真实消费消息。
        consumeproc
    4. 点赞是单独处理， 我理解点赞是高频操作，怕影响正常业务消费 
        likeConsume ---->consumelikeproc

    5. 数据变动 同步ES 协程， 这里感兴趣的一点，这是里insert还是update操作
        searchproc
    6. map协程序
       mappingproc， 暂时不太清楚用途
```

现在主要看 `consumeproc` 实际消费流程：
根据不同的action 操作类型， 执行不同的业务处理。

* add： 发布评论
* add_top： 增加置顶评论
* rpt：举报


下面我们看异步发布流程

```
app/job/main/reply/service/reply.go  文件对应的addReply 方法

1. 首先获取subject 主题信息
2. 更新subject 的RCount， ACount, Count 都+1
3. 更新楼层floor, 修改时间MTime， 更新评论内容的rpid，创建时间CTime，修改时间MTime, Ats列表(根据@正则匹配)，topic信息(根据##正则匹配)
4. 开始事务写mysql 数据 tranAdd， 我们具体看下对应实现
    主题表subject修改： 根据文章ID 分50张表
        count： 楼层总数
        rcount: right_count 正常总数, 一级评论的正常数 针对root, 如果非root评论，则是子评论的总数
        acount: 全部评论总数
        mcount: monitor_count 受监控总数

        开始mysql事务 begin
        如果是正常评论， 则subject的count=count+1,rcount=rcount+1,acount=acount+1
        如果是非正常评论，则只有 count=count+1

        如果是检查或者是监控状态， 则 subject的 mcount=mcount+1， 即 monitor_count 受监控总数

    内容表content插入，这个表都是用户原始提交的信息：分200张表
        NSERT IGNORE INTO reply_content_%d (rpid,message,ats,ip,plat,device,version,ctime,mtime,topics) VALUES(?,?,?,?,?,?,?,?,?,?)
    评论索引表插入，这个表主要用于前台检索：分200张表
        INSERT IGNORE INTO reply_%d (id,oid,type,mid,root,parent,dialog,floor,state,attr,ctime,mtime) VALUES(?,?,?,?,?,?,?,?,?,?,?,?)
5. 最后提交事务, 后期如果分库， 则相同hash 的评论信息，必须分到同一个mysql数据库
    比如我们要分10数据库, 分库问题处理， 怎么这50张表合理分配， 200张表合理分配
    10个数据库，
    第9个数据库 
       subject索引：
       09, 19, 29, 39, 49  
       reply 分表结构
       09, 19, 29, 39, 49, 59, 69, 79, 89,99, 109, 119,129,139, 149,159,169,179,189, 199
       content分表结构
       09, 19, 29, 39, 49, 59, 69, 79, 89,99, 109, 119,129,139, 149,159,169,179,189, 199

    (oid%10)分库, (oid%50)分subject表，(oid%200)分reply表， 主要分库个数必须是 50 和200的公约数
6. 新增到memchace缓存
    subject缓存：
    key特征：s_oid_type, 
    value: 直接是subject对象
    expire:过期时间 一个小时(测试环境)

    reply缓存：
    key特征：r_oid, 这里比较奇怪， 没有用到type, 
    value: 直接是reply对象
    expire:过期时间 一个小时(测试环境)
7. 如果是正常评论， 
    a. 发布 全部评论总数mq(业务方订阅消费， 这个是总数 acount)
    b. 检查redis缓存是否过期， 如果没有过期，则追加：列表数据
        redis为有序集合， 长度上限为20000, 测试环境过期时间为8小时， 格式为 
        %s_%d_%d_%d， 即 i_oid_type_sort 最后一个是排序方式， 可能是根据楼层floor，总数count,喜欢like(喜欢这个是有权重的概念)
        redis类型： 有序集合

        追加方式： ZADD->EXPIRE

        楼层排序类型
        sub-key: rpid
        value: floor

        总数排序：
        sub-key: rpid
        value: RCount

        喜欢排序
        sub-key: rpid
        value: score(通过算法，喜欢数)

    c. 发送通知消息 (比如您收到了xx条评论)
    e. 发送 公共事件
```




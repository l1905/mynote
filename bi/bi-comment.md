# comment项目分析

语言主要基于golang , 关系型数据库使用mysql, 缓存使用memcache+redis, 后台搜索+个人中心评论列表使用ES

我们依次进行mysql， memcache, redis 分析

## 数据库分析

主要分为以下表结构

1. 评论内容表：主要存放内容相关
2. 评论摘要索引表： 主要存放排序检索相关
3. 文章所属评论摘要表：主要存放文章所属的评论信息，比如文章的评论总数， 受监控评论总数
4. 文章举报摘要表：以文章ID文主键， 存放举报相关摘要信息
5. 用户举报日志表：存放用户举报流水相关
6. 管理员操作日志： 主要存放管理员后台操作日志
7. 附属表信息： 比如针对文章的配置信息表， emoji管理表, 点赞，点踩流水， 这里不详细说了


### 1. 评论内容表 

主要作用： 存放评论主信息

`reply_content_%d`, 按文章ID，进行取模分表， 分200张表
 
 1. rpid： 文章ID
 2. message: 评论内容
 3. ats： @列表
 4. ip： 发表文章时的IP信息
 5. plat： 平台， 0未知，1web，2安卓，3iphone，4wp 5 ipad, 6wp win10
 6. device： 设备信息
 7. version： 版本信息
 8. ctime： 创建时间
 9. mtime： 修改时间
 10. topics： 主题相关信息， 比如#主题
 
 
### 2. 评论摘要索引表

主要作用： 用户排序等功能

`reply_%d`, 按文章ID，进行取模分表， 分200张表

1. id: 评论ID
2. oid: 文章ID
3. type： 文章类型, 1视频，2主题，3
4. mid： 评论ID作者
5. root： 根评论ID
6. parent: 父评论ID
7. dialog: 对话ID， 一级子评论的会话ID是自己， 其子子评论的会话ID是父评论ID， 即会话以一级子评论开始
8. floor： 楼层
9. state: 评论状态, 0正常，1被up主隐藏，2被过滤, 3被管理员删除, 4被用户删除, 5被监管，6大数据过滤, 7被管理置顶，8被up主隐藏,9在黑名单中，10被协管删除，11被审阅 12被折叠
10. attr： 属性信息， bit位标记, 0位：管理员置顶，1位：up主置顶, 2位：垃圾， 7位：折叠
11. ctime: 创建时间
12. mtime: 修改时间
13. count: 评论总数， 子评论相关
14. rcount: 正确展示评论总数， 子评论相关
15. like： 点赞总数
16. hate： 点值总数


### 3. 文章所属评论摘要表

主要作用： 记录用户总数相关信息， 比如评论总数

`reply_subject_%d` ： 按文章ID，进行取模分表， 分50张表,

主要有以下字段：

1. oid: 文章ID
2. type: 文章类型
3. mid： 文章作者
4. count： 评论总数(一级)
5. rcount：正确评论总数
6. acount: 全部评论总数
7. state: 状态
8. attr： 属性信息, bit位标记, 0位：管理员置顶，1位：up主置顶, 7位：折叠
9. ctime：创建时间
10. mtime: 修改时间
11. meta： meta信息

### 4. 举报摘要表

主要作用： 记录文章举报摘要相关数据

`reply_report_%d` 按文章ID，进行取模分表， 分200张表

1. oid: 文章ID
2. type: 文章类型
3. rpid： 最后评论ID(todo 不确认）
4. mid： 最后举报作者ID(todo 不确认)
5. reason: 最后举报原因
6. content: 举报内容
7. count: 被举报总数
8. score: (todo 不确认)
9. state: 举报状态, 0待一审,1移除,2忽略,3一审移除, 4待二审, 5二审移除,6二审忽略,7一审忽略,8举报转移风纪委
10. Attr: 属性信息，bit位标记,State状态是否曾今从待一审\二审转换成待二审\一审
10. ctime: 创建时间
11. mtime： 修改时间


### 5. 举报日志表

主要作用：记录用户举报操作

`reply_report_user_%d` 根据文章ID，分200张表

oid,type,rpid,mid,reason,content,state,ctime,mtime

1. oid: 文章ID
2. type: 文章类型
3. rpid： 评论ID
4. mid： 举报者 用户ID
5. reason: 举报原因 0其他 1广告 2色情 3刷屏，4引战 5剧透 6政治 7人身攻击 8视频不相关 9违禁 10低俗 11非法网站 12赌博诈骗 13传播不实 14怂恿教唆 15侵犯隐私 16抢楼
6. content: 举报状态
7. state: 举报状态, 0新增 1已反馈
8. ctime: 创建时间
9. mtime: 修改时间


### 6. 管理员操作日志

主要作用： 记录管理员操作相关

`reply_admin_log`

oid,type,rpid,adminid,result,remark,isnew,isreport,state,ctime,mtime


1. id： 自增ID
2. type: 文章类型
3. oid： 文章ID
4. rpid: 评论ID
5. adminid: 管理员ID
6. result: 未知todo
7. remark： 标记
8. isnew: 是否是最新评论
9. isreport: 是否是举报
10. state： 状态, 0管理员删除评论，1管理员通过举报删除[已废弃], 2管理员忽略举报[已废弃], 3管理员恢复评论，4管理员编辑内容，5管理员通过待审，6修改主题内容，7置顶评论，8修改主题用户ID,9举报一审忽略，10举报二审忽略，11举报一审删除，12举报二审删除 13举报一审恢复，14举报二审恢复，15对点赞点踩设置 16up主删除评论, 17用户删除评论， 18协管删除评论 19设置监控状态,20转一审，21转二审，22移交仲裁，23设置举报状态，24先发候审打开，25先发后审关闭， 26先审后发打开， 27先审后发关闭，28标记为垃圾
11. ctime
12. mtime


## REDIS缓存分析

以获取评论列表为例子

1. 排序评论列表以有序集合方式存储

`key`格式为：`i_%oid_%type_%sort`

1. oid: 文章ID
2. type: 文章类型
3. sort： 排序类型 0通过楼层排序，1通过总数排序，3通过喜欢like排序

2. 受审核评论列表，以有序集合方式存储 -----> 目前没有找到调用方式

`key`格式如下 `ai_%oid_%type`
 
3. 二级子评论列表， 以有序集合方式存储

`key`格式为:`ri_%rpid`

member：子评论ID

score: 子评论创建时间


## MEMCACHE存储分析

1. 存储评论详情信息

`key`格式如下：

`r_%rpid`： 通过压缩成json， 存成k-v对，主要json数据字段为 `reply_%d`表中字段+ `reply_content_%d`

`reply_%d`中字段有

id,oid,type,mid,root,parent,dialog,count,rcount,`like`,hate,floor,state,attr,ctime,mtime

`reply_content_%d`中有：

rpid,message,ats,ip,plat,device,topics

2. 存储评论摘要信息

`key`格式如下：

`s_%oid_%type`, 主要有以下字段

id, oid, type, mid, count, rcount, acount, state, attr, meta, ctime, mtime.















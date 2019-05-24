# ES 刷新问题记录


###  问题背景


评论管理后台，使用ES， 修改ES数据的字段状态， 重新搜索，发现已修改完状态的数据信息依然能被检索到。


### 解决问题思路

1. 首先怀疑是页面浏览器缓存， 经验证不是
2. 怀疑是ES修改后未生效， 确实是该问题导致



网上找到的资料如下(https://blog.csdn.net/u011228889/article/details/80855431)：



ES的索引数据是写入到磁盘上的。但这个过程是分阶段实现的，因为IO的操作是比较费时的。


先写到内存中，此时不可搜索

默认经过 1s 之后会被写入 lucene 的底层文件 segment 中 ，此时可以搜索到

refresh 之后才会写入磁盘

以上过程由于随时可能被中断导致数据丢失，所以每一个过程都会有 translog 记录，如果中间有任何一步失败了，等服务器重启之后就会重试，保证数据写入。translog也是先存在内存里的，然后默认5秒刷一次写到硬盘里。

在 index ，Update , Delete , Bulk 等操作中，可以设置 refresh 的值。取值如下：

`refresh=true `

更新数据之后，立刻对相关的分片(包括副本) 刷新，这个刷新操作保证了数据更新的结果可以立刻被搜索到。

`refresh=wait_for `

这个参数表示，刷新不会立刻进行，而是等待一段时间才刷新 ( index.refresh_interval)，默认时间是 1 秒。刷新时间间隔可以通过index 的配置动态修改。或者直接手动刷新 POST /twitter/_refresh

`refresh=false `

refresh 的默认值，更新数据之后不立刻刷新，在返回结果之后的某个时间点会自动刷新，也就是随机的，看es服务器的运行情况。

那么选择哪种刷新方式？

wait_for 和 true 对比，前者每次会积累一定的工作量再去刷新

true 是低效的，因为每次实时刷新会产生很小的 segment，随后这些零碎的小段会被合并到效率更高的大 segment 中。也就是说使用 true 的代价在于，在 index 阶段会创建这些小的 segment，在搜索的时候也是搜索这些小的 segment，在合并的时候去将小的 segment 合并到大的 segment 中

不要在多个请求中对每一条数据都设置 refresh=wait_for ， 用bulk 去批量更新，然后在单个的请求中设置 refresh=wait_for 会好一些

如果 index.refresh_interval: -1 ，将会禁用刷新，那带上了 refresh=wait_for 参数的请求实际上刷新的时间是未知的。如果 index.refresh_interval 的值设置的比默认值( 1s )更小，比如 200 ms，那带上了 refresh=wait_for 参数的请求将很快刷新，但是仍然会产生一些低效的segment。

refresh=wait_for 只会影响到当前需要强制刷新的请求，refresh=true 却会影响正在处理的其他请求。所以如果想尽可能小的缩小影响范围时，应该用 refresh=wait_for


### 重点记录

这里的 `index.refresh_interval` 默认是 `1s` ，但支持毫秒表示方式， 我们修改成了 `200ms`


参考文章：

1. https://blog.csdn.net/u011228889/article/details/80855431
2. https://www.elastic.co/guide/en/elasticsearch/reference/5.5/docs-refresh.html
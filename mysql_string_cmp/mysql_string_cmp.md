### Mysql字符串 大小比较

需求背景：
客户端传递需求号， 比如 xxx.xxx.xx 9.3.15,  来获取数据表中对应的数据

数据表设计如下



```mysql
CREATE TABLE test_table (
 id bigint(20) unsigned NOT NULL AUTO_INCREMENT,
 start_version varchar(30) NOT NULL DEFAULT '' COMMENT '开始版本',
 end_version varchar(30) NOT NULL DEFAULT '' COMMENT '结束版本',
 PRIMARY KEY (id),
 KEY version_combine (start_version,end_version) USING BTREE
) ENGINE=InnoDB  DEFAULT CHARSET=utf8 COMMENT='version版本比较'
```



之前做法

1. 全查出

```
select * from test_table;
```

![image-20190327224730749](assets/image-all.png)

2. 在内存中排序。

`缺点` ： 没有利用上mysql索引， 筛选出数据不精确。并且浪费内存

`改进目标`： 利用mysql数据表索引

所以在这里，我们需要了解Mysql数据库如何对string类型像数字一样，进行比较操作(>,>=, <, <=)。

字符串比较主要依赖 当前列字段的「字符集」和「字符序」

1. 字符集(character set)： 定义了字符以及字符的编码, 比如utf8,unicode。
2. 字符序(collation): 定义了字符的比较规则。 比如字符 a， b的比较规则

字符串从左到右依次比较对应字符， 根据字符序规则， 确认比较结果

对于版本比较， 则需要我们定长

```
 比如限定长度为 xxxx.xxxx.xxxx.xxxx,
```

 由4段长度为4位数字的段通过 点拼接而成。 这样的目的是为了准确匹配两个版本号， 调整数据表结构


调整完后， 我们数据表内容如下：

查询SQL如下：

```mysql
select * from test_table where '0009.0002.1111' >= start_version  and '0009.0002.0001' <= end_version 
```



数据表内容如下：

![image-20190327230026038](assets/update.png)



#### 其他改进方法

利用IP：

假设 我们的版本号， 每一段值都<=255, 则我们可以使用IP比较方法



```
select inet_aton('192.168.1.1')

```



我们数据表内容如下



![image-20190327230644909](assets/image-string-ip.png)





参考资料：

1. https://stackoverflow.com/questions/26080187/sql-string-comparison-greater-than-and-less-than-operators
2. https://www.cnblogs.com/chyingp/p/mysql-character-set-collation.html




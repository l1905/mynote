## PHP查询Mysql大数据问题解决

某些情况下，我们查询mysql中表的数据太大， 可能会导致PHP端内存溢出

比如

```
select * from user_info

#查询结果有30万条数据
```
这个会直接导致我们的php内存溢出。

### 解决办法

PHP查询mysql分为两种

1. Buffered queries 默认情况

请求结果直接返回给php

2. Unbuffered queries 

客户端请求查询后， 会把查询默认缓存在mysql的server 端， 然后php再慢慢获取

例如

```
<?php
$mysqli  = new mysqli("localhost", "my_user", "my_password", "world");
$uresult = $mysqli->query("SELECT Name FROM City", MYSQLI_USE_RESULT);

if ($uresult) {
   while ($row = $uresult->fetch_assoc()) {
       echo $row['Name'] . PHP_EOL;
   }
}
$uresult->close();
?>
```


查询mysql文档， 主要是是受`mysql_use_result`控制，

mysql官网参考
https://dev.mysql.com/doc/refman/8.0/en/mysql-use-result.html


PHP文档地址
http://php.net/manual/zh/mysqlinfo.concepts.buffering.php


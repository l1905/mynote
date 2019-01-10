dns服务器解析相关


####  dns解析

一句话简介
通过[域名]获取对应服务器访问IP

假设，我们本地请求百度 www.baidu.com
请求伪代码流程

```
server_ip = blackFunc(domain)
{   
    //本地dns缓存
    if(system_cached_match(domain))
    {
        return server_ip
    } else if (system_host_file_match(domain)) //匹配本地host文件
    {
        return server_ip
    } else if (request_namespace_server_by_resolv-conf(domain)) //请求本地 /etc/resolv.conf 指定的域名服务器
    {
        return server_ip
    } else {
        //错误
    }
    //请求namespace server 
    //处理逻辑    
}
```

上面伪代码引入了namespcae_server的概念， 即根服务器，顶级服务器， 这里不想详细说， 具体看参考资料

#### 常见域名

[可选三级域名].主机名.次级域名.顶级域名.根域名

比如
www.smzdm.com.

根域名 .
顶级域名 com
次顶级域名 smzdm
可选三级域名 www

类似证书签发， 依次从根节点签发, 层层授权， 上层污染，下层全部受影响


常用debug命令 , 本地dns服务器完成递归操作
dig www.baidu.com

指定dns服务器
dig @8.8.8.8 www.baidu.com

从根服务器递归展现过程
dig +trace www.baidu.com

host使用
host -v www.smzdm.com

#### 网站常用dns服务器配置参数

A记录：地址记录，用来指定域名的IPv4地址（如：8.8.8.8），如果需要将域名指向一个IP地址，就需要添加A记录。
重点 A记录指向IP

CNAME： 如果需要将域名指向另一个域名，再由另一个域名提供ip地址，就需要添加CNAME记录。
重点 cname指向域名

TXT：在这里可以填写任何东西，长度限制255。绝大多数的TXT记录是用来做SPF记录（反垃圾邮件）。

NS：域名服务器记录，如果需要把子域名交给其他DNS服务商解析，就需要添加NS记录。
域名服务器

AAAA：用来指定主机名（或域名）对应的IPv6地址（例如：ff06:0:0:0:0:0:0:c3）记录。
IPv6的IP地址

MX：如果需要设置邮箱，让邮箱能收到邮件，就需要添加MX记录。
邮箱

显性URL：从一个地址301重定向到另一个地址的时候，就需要添加显性URL记录（注：DNSPod目前只支持301重定向）。

非dns主责

隐性URL：类似于显性URL，区别在于隐性URL不会改变地址栏中的域名。
非dns主责

SRV：记录了哪台计算机提供了哪个服务。格式为：服务的名字、点、协议的类型，例如：_xmpp-server._tcp。

SOA： 描述信息 https://my.oschina.net/u/1382972/blog/340036


时间操作
1.申请免费域名：my.freenom.com
2. 申请dns解析: dnspod

概念：
dns根服务器文件： The Root Hints Data File， 目前没有找到根服务器的 域名和ip的映射文件， 理论上是需要存在的

todo: 实践搭建dns服务器

参考文件
http://www.unixfu.ch/how-do-i-update-the-root-hints-data-file-for-bind-named-server/
https://zhuanlan.zhihu.com/p/31568450
http://www.ruanyifeng.com/blog/2016/06/dns.html

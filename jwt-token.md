## jwt理解

### 一句话理解：

生成加密token， 无状态， 无需服务端维持状态


### 三部分组成

Header（头部）
Payload（负载）
Signature（签名）

header 
指明加密使用那种规则，是sha256,还是其他
base64简单encode

payload 指明需要加密的内容, 里面含有用户信息，base64简单encode

Signature hash加密后的密文内容

jwt token 具体结构

token = Header（头部） + “.” + Payload（负载）+ “.” + Signature

服务端 做校验:
Signature === encryptionFunction(Payload, secret)


### 核心关键点：


1. secret 不能泄漏， 不能写在代码层面中， 只能线上数据库存一分,否则对外公开拿到加密算法

2. 无状态， 无法强制token退出， payload中有效时间字段，可以设置短一些，比如30分钟，可以有效减少盗用

3. 必须https加密， 劫持后， 被恶意获取token 会造成信息泄漏

4. palyload中不能放敏感信息，必须要再次加密

5. 对token整体可以二次加密(采用对称加密算法， 比如AES, DES)



### 参考链接:

https://jwt.io/#debugger

https://zhuanlan.zhihu.com/p/27370773

https://tools.ietf.org/html/rfc7519

http://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html

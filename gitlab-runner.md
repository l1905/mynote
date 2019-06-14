## 搭建gitlab CI工具runner

#### 1. 安装runner(运维操作)
##### 1.1. 下载runner工具

核心点，版本需要跟gitlab匹配， 否则会提示 404 not found

公司内部gitlab依赖的runner版本是1.11.2, 因此需要下载指定版本

方法1

```
> yum install gitlab-ci-multi-runner-1.11.2-1
```
方法2
自己手动二进制文件

```
> wget https://gitlab-ci-multi-runner-downloads.s3.amazonaws.com/v1.11.2/index.html

> cp gitlab-ci-multi-runner-linux-amd64 /usr/local/bin/gitlab-runner

> sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash

> sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner

> sudo gitlab-runner start
    
```

查看状态

```
> gitlab-ci-multi-runner status
```

总结: <em>下载runner版本必须跟gitlab保持一致， 在这里耽搁时间比较长</em>


#### 2. 注册runner(开发操作)
 
 1. 进入项目的 设置/runner目录 ， 获取地址 和token， 用于下面步骤用
 2. runner服务器执行注册命令
 
 ```
 gitlab-ci-multi-runner register
 ```
 输入URL， TOKEN， 标签等，选择 最后选择shell运行
 
```
 [root@10-9-136-206 ~]# gitlab-runner register
Running in system-mode.

Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
https://gitlab-team.smzdm.com/ci
Please enter the gitlab-ci token for this runner:
obWynEujvbGrBkUCHgN9
Please enter the gitlab-ci description for this runner:
[10-9-136-206]: 206
Please enter the gitlab-ci tags for this runner (comma separated):
158
Whether to run untagged builds [true/false]:  这里要选择true，否则会识别出未打标签的构建

[false]: true
Registering runner... succeeded                     runner=obWynEuj
Please enter the executor: ssh, docker-ssh+machine, kubernetes, docker-ssh, parallels, virtualbox, docker+machine, docker, shell:
shell
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!

```
 
 
 3. 验证注册成功
 
 ```
 > gitlab-ci-multi-runner list
 ```
 
  从gitlab刷新获取到runner信息， 
 

#### 3.常用命令：
1. 从gitlab移除不用的runner， 需要runner服务器重新注册

```
> gitlab-ci-multi-runner verify --delete
> gitlab-ci-multi-runner register
```


启动runner

```
systemctl start gitlab-runner
```

完成gitlab的runner安装

其他参考链接

1. 必须指定用户install, https://docs.gitlab.com/runner/install/linux-manually.html
2. run start install 关系 https://www.cnblogs.com/jiukun/p/7481287.html
3. https://gitlab.com/gitlab-org/gitlab-runner/issues/2164


目前158上runner测试环境为：10-9-136-206






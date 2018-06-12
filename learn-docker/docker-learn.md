# docker学习记录

主要参考官方网文档， 简单记录内容使用

docker基本概念： 镜像image, 容器container，不同容器共用底层image + docker底层

最终目标： 在同一台机器上， 实现资源的隔离 + 镜像共享

## 安装相关

mac平台，下载安装即可

需要注意的是，因为墙，国内无法正常访问docker源， 因此需要修改docker源 为国内源

这里用到的是: https://www.daocloud.io/mirror#accelerator-doc


## container容器命令相关(常用)

> docker container COMMAND

查看帮助信息 docker container --help

#### 列出所有容器(正启动，未有退出
docker container 或者 docker ps

#### 列出已经停止的所有容器
docker container -a

#### 启动容器

docker container start [OPTIONS] CONTAINER [CONTAINER...]

#### 停止容器

docker container stop [OPTIONS] CONTAINER [CONTAINER...]

#### 删除容器
docker container rm [OPTIONS] CONTAINER [CONTAINER...]

#### 新起容器，执行命令

docker container run [OPTIONS] IMAGE [COMMAND] [ARG...]

镜像获取优先级 --->local--->remote source

关键参数OPTION 

-i 交互模式

-t 新起putty命令行

-d backgroup执行

-v 宿主机 文件夹映射

-p 端口映射， 将container内部端口映射到外部

--read-only Mount the container's root filesystem as read only 理解为宿主机根目录为只读

####在正在执行的容器中 执行命令
docker container exec [OPTIONS] CONTAINER COMMAND [ARG...]

Run a command in a running container

docker container rm [OPTIONS] CONTAINER [CONTAINER...]

#### 查看镜像详细信息 

docker container inspect

#### 查看资源占用(cpu, 内存， IO)

docker stats

## image 相关(镜像)

#### 搜索镜像

docker search ubuntu (从Docker Hub中搜索)

#### 搜索镜像并且需要携带tags 版本号 

官方没有自带， 需要写脚本完成， 参考stackoverflow，封装shell脚本完成

https://stackoverflow.com/questions/28320134/how-to-list-all-tags-for-a-docker-image-on-a-remote-registry

参考调用方式：dockertags ubuntu

#### 从源获取image

docker pull image

#### 根据dockfile 构件image

docke build

Build an image from a Dockerfile

Usage:	docker build [OPTIONS] PATH | URL | -

#### 删除镜像image

docker image rm 


#### 根据image查看dockerfile历史

docker history [OPTIONS] IMAGE

docker history --no-trunc IMAGE


## dockfile解读 TODO

dockerfile 用于制作image镜像，

常使用方式为 docker build -t name:tag .

一般使用格式

```
# Comment(备注)
INSTRUCTION(指令) arguments(参数)
```
指令不区分大小写， 但一般用大写 用于区分参数

**dockerfile**文件必须以`FROM`指令开头

\#号参数 在新一行 行首时表达备注， 其他地方没有此含义

#### 解析指令(parser directives)

只能出现在最前面， 只支持 escape 指令

#### 环境变量 
支持两种格式 `$variable_name` or `${variable_name}`

在整个指令行中只使用一个值，参考下面这个例子：

```
ENV abc=hello
ENV abc=bye def=$abc
ENV ghi=$abc
```

最后abc=bye, def=hello, ghi=bye

#### FROM

继承某镜像

```
FROM <image> [AS <name>]

FROM <image>[:<tag>] [AS <name>]

FROM <image>[@<digest>] [AS <name>]

```
`ARG`是唯一用于 `FROM` 的指令， 字面意义即是 参数

#### RUN

```
RUN <command> (shell form, the command is run in a shell, which by default is /bin/sh -c on Linux or cmd /S /C on Windows)

RUN ["executable", "param1", "param2"] (exec form)

```

`\`用于换行

```
RUN /bin/bash -c 'source $HOME/.bashrc; \
echo $HOME'

RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'

RUN ["/bin/bash", "-c", "echo hello"]

```

#### CMD

`CMD` 指令有以下三种形式

```
CMD ["executable","param1","param2"] (exec form, this is the preferred form)
CMD ["param1","param2"] (as default parameters to ENTRYPOINT)
CMD command param1 param2 (shell form)
```
指定容器运行时的默认参数
在dockfile文件中， 只能有一个`CMD`指令， 如果有多个`CMD`指令， 则只有最后一个`CMD`指令会运行

RUN指令， 并且commit提交为新镜像

CMD指令 构建时，没有操作， 只有执行container时，才会有操作

命令行传CMD会覆盖

#### EXPOSE 端口映射

```
EXPOSE <port> [<port>/<protocol>...]

docker run -p  可以覆盖
docker run -p 80:80/tcp -p 80:80/udp ...
```

#### ADD 

`ADD`指令 copy文件，目录，远程URL内容到目的路径

```
ADD [--chown=<user>:<group>] <src>... <dest>
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"] (this form is required for paths containing whitespace)
```

#### COPY

不支持远程复制， 其他和ADD功能没明显区别

#### ENTRYPOINT

```
ENTRYPOINT ["executable", "param1", "param2"] (exec form, preferred)
ENTRYPOINT command param1 param2 (shell form)
```

字面意义为进入点， 和CMD类似

详细解读： https://www.cnblogs.com/sparkdev/p/8461576.html
#### VOLUME 数据卷

-v参数适用

--mount和 -v结果一致 

```
$ docker run -d \
  --name devtest \
  --mount source=myvol2,target=/app \
  nginx:latest
  
$ docker run -d \
  --name devtest \
  -v myvol2:/app \
  nginx:latest
```

调用方式： 

```
$ docker run -d \
  --name=nginxtest \
  -v nginx-vol:/usr/share/nginx/html:ro \ 只读
  nginx:latest
```
	
```
$ docker run -d \
  --name=nginxtest \
  --mount source=nginx-vol,destination=/usr/share/nginx/html,readonly \
  nginx:latest
```

docker inspect nginxtest 查看具体信息



#### USER 用户相关

#### WORKDIR 工作目录


## docker-compose相关 TODO


## 搭建 PHP + NGINX环境

#### 先搭建nginx

0. 先理解 nginx镜像生成时， 制作者将命令已经打包， 比如 启动容器时， 会自动启动nginx , 不需要用户手动再次启动  验证方式， 通过 container inspect 查看

1. 继续通过container inspect 查看nginx 配置文件+access.log+error.log路径 

3. 因为nginx配置文件， error.log, access.log是用户经常修改使用， 因此需要将此三个文件映射到 宿主目录中， 便于修改查看

4. 将内部端口映射到宿主 

5. 将nginx root目录映射到宿主目录

```
docker rm -f "nginx-server" &> /dev/null
docker container  run --name nginx-server -d  \
    -p 81:80 \
    -v /Users/xxxx/codesrc/docker/nginx/access.log:/var/log/nginx/access.log \
    -v /Users/xxxx/codesrc/docker/nginx/error.log:/var/log/nginx/error.log \
    -v /Users/xxxx/codesrc/docker/nginx/nginx.conf:/etc/nginx/nginx.conf \
    -v /Users/xxxx/codesrc/docker/nginx/php:/webroot \
    nginx:latest
```
将内部容器中nginx.conf 复制到 宿主目录中

> docker cp nginx-server:/etc/nginx/conf.d/default.conf default1.conf

这里面关键要点 需要将nginx.conf中 fastcgi_pass 参数 指向**外网IP**地址， 同目录给出相应的nginx.conf

#### PHP-FPM搭建同理

```
docker rm -f "php-fpm" &> /dev/null
    docker run --name php-fpm -d -p 9000:9000 \
        -v /Users/xxxx/codesrc/docker/nginx/php:/webroot \
        php:7.2.6-fpm
```


## 资源隔离相关(TODO)

自己理解docker将资源隔离， 可以快速扩容， 应该就是云主机的操作方式。

自己在云主机上的买的内存应该是 可以用到多大， 但平时自己用不到， 云主机所容

原理参考左耳朵耗子文章 
https://coolshell.cn/tag/docker

其他参考文章

https://github.com/qianlei90/Blog/issues/35

https://docs.docker.com/engine/reference/builder/#format

https://www.cnblogs.com/sparkdev/p/8461576.html

http://www.cnblogs.com/ilinuxer/p/6613904.html














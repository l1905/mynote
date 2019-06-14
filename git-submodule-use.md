## git submodule使用

### 使用场景

在仓库代码中，将某一文件夹， 映射为另一仓库代码代码

在我们公司的使用场景是： 测试环境前端打包出来的hash值 和线上环境hash值不一致， 并且前端希望将hash值放到前端代码仓库中来管理。



## 使用流程

- 我们在github上新建测试代码仓库:

 
  `submodule-project`仓库: https://github.com/l1905/submodule-project 主要用于子模块
  
  `git-main`仓库：https://github.com/l1905/git-main，主要用于我们的主代码模块



- 在`git-main`项目中， 通过`git module`命令，引入子模块

  
  ```git submodule add https://github.com/l1905/submodule-project submodule-project```
  
  会在最顶层目录， 新增`.gitmodules`配置文件
  
  ```
  ➜  git-main git:(master) cat .gitmodules
[submodule "submodule-project"]
	path = submodule-project  #对应文件夹名称
	url = https://github.com/l1905/submodule-project #对应子模块URL地址
  ```
  
- 提交目录

   ```
   git commit -am 'added submodule-project module'
   ```
- 合作者首次拉取clone项目代码

   ```
   git clone --recursive https://github.com/l1905/git-main
   ```  
- 拉取上游修改

  如果submodule-project有新的提交， `git-main`项目需要拉取submodule-project最新修改， 并作提交操作
  
  ```
  git submodule update
  根据子模块hash, 拉取子模块对应代码提交
  
  
  git submodule update --remote
  等同于：
  git fetch
  git merge origin/master
  
  先拉取代码， 后将原称分支代码合并到 当前hash分支
  
  ```
  ➜  git-main git:(master) git submodule update --remote
remote: Enumerating objects: 5, done.
remote: Counting objects: 100% (5/5), done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 3 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), done.
From https://github.com/l1905/submodule-project
   8f3346a..8fa0019  master     -> origin/master
Submodule path 'submodule-project': checked out '8fa00190b408491736623f6d060598de62f70d1d'
➜  git-main git:(master) ✗ git st
On branch master
Your branch is up-to-date with 'origin/master'.
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)
  ```
	
  然后再做提交操作
  
  ```
  ➜  git-main git:(master) ✗ git add submodule-project
➜  git-main git:(master) ✗ git ci -am 'update-submodule'
[master fc7854f] update-submodule
 1 file changed, 1 insertion(+), 1 deletion(-)
➜  git-main git:(master) git pull
Already up-to-date.
➜  git-main git:(master) git push
Counting objects: 2, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (2/2), 237 bytes | 0 bytes/s, done.
Total 2 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
To https://github.com/l1905/git-main
   dac1533..fc7854f  master -> master
  ```
  
- `submodule-project`引入持续集成, 当有更新时， 自动在`git-main`项目作提交操作

在submodule-project项目中引入.travis.yml文件

```
language: php
php:
  - 7.1.9
env:
  global:
  - GIT_USER="l1905"
  - EMAIL="litonglitong@hotmail.com"
  - REPO="git-main"
  - FILES="foo.txt"
  - GH_REPO="github.com/${GIT_USER}/${REPO}.git"
script:
  - echo "hello world submodule" 
after_success:
  - git clone --recursive git://${GH_REPO}
  - cd ${REPO}
  - git remote
  - git submodule update --remote
  - git config user.email ${EMAIL}
  - git config user.name ${GIT_USER}
  - git add submodule-project
  - git commit -m "update submodule-project hash"
  - git push "https://${TRAVIS_SECURE_TOKEN_NAME}@${GH_REPO}" master > /dev/null 2>&1

```

线上更新脚本 增加如下如下命令：

```
/usr/local/bin/git submodule init
/usr/local/bin/git submodule update
```

### 其他

1. travis 需要去官网授权访问仓库代码
2. travis 可以平台设置加密环境变量， 主要为了避免敏感信息写在代码中泄漏， 设置时自动加密， 获取时自动解密


### 参考文章：

1. https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97
2. https://stackoverflow.com/questions/1777854/how-can-i-specify-a-branch-tag-when-adding-a-git-submodule

  
  
  

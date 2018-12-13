
### svn项目 迁移到git项目(内部gitlab) 

#### 1. ubuntu环境安装：

```shell
sudo apt-get install git-core git-svn ruby
```

安装ruby的迁移脚本

```shell
sudo gem install svn2git
```

如果比较慢， 需要替换gem源为国内ruby-china， 替换教程 ：https://gems.ruby-china.com/

####2. svn需要手动从test分支 新切分支到branches/test， 因为迁移脚本只会迁移 branches, trunk两个目录， 其他均忽略

主要是自己项目目录单独维护一份test分支，需要对此分支进行迁移

### 3. 迁移细节

新建git项目目录，比如

```shell
mkdir git_demo

cd git_demo
```

需要对svn， git项目 log用户名做对应映射

首先要查看svn项目中都有哪些人提交过代码

```shell
svn log --quiet | grep -E "r[0-9]+ \| .+ \|" | cut -d'|' -f2 | sed 's/ //g' | sort | uniq
```

基本格式

svn用户名 = git用户名 <用户所属邮箱@self>

例如

```
litongxue = litongxue <litongxue@self.com>

wangtongxue = wangtongxue <wangtongxue@self.com>
```
编辑命名为 authors.txt

执行迁移命令

```shell
svn2git https://svn.team.bq.com/www/src/ --authors ../authors.txt
```


{https://svn.team.bq.com/www/src/} 需要替换为自己对应的svn项目
####4. 验证是否迁移成功

git的master分支对应svn项目的trunk分支
git中的其他分支对应svn项目branches目录中的其他分支

从git中列出所有分支

```shell
git branch
```

验证提交记录已经迁移过来

```shell
 git log
```

#### 5. 将本地git项目推到远端gitlab中

把远端gitlab加入

```shell
git remote add origin https://gitlab-team.self.com/self/demo.git
git push --all origin
git push --tags origin
```

#### 6. 登录gitlab 查看是否迁移成功




#### 迁移总结

1. svn2git安装依赖 ``git svn``命令
2. 迁移只支持trunk分支, branchs目录， 其他定制分支不支持用svn2git迁移


#### 参考链接

https://github.com/nirvdrum/svn2git






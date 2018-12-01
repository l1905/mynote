Travis提供功能的定义：

提供一个「云环境」(travis服务器)，在「特定场景事件」下 执行「构建文件」，得出「代码构建状态」(成功／失败)

#### 云环境

travis提供， 我理解是travis提供独立virtual machine, 为执行构建文件 提供一个简单的环境， 可以理解成ubuntu, centos等环境

#### 特定场景事件
 1. github 代码仓库有代码提交， 有合并pr，等触发式性事件
 2. 定期(每天，每周，每月) 定期执行「构建文件」类似 cronjob 

 
#### 构建文件
即项目跟目录的.travis.yaml文件，描述构建过程， 预装环境依赖，定义钩子，部署项目
预定义钩子阶段， 从上到下依次执行：
    
1. OPTIONAL Install apt addons
2. OPTIONAL Install cache components
3. before_install
4. install
5. before_script
6. script
7. OPTIONAL before_cache (for cleaning up cache)
8. after_success or after_failure
9. OPTIONAL before_deploy
10. OPTIONAL deploy
11. OPTIONAL after_deploy
12. after_script
    
#### 代码构建状态
根据script阶段运行状态， 判断最终执行状态

#### 思考：
1. 可以持续集成，检测到项目代码提交，即做些测试构建流程
2. 利用github+travis 可以实现一些定时任务，并且不依赖项目代码


#### 参考文档：

1. http://www.ruanyifeng.com/blog/2017/12/travis_ci_tutorial.html
 
2. https://docs.travis-ci.com/user/for-beginners/

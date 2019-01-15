# composer-demo

之前同事有分享过composer的使用， 但是没有完全实践， 有些地方理解不是太深刻， 现在从新实践理解



### 版本控制



为了解决各个开源软件中间对特定版本的依赖， 业界有现成的规范 https://semver.org/， 大部分开源软件都已该规范为准。



下面是对该规范的理解： 

版本主要是以下形式



>  MAJOR.MINOR.PATCH



约束条件： 不允许以0为前缀， 不允许为负值



比如 10.1.2



> 标记版本号的软件发行后，禁止（MUST NOT）改变该版本软件的内容。任何修改都必须（MUST）以新版本发行。

已经发布的版本，就是泼出去的水， 不能做修改内容， 如果希望做新的修改，必须发布新的内容。



我们详细解释下

MAJOR(主版本号)： 对应大版本， 不兼容已经发布的部分API， 比如 yii2 和yii1是两个不兼容的版本，

MINOR(次版本号) 功能更新

PATH(修订号)： 从字面意义就是打补丁， 其实就是为了修复bug



> 主版本号为零（0.y.z）的软件处于开发初始阶段，一切都可能随时被改变。这样的公共 API 不应该被视为稳定版

还没有达到稳定版本， 这样的版本是需要少用到线上环境。



### 下面主要解释通用的发行版本关系



#### Alpha：

lpha是内部测试版,一般不向外部发布,会有很多Bug.除非你也是测试人员,否则不建议使用.是希腊字母的第一位,表示最初级的版本，alpha 就是α，beta 就是β ，alpha 版就是比

beta还早的测试版，一般都是内部测试的版本。

#### Beta:

该版本相对于α版已有了很大的改进，消除了严重的错误，但还是存在着一缺陷，需要经过多次测试来进一步消除。这个阶段的版本会一直加入新的功能。

#### RC：(Release Candidate):

Candidate是候选人的意思，用在软件上就是候选版本。Release.Candidate.就是发行候选版本。和Beta版最大的差别在于Beta阶段会一直加入新的功能，但是到了RC版本，几乎

就不会加入新的功能了，而主要着重于除错!  RC版本是最终发放给用户的最接近正式版的版本，发行后改正bug就是正式版了，就是正式版之前的最后一个测试版。

#### GA：（general availability）

比如：Apache Struts 2 GA这是Apache Struts 2首次发行稳定的版本，GA意味着General Availability，也就是官方开始推荐广泛使用了。

#### Release:

该版本意味“最终版本”，在前面版本的一系列测试版之后，终归会有一个正式版本，是最终交付用户使用的一个版本。该版本有时也称为标准版。一般情况下，Release不会以单词形式出现在软件封面上，取而代之的是符号(R)。



> 先行版本号可以（MAY）被标注在修订版之后，先加上一个连接号再加上一连串以句点分隔的标识符来修饰。标识符必须（MUST）由 ASCII 字母数字和连接号 [0-9A-Za-z-] 组成，且禁止（MUST NOT）留白。数字型的标识符禁止（MUST NOT）在前方补零。先行版的优先级低于相关联的标准版本。被标上先行版本号则表示这个版本并非稳定而且可能无法满足预期的兼容性需求。范例：1.0.0-alpha、1.0.0-alpha.1、1.0.0-0.3.7、1.0.0-x.7.z.92。



#### 版本关系

同级别下 alpha<beta<rc 预发布版本号

1.0.0-alpha < 1.0.0-alpha.1 < 1.0.0-alpha.beta < 1.0.0-beta < 1.0.0-beta.2 < 1.0.0-beta.11 < 1.0.0-rc.1 < 1.0.0



> 版本编译元数据可以（MAY）被标注在修订版或先行版本号之后，先加上一个加号再加上一连串以句点分隔的标识符来修饰。标识符必须（MUST）由 ASCII 字母数字和连接号 [0-9A-Za-z-] 组成，且禁止（MUST NOT）留白。当判断版本的优先层级时，版本编译元数据可（SHOULD）被忽略。因此当两个版本只有在版本编译元数据有差别时，属于相同的优先层级。范例：1.0.0-alpha+001、1.0.0+20130313144700、1.0.0-beta+exp.sha.5114f85。



现行版本号也可以是以时间戳+编译数据为后缀



composer中可以通过 minimum-stability， 指定最小依赖稳定性 ， 中 有以下可选项：

 `dev`,< `alpha`<, `beta`<, `RC`<,  `stable`

minimum-stability默认是stable, 会忽略 dev, alpha, beta, rc等版本， 这个我会实践验证



### Tilde Version Range (~)



- ~4.1.3 means >=4.1.3,<4.2.0, 

- ~4.1 means >=4.1.0,<5.0.0 (most used),， 

- ~0.4 means >=0.4.0,<1.0.0,

- ~4 means >=4.0.0,<5.0.0.

  

和下面这个容易混： 一般情况下选择下面这个^



### Caret Version Range (^)



- ^4.1.3 (most used) means >=4.1.3,<5.0.0, 
- ^4.1 means >=4.1.0,<5.0.0, same as ~4.1 but:
- ^0.4 means >=0.4.0,<0.5.0, this is different from ~0.4 and is more useful for defining backwards compatible version ranges.
- ^4 means >=4.0.0,<5.0.0 which is the same as ~4 and 4.*.



常见的限制关系如下：

| Constraint     | Internally                   |
| -------------- | ---------------------------- |
| `1.2.3`        | `=1.2.3.0-stable`            |
| `>1.2`         | `>1.2.0.0-stable`            |
| `>=1.2`        | `>=1.2.0.0-dev`              |
| `>=1.2-stable` | `>=1.2.0.0-stable`           |
| `<1.3`         | `<1.3.0.0-dev`               |
| `<=1.3`        | `<=1.3.0.0-stable`           |
| `1 - 2`        | `>=1.0.0.0-dev <3.0.0.0-dev` |
| `~1.3`         | `>=1.3.0.0-dev <2.0.0.0-dev` |
| `1.4.*`        | `>=1.4.0.0-dev <1.5.0.0-dev` |



#### composer中install update require使用场景

- `composer install` - 如有 composer.lock 文件，直接安装，否则从 composer.json 安装最新扩展包和依赖；
- `composer update` - 从 composer.json 安装最新扩展包和依赖；
- `composer update vendor/package` - 从 composer.json 或者对应包的配置，并更新到最新；
- `composer require new/package` - 添加安装 `new/package`, 可以指定版本，如： composer require new/package ~2.5.

#### 参考文章：

1. https://laravel-china.org/index.php/docs/composer/2018/versions/2098
2. 语义约定：https://semver.org/lang/zh-CN/
3. composer官网： https://getcomposer.org/doc/articles/versions.md#tilde-version-range-
























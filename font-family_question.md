# 浏览器选择字体类型问题记录

## 背景

公司有个需求，

需要在客户端内嵌的h5页面里， 将当前页面内容以截图的形式进行分享。

所以涉及到截图。 目前我们采用后端截图方式。

即：

前端通过调用后台node服务(使用`puppeteer`包)， 实现后端截图-->上传到图床-->返回图片地址。


目前遇到的问题是， node服务所在的服务器截图后， 图片中内容字体和设计师要求的不符。

所以有必要了解一下浏览器加载字体的选择


## 规则理解

不同操作系统会内置自己的字体格式

一套字体会有对应的加载规则

常见css设置方式

```
font-family: Georgia, "Times New Roman", 
             "Microsoft YaHei", "微软雅黑", 
             STXihei, "华文细黑", 
             serif;
```

有以下三种加载规则

```
（1）优先使用排在前面的字体。

（2）如果找不到该种字体，或者该种字体不包括所要渲染的文字，则使用下一种字体。

（3）如果所列出的字体，都无法满足需要，则让操作系统自行决定使用哪种字体。

根据这些规则，font-family应该优先指定英文字体，然后再指定中文字体。否则，中文字体所包含的英文字母，会取代英文字体，这往往很丑陋。
```

即因为中文字体一般有英文字母， 他可以解析英文字母，
但是英文字体一般没有中文字母。

可以在命令行执行以下命令，查看页面加载顺序

```
window.getComputedStyle(document.body)['font-family'];

""Helvetica Neue", Helvetica, "PingFang SC", "Hiragino Sans GB", "Microsoft YaHei", "Noto Sans CJK SC", "WenQuanYi Micro Hei", Arial, sans-serif"
```

但是系统字体中如果没有上述指定要求的字体， 则系统决定使用那种字体， 
这里系统有一套加载规则， 不详细论述了

## 参考资料

1. [中文字体网页开发指南](http://www.ruanyifeng.com/blog/2014/07/chinese_fonts.html)






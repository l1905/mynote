# wp系列学习之-hook函数

wp作为世界上使用最广泛的php cms系统，为了更好理解wp的设计精髓， 我分几个章节分别介绍其核心模块。 

首先我们学习hook插件hook钩子原理


## hook钩子简介
hook钩子： 即先在某些逻辑代码中，占住位置点， 恶俗点即是占坑， 然后在程序代码的其他地方，定义程序逻辑方法， 并且把这些逻辑方法 统一挂钩到

我们前面站的`坑`中， 当逻辑代码执行到 `占坑`位置时， 依次执行我们挂钩的方法。
 

程序设计中， 开发者会广泛使用hook原理， 
单元测试中`测试前`,`测试中`，`测试后`这些也即是 hook点， 通常CI持续继承中， 也会区分出 `before_script`, `after_script`等这些hook点。


## wp中钩子定义位置

```shell
wp-includes/plugin.php # 主要对系统提供全局方法，例如xxxx_action, xxxx_filter
wp-includes/class-wp-hook.php #将hook点以迭代器类形式，作为底层存储

```

我们代码中，会经常使用

`add_action`, `add_filter` 等方法， 这些方法即将我们指定的方法， 放到坑位中， 比如

```php
add_filter('woocommerce_payment_gateways', 'add_omoney_gateway_class');
function add_omoney_gateway_class($methods)
{
    $methods[] = 'WC_Gateway_Omoney';
    return $methods;
}
```

在上述代码中， 我们将`add_omoney_gateway_class`方法追加到 `woocommerce_payment_gateways`坑位中。


wp中， 有两种形式的hook形式，

1. xxxx_action:  依次执行坑位中 我们预先定义的方法
2. xxxx_filter:  依次执行坑位中， 我们预先定义的方法， `但是可以获取返方法回值`


我们平时使用这两种hook时， 如果需要对方法返回值进行操作， 那么我们使用 `xxxx_filter`， 其他情况使用`xxxx_action`即可。


一般我们是添加方法到占坑位， 即 `add_action`, `add_filter`, 因为大部分时候系统已经帮我们在系统执行逻辑中，做好坑位执行逻辑了。
 
 
 以上方法支持添加优先级， 比如在坑位中， A方法先执行， B方法后执行， 有`priority`属性
 
 
 
 系统中执行占坑位的方法有以下两种
 
 
 1. do_action:
 2. apply_filters:
 
 
 
 如果在filter方法中， 我们可以查看到当前执行的`filter`名称， 通过 `current_filter`获取当前执行的filter名称
 
 
 
 
 
 ## 参考资料
 
 1. https://www.toolmao.com/wordpress-hook-plugins-analysis

 











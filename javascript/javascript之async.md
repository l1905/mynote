## javascript之Promises, async/await 核心概念理解

### 1. 回调(callbacks)调用方式

#### 基本应用

js中大多以事件回调方式来实现异步调用。 这里我们要实现“异步加载js文件“ 功能。

```
//定义加载脚本函数。
function loadScript(src) {
  let script = document.createElement('script');
  script.src = src;
  document.head.append(script);
}
```

//这里加载脚本文件是异步方式， 我们需要等脚本文件正式加载完成后， 才能调用脚本文件中的函数。

```
loadScript('1.js');
```
该调用被称为“异步“调用， 因为脚本loading没有被实时，同步加载， 而是稍后异步加载脚本文件。

我们可以验证：

```
loadScript('1.js');
newFunction(); //这个函数来自1.js文件中， 因为文件还没有正式加载， 因此该函数还不存在。
```

对于dom元素， 我们知道在其对象上， 存在onload事件， 该事件是代表dom加载时， 执行该事件， 因此我们可以将回调函数放到该事件中去执行。

```
function loadScript(src, callback) {
  let script = document.createElement('script');
  script.src = src;

  script.onload = () => callback(script);

  document.head.append(script);
}
```

经过调整，我们将函数转变为如下方式执行：

```
loadScript('1.js', function() {
  // the callback runs after the script is loaded
  newFunction(); // so now it works
  ...
});
```

这是回调风格的 异步函数调用。 需要提供callback参数 用来异步回调。


#### 在回调函数中使用回调函数。

比如，我们要实现， 依次加载 1.js, 2.js, 3.js， 即加载顺序为 1.js---->2.js---->3.js

```
loadScript('1.js', function(script) {

  alert(`Cool, the ${script.src} is loaded, let's load one more`);

  loadScript('2.js', function(script) {
    alert(`Cool, the second script is loaded`);

    loadScript('3.js', function(script) {
      alert(`Cool, the second script is loaded`);
    });
  });

});
```

层层嵌套，影响可读性。使得代码不可维护。

### 2. Promise异步调用方式

Promise从语义上来讲 是“承诺” , 主要是为了解决callback存在的问题。

通俗理解

> 你是一位顶尖歌手， 粉丝们不管白天黑夜都想你要唱片。
> 为了摆脱这种纠缠， 你承诺， 一旦唱片制作(比较耗时间)完成， 你就发给粉丝们， 你列出一份歌单，告诉粉丝们它们可以订阅这些变化信息，当唱片完成，它们可以把信息放到邮箱中， 所有的订阅者都可以立刻接收到， 如果出现了一些小问题，比如唱片发版被取消，它们也可以立刻知道。

与观察者模型有些类似。

声明方式

```
let promise = new Promise(function(resolve, reject) {
  // executor (the producing code, "singer")
});
```

当Promise被创建时， executor方法立刻被执行。

Promise对象有以下两个属性：

1. state ---状态字段， 初始值为"pending(正在执行)", 可以被修改为“fulfilled(成功完成)"或者"rejected(失败拒绝)"状态
2. result ---可以设置为任意值, 初始状态为 "undefined"

当executor完成作业， 它应该执行以下函数中的一个,并且只会执行一个：

1. resolve(value) ---表明这个作业执行成功
    1. 设置state为"fullfilled"状态
    2. 设置变量result 为value
    
2. reject(error) --表明遇到error异常

    1. 设置state为"rejected"状态
    2. 设置result为error对象。
    
注意点：executor只能有一个resolve或者reject, promise的状态改变是最终(final)的。

```
let promise = new Promise(function(resolve, reject) {
  resolve("done");

  reject(new Error("…")); // ignored被忽略
  setTimeout(() => resolve("…")); // ignored被忽略
});
```

state和result是内部变量， 我们没办法直接访问.

`.then`主要用来根据state状态情况，对result结果做指定的操作， 即消费异步操作结果：

表达形式为：

```
promise.then(
  function(result) { /* handle a successful result */ },
  function(error) { /* handle an error */ }
);
```

`.then`第一个函数参数：

1. 在promise resoved完成后 执行
2. 接收result结果

`.then`第二个函数参数：

1. 在promise 被rejected完成后，执行
2. 接收error对象

举例说明：

```
let promise = new Promise(function(resolve, reject) {
  setTimeout(() => resolve("done!"), 1000);
});

// resolve runs the first function in .then
promise.then(
  result => alert(result), // shows "done!" after 1 second
  error => alert(error) // doesn't run
);

```

第一个函数将被执行

可以使用catch方法， 例如

```
let promise = new Promise((resolve, reject) => {
  setTimeout(() => reject(new Error("Whoops!")), 1000);
});

// .catch(f) is the same as promise.then(null, f)
promise.catch(alert); 
```

.catch(f) 等价于 .then(null, f)， 前面是简写模式。

### promises chaing (promises链模式)

主要是为了解决callback中 依次依赖调用的问题。

```
new Promise(function(resolve, reject) {

  setTimeout(() => resolve(1), 1000); // (*)

}).then(function(result) { // (**)

  alert(result); // 1
  return result * 2;

}).then(function(result) { // (***)

  alert(result); // 2
  return result * 2;

}).then(function(result) {

  alert(result); // 4
  return result * 2;

});
```
执行结果为：

![](https://javascript.info/article/promise-chaining/promise-then-chain@2x.png)


```
throw new Error("Whoops!"); 等价于

reject(new Error("Whoops!"));

```

throw出错误：

```
new Promise(function(resolve, reject) {
  resolve("ok");
}).then(function(result) {
  throw new Error("Whoops!"); // rejects the promise
}).catch(alert); // Error: Whoops!
```

对于其他的错误抛出：

```
new Promise(function(resolve, reject) {
  resolve("ok");
}).then(function(result) {
  blabla(); // no such function
}).catch(alert); // ReferenceError: blabla is not defined
```

throw cach使用例子

```
// the execution: catch -> catch -> then
new Promise(function(resolve, reject) {

  throw new Error("Whoops!");

}).catch(function(error) { // (*)

  if (error instanceof URIError) {
    // handle it
  } else {
    alert("Can't handle such error");

    throw error; // throwing this or another error jumps to the next catch
  }

}).then(function() {
  /* never runs here */
}).catch(error => { // (**)

  alert(`The unknown error has occurred: ${error}`);
  // don't return anything => execution goes the normal way

});

```

最后被catch捕获。


`.then/catch(handler)`使用总结相关：

1. 如果返回值， 或者没有return, 则新的 promised将会被解析，最接近的reslove handler(.then的第一个参数)将会被调用

2. 如果抛出异常， 则最近的rejection handler 将会执行(.then的第二个参数活着catch)

3. 如果返回一个promise, 则按另一个promise 过程执行。


### Async/await


```
async function f() {
  return 1;
}
```

async关键词：

一个函数， 加上关键词后返回promise对象，

函数前面跟着async 定义为：

自动包裹着被resolved的promise对象值

```
async function f() {
  return 1;
}

f().then(alert); // 1
```

等价于：

```
async function f() {
  return Promise.resolve(1);
}

f().then(alert); // 1
```

async确保函数返回promise, 里面是非promise

#### Await

```
// works only inside async functions
//只作用于 async函数中
let value = await promise;
async function f() {

  let promise = new Promise((resolve, reject) => {
    setTimeout(() => resolve("done!"), 1000)
  });

  let result = await promise; // wait till the promise resolves (*) 一直等待promise的resoved值

  alert(result); // "done!"
}

f();
```

await不消耗cpu资源， 是一种等待操作， 可以让出cpu

异步调用方式 调整为 同步方式调用， 使得代码增加可读性


 

#### 参考文章

1. https://javascript.info/async
2.  https://developers.google.com/web/fundamentals/primers/promises
3. https://medium.com/javascript-scene/master-the-javascript-interview-what-is-a-promise-27fc71e77261
4.  https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/Arrow_functions
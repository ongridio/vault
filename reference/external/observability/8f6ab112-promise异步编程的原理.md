---
title: promise异步编程的原理
source: http://www.cnblogs.com/bdbk/p/5176594.html
kind: external
domain: observability
author: Bdbk
original_date: 2016-02-06
fetched_at: 2026-05-16
bookmark_title: promise异步编程的原理 - bdbk - 博客园
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](http://www.cnblogs.com/bdbk/p/5176594.html)
> 作者：Bdbk
> 原始日期：2016-02-06
> 抓取日期：2026-05-16

# promise异步编程的原理

# promise异步编程的原理

## 一.起源

JavaScript中的异步由来已久，不论是定时函数，事件处理函数还是ajax异步加载都是异步编程的一种形式，我们现在以nodejs中异步读取文件为例来编写一个传统意义的异步函数：

```
var fs = require('fs');
function readJSON(filename,callback){
fs.readFile(filename,'utf8',function(err,res){
if(err){
return callback(err,null);
}
try{
var res = JSON.parse(res);
}catch(ex){
callback(ex)
}
callback(null,res);
});
}
```

如果我们想异步读取一个json文件，它接受2个参数，一个文件名，一个回调函数。文件名必不可少，关键就在这个callback上面了，当我们要执行这个readJSON函数时，要自己构造想要的回调函数，但是在readJSON函数内部传递callback时候不知道他的参数，显然是不友好的。下面在看一种异步编程的代码：

```
fs.readFile('file1.txt','utf8',function(err,res){
fs.readFile('file2.txt','utf8',function(err,res){
fs.readFile('file2.txt','utf8',function(err,res){
console.log(res);
});
});
});
```

这里嵌套了3个异步回调函数，他们的执行时刻都是不可预测的并且这样写代码也不符合普通程序的执行流程。

所以，问题来了，promise提供了一个解决上述问题的模式。

## 二.定义

promise是对异步编程的一种抽象。它是一个代理对象，代表一个必须进行异步处理的函数返回的值或抛出的异常。也就是说promise对象代表了一个异步操作，可以将异步对象和回调函数脱离开来，通过then方法在这个异步操作上面绑定回调函数。

下面介绍具体API，这里遵循的是commonJS promise/A+规范。

### 1.状态

promise有3种状态：pending（待解决，这也是初始化状态），fulfilled（完成），rejected（拒绝）。

### 2.接口

promise唯一接口then方法，它需要2个参数，分别是resolveHandler和rejectedHandler。并且返回一个promise对象来支持链式调用。

promise的构造函数接受一个函数参数,参数形式是固定的异步任务，举一个栗子：

```
function sendXHR(resolve, reject){
var xhr = new XMLHttpRequest();
xhr.open('get', 'QueryUser', true);
xhr.onload = function(){
if((xhr.status >= 200 && xhr.status < 300) || xhr.status === 304){
resolve(xhr.responseText);
}else{
reject(new Error(xhr.statusText));
}
};
xhr.onerror = function(){
reject(new Error(xhr.statusText));
}
xhr.send(null)
}
```

## 三.实现

要实现promise对象，首先要考虑几个问题：

1.promise构造函数中要实现异步对象状态和回调函数的剥离，并且分离之后能够还能使回调函数正常执行

2.如何实现链式调用并且管理状态

首先是构造函数：

//全局宏定义 var PENDING = 0; var FULFILLED = 1; var REJECTED = 2; //Promise构造函数 function Promise(fn){ var self = this; self.state = PENDING;//初始化状态 self.value = null;//存储异步结果的对象变量 self.handlers = [];//存储回调函数，这里没保存失败回调函数，因为这是一个dome //异步任务成功后处理，这不是回调函数 function fulfill(result){ if(self.state === PENDING){ self.state = FULFILLED;

self.value = result; for(var i=0;i<self.handlers.length;i++){ self.handlers[i](result); } } } //异步任务失败后的处理， function reject(err){ if(self.state === PENDING){ self.state = REJECTED; self.value = err; } } fn&&fn(fulfill,reject); };

构造函数接受一个异步函数，并且执行这个异步函数，修改promise对象的状态和结果。

回调函数方法then：

//使用then方法添加回调函数，把这次回调函数return的结果当做return的promise的resolve的参数 Promise.prototype.then = function(onResolved, onRejected){ var self = this; return new Promise(function(resolve, reject){ var onResolvedFade = function(val){ var ret = onResolved?onResolved(val):val;//这一步主要是then方法中传入的成功回调函数通过return来进行链式传递结果参数 if(Promise.isPromise(ret)){//回调函数返回值也是promise的时候 ret.then(function(val){ resolve(val); }); } else{ resolve(ret); } }; var onRejectedFade = function(val){ var ret = onRejected?onRejected(val):val; reject(ret); }; self.handlers.push(onResolvedFade); if(self._status === FULFILLED){ onResolvedFade(self._value); } if(self._status === REJECTED){ onRejectedFade(self._value); } }); }

通过上面的代码可以看出，前面提出的2个问题得到了解决，1.在promise对象中有3个属性，state，value，handlers，这3个属性解决了状态和回调的脱离，并且在调用then方法的时候才将回调函数push到handlers属性上面（此时state就是1，可以在后面的代码中执行onResolve）2.链式调用通过在then方法中返回的promise对象实现，并且通过onResolvedFade将上一个回调的返回值当做这次的result参数来执行进行传递。

测试代码：

function async(value){ var pms = new Promise(function(resolve, reject){ setTimeout(function(){ resolve(value);; }, 1000); }); return pms; } async(1).then(function(result){ console.log('the result is ',result);//the result is 2 return result; }).then(function(result){ console.log(++result);//2 });


**总之，不同框架对promise的实现大同小异，上面的代码是ECMASCRIPT6标准的promise简单实现。jquery和其他框架也有实现，不过听说jquery的实现很糟糕- -！**
---
title: promises与observables的区别
source: http://blog.csdn.net/napolunyishi/article/details/51339006
kind: external
domain: observability
author: 成就一亿技术人
original_date: 2016-07-29
fetched_at: 2026-05-16
bookmark_title: promises与observables的区别 - 李子无为的专栏 - CSDN博客
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.csdn.net](http://blog.csdn.net/napolunyishi/article/details/51339006)
> 作者：成就一亿技术人
> 原始日期：2016-07-29
> 抓取日期：2026-05-16

# promises与observables的区别

1.observables 是lazy evaluation。

比如下面的代码片段，对于promise，无论是否调用then，promise都会被立即执行；而observables却只是被创建，并不会执行，而只有在真正需要结果的时候，如这里的foreach，才会被执行。

再举个例子，比如这里不是用setTimeout模拟异步操作，而是去请求一个url，那对于promise来说，then的作用是处理返回结果，而http请求在第一步就已经发送了；相反，对于observable来说，由于它发现你其实现在并不需要异步调用的结果，所以它干脆就不发送请求，而只有你真正需要响应数据的时候才会发送请求。

```
var promise = new Promise((resolve) => {
setTimeout(() => {
resolve(42);
}, 500);
console.log("promise started");
});
//promise.then(x => console.log(x));
var source = Rx.Observable.create((observe) => {
setTimeout(() => {
observe.onNext(42);
}, 500);
console.log("observable started");
});
//source.forEach(x => console.log(x));
```


2.observables可以被cancel。

observable能够在执行前或者执行过程中被cancel，或者叫做dispose。

下面的例子中，observable在0.5秒的时候被dispose，所以日志“observable timeout hit”不会被打印。

```
var promise = new Promise((resolve) => {
setTimeout(() => {
console.log("promise timeout hit")
resolve(42);
}, 1000);
console.log("promise started");
});
promise.then(x => console.log(x));
var source = Rx.Observable.create((observe) => {
id = setTimeout(() => {
console.log("observable timeout hit")
observe.onNext(42);
}, 1000);
console.log("observable started");
return () => {
console.log("dispose called");
clearTimeout(id);
}
});
var disposable = source.forEach(x => console.log(x));
setTimeout(() => {
disposable.dispose();
}, 500);
```


3.observable可以retry，或者多次调用。

上面的代码，可以拿到promise和observable的变量。对于promise，不论在后面怎么调用then，实际上的异步操作只会被执行一次，多次调用没有效果；但是对于observable，多次调用forEach或者使用retry方法，能够触发多次异步操作。

4.observable可以进行组合变换。

observable可以看做列表，可以进行各种组合变换，即LINQ操作，比如merge，zip，map，sum等等。这是observable相对于promise的一大优势。



参考：

https://egghead.io/lessons/rxjs-rxjs-observables-vs-promises

http://reactivex.io/documentation/operators.html
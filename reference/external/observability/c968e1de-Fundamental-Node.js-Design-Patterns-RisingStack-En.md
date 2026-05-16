---
title: Fundamental Node.js Design Patterns - RisingStack Engineering
source: https://blog.risingstack.com/fundamental-node-js-design-patterns/
kind: external
domain: observability
author: RisingStack Engineering
original_date: 2015-07-14
fetched_at: 2026-05-16
bookmark_title: Fundamental Node.js Design Patterns | RisingStack
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.risingstack.com](https://blog.risingstack.com/fundamental-node-js-design-patterns/)
> 作者：RisingStack Engineering
> 原始日期：2015-07-14
> 抓取日期：2026-05-16

# Fundamental Node.js Design Patterns - RisingStack Engineering

When talking about design patternsIf you encounter a problem that you think someone else solved already, there's a good chance that you can find a design pattern for it. Design patterns are "blueprints" prepared in a way to solve one (or more) problems in a way that's easy to implement and reuse. It also helps your team to understand your code better if they... you may think of **singletons**, **observers** or **factories**. This article is not exclusively dedicated to them but deals with other common patterns as well, like **dependency injection** or **middlewares**.

## What are design patterns?


A design pattern is a general, reusable solution to a commonly occurring problem.

### Singletons

The singleton patterns restrict the number of instantiations of a “class” to one. Creating singletons in Node.jsNode.js is an asynchronous event-driven JavaScript runtime and is the most effective when building scalable network applications. Node.js is free of locks, so there's no chance to dead-lock any process. is pretty straightforward, as `require`

is there to help you.

*//area.js*
var PI = Math.PI;
function circle (radius) {
return radius * radius * PI;
}
module.exports.circle = circle;


It does not matter how many times you will require this module in your application; it will only exist as a single instance.

```
var areaCalc = require('./area');
console.log(areaCalc.circle(5));
```


Because of this behaviour of `require`

, singletons are probably the most common Node.js design patterns among the modules in NPMnpm is a software registry that serves over 1.3 million packages. npm is used by open source developers from all around the world to share and borrow code, as well as many businesses. There are three components to npm: the website the Command Line Interface (CLI) the registry Use the website to discover and download packages, create user profiles, and....

### Observers

An object **maintains a list of dependents/observers and notifies them** automatically on state changes. To implement the observer pattern, `EventEmitter`

comes to the rescue.

*// MyFancyObservable.js*
var util = require('util');
var EventEmitter = require('events').EventEmitter;
function MyFancyObservable() {
EventEmitter.call(this);
}
util.inherits(MyFancyObservable, EventEmitter);


This is it; we just made an observable object! To make it useful, let’s add some functionality to it.

```
MyFancyObservable.prototype.hello = function (name) {
this.emit('hello', name);
};
```


Great, now our observable can emit event – let’s try it out!

```
var MyFancyObservable = require('MyFancyObservable');
var observable = new MyFancyObservable();
observable.on('hello', function (name) {
console.log(name);
});
observable.hello('john');
```


### Factories

The factory pattern is a creational pattern that doesn’t require us to use a constructor but provides a **generic interface for creating objects**. This pattern can be really useful when the creation process is complex.

```
function MyClass (options) {
this.options = options;
}
function create(options) {
```*// modify the options here if you want*
return new MyClass(options);
}
module.exports.create = create;


Factories also make testing easier, as you can inject the modules dependencies using this pattern.

### Dependency Injection

Dependency injection is a software design pattern in which one or more dependencies (or services) are injected, or passed by reference, into a dependent object.


In this example, we are going to create a `UserModel`

which gets a database dependency.

```
function userModel (options) {
var db;
if (!options.db) {
throw new Error('Options.db is required');
}
db = options.db;
return {
create: function (done) {
db.query('INSERT ...', done);
}
}
}
module.exports = userModel;
```


Now we can create an instance from it using:

```
var db = require('./db');
var userModel = require('User')({
db: db
});
```


Why is it helpful? It makes testing a lot easier – when you write your unit tests, you can easily inject a fake `db`

instance into the model.

### Middlewares / pipelines

Middlewares are a powerful yet simple concept: the **output of one unit/function is the input for the next**. If you ever used Express or Koa then you already used this concept.

It worths checking out how Koa does it:

```
app.use = function(fn){
this.middleware.push(fn);
return this;
};
```


So basically when you add a middleware it just gets pushed into a `middleware`

array. So far so good, but what happens when a request hits the server?

```
var i = middleware.length;
while (i--) {
next = middleware[i].call(this, next);
}
```


No magic – your middlewares get called one after the other.

### Streams

You can think of streams as special pipelines. They are better at processing bigger amounts of flowing data, even if they are bytes, not objects.

```
process.stdin.on('readable', function () {
var buf = process.stdin.read(3);
console.dir(buf);
process.stdin.read(0);
});
```


```
$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume2.js
<Buffer 61 62 63>
<Buffer 0a 64 65>
<Buffer 66 0a 67>
<Buffer 68 69 0a>
```


*Example by substack*

To get a better understanding of streams check out substack’s Stream Handbook.

## Further reading

- Node.js Best Practices
- Callback convention, asyncAsynchrony, in software programming, refers to events that occur outside of the primary program flow and methods for dealing with them. External events such as signals or activities prompted by a program that occur at the same time as program execution without causing the program to block and wait for results are examples of this category. Asynchronous input/output is an... code patterns, error handling and workflow tips.
- Node.js Best Practices Part 2
- The next chapter, featuring pre-commit checks, JavaScript code style checker and configuration.
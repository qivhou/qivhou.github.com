---
layout : post
category : translation
title : JavaScript中的Promise和Deferred对象  第一部分：理论和语义
tags : [翻译, jQuery, Deferred, Promise, javascript]
---
{% include JB/setup %}

原文链接：[Promise & Deferred objects in JavaScript Pt.1: Theory and Semantics](http://blog.mediumequalsmessage.com/promise-deferred-objects-in-javascript-pt1-theory-and-semantics)

## 介绍

在过去一段并不长的时间里，对于Javascript的程序员来说，处理异步事件的主要的工具是(callback)回调。

>(callback) 回调是一段可执行的代码，被当做一个参数传递给其他代码，它预期会被在合适的时候回调(执行)。 
——[Wikipedia](https://en.wikipedia.org/wiki/Callback_computer_programming)

换句话来说，一个函数可以被当做参数传递给另一个函数并且在被调用时执行。

(callbacks) 回调本身没有错，但是依赖于我们具体的编程环境，我们在管理异步事件上有一些其他可用的选择。在这篇文章中我的目标是考察一个可用工具集合：`promise`对象和`deferred`对象。在第一部分，我们将涵盖理论和语义，在第二部分我们将看看如何使用。

在Javascript中如何有效处理异步事件的关键点之一是理解程序会继续执行即使它没有得到为了进行中工作所需要的值。处理*从还没有完成的任务中获取未知值*使得javascript中的异步事件处理极具挑战，特别是当你第一次接触它的时候。

这种情形的一个经典的例子是`XMLHttpRequest (Ajax)`. 想象一下我们要：

*  发起一个`Ajax`请求来获取一些数据

*  立即用这些数据做一些事情

*  然后做其他事情

在我们的程序中我们初始化我们的`Ajax`请求。这个请求被发起但是不同于同步事件，我们程序的执行在服务器响应的时候不会停止而是继续运行。到我们从`Ajax`请求得到响应数据的时候，这个程序已经执行完毕。

## Promises 和 Deferreds : 它们是什么？

`Promises` 是一种[从1976年左右就已有的](http://en.wikipedia.org/wiki/Futures_and_promises)程序结构，简要的说：

* 一个`promise`代表一个还不知道的值

* 一个`deferred`代表一个还没有完成的任务

从更高一点的层次来考虑，JavaScript中的`promises`给了我们在并行方式下给同步代码编写异步代码的能力。在我们具体了解前，让我们用一个图解来得到一个大体的概览。

![Promises](http://www.mediumequalsmessage.com/blog-images/promises.png)

`promise`是一个最初未知结果的占位符，而`deferred`代表由这个值导致的计算。每一个`deferred`有一个`promise`，`promise`作为一个未来结果的代理。当一个`promise`是一个由异步函数返回的值时，`deferred`可以被它的 (caller) 调用者 (resolve) 解决或者 (reject) 拒绝，(caller) 调用者从 (resolver) 解决者分离出这个`promise`。`promise`自己可以被传给任意数量的 (consumer) 消费者，并且每一个消费者将独立观察结果，同时`resolver`/`deferred`可以被传给任意数量的 (producer) 生产者，并且`promise`将被其中第一个 (resolve) 解决它的生产者解决。从语义的角度上来看，它意味着我们可以不调用一个函数((`callback`) 回调 )就返回一个值(`promise`)。

## Promises 根据 Promise/A 提案

`Promises/A`提案提议以下标准行为和无关实现细节的*API*。
一个 `promise` :

* 代表一个独立操作完成时返回的最终值。

* 可能处于三种状态中的一种：(`unfulfilled`)未完成，(`fulfilled`)完成和(`failed`)失败,而且只能从(`unfulfilled`)未完成状态转换为(`fulfilled`)完成或者(`failed`)失败状态。

* 有一个函数作为"then"属性的值，(必须返回一个`promise`对象)。

* 添加`fulfilledHandler`(完成时处理程序),`errorHandler`(错误时处理程序)和`progressHandler`(进行中处理程序)，它们将在`promise`(completion)完成时被调用。
```javascript
    then(fulfilledHandler, errorHandler, progressHandler)
```
* 从回调处理程序返回的值是返回的`promise`对象的完成值。

* `promise`对象的值必须不能被改变(避免由监听者的意外行为产生的副作用)

换句话说，先把一些细节剥离出来：
一个`promise`对象作为一个未来值的代理，有三种可能的状态而且需要有一个函数对它的三种状态添加处理程序：`fulfilledHandler`(完成时处理程序),`errorHandler`(错误时处理程序)和`progressHandler`(进行中处理程序)(可选的)，当处理程序完成执行的时候，返回一个新的`promise`对象(允许链式操作)，这个新的`promise`对象将被`resolved`或者`rejected`。

## promise的返回值和状态

一个`promise`对象有三个可能的状态:(`unfulfilled`)未完成状态，(`fulfilled`)完成状态和(`failed`)失败状态。

* `unfulfilled`状态： 因为一个`promise`对象是一个未知值的代理，所以它起始时是`unfulfiled`状态。

* `fulfilled`状态： 当一个`promise`对象被等待的值填充的时候。

* `failed`状态： 如果当前`promise`被返回(exception)异常时，它在`failed`状态。

一个`promise`对象只能从(`unfulfilled`)未完成状态转换为(`fulfilled`)完成或者(`failed`)失败状态, 在(resolution)被解决或(rejection)被拒绝后，所有观察者被通知并且得到`promise`对象或者返回值。一旦`promise`对象被(resolved)解决或者(rejected)拒绝,它的状态和结果值将不能再被修改。

这儿是一个例子：

```javascript
// Promise to be filled with future value
var futureValue = new Promise();

// .then() will return a new promise
var anotherFutureValue = futureValue.then();

// Promise state handlers ( must be a function ).
// The returned value of the fulfilled / failed handler will be the value of the promise.
futureValue.then({

    // Called if/when the promise is fulfilled
    fulfilledHandler: function() {},

    // Called if/when the promise fails
    errorHandler: function() {},

    // Called for progress events (not all implementations of promises have this)
    progressHandler: function() {}
});

```

## 实现差异与性能

当选择一个`promise`库的时候，有许多需要考虑的因素。并不是所有库的实现都是相同的，它们可能会在API提供的功能，性能甚至行为上都有所不同。

因为`Promise/A` 提案仅仅对`promises`的行为进行了概述，并没有涉及具体的实现，不同的`promise`库包含不同的特性集。所有兼容`Promise/A`的实现有一个`.then()`函数但是在他们的API中也有不同特性。另外，他们仍然可以互相转换`promises`。`jQuery`是一个需要注意的特例，因为它的`promises`的实现并不是完全兼容`Promise/A`的。这个决定的影响被记录在[这里](https://gist.github.com/domenic/3889970)，讨论在[这里](http://bugs.jquery.com/ticket/11010)。在`Promise/A`兼容库上，一个抛出的异常被解释为一个(rejection)拒绝，`errHandler()`伴随着异常的抛出被调用。在`jQuery`的实现中，一个不被捕获的异常将停止程序的执行。作为不同实现的一个后果，在`jQuery`与那些返回或者期待得到兼容`Promise/A`的`promises`的库工作时，会有互通性的问题。对这个问题的一个解决方案是用其他`promise`库把`jQuery`的`promises`转换为`Promise/A`兼容的`promises`，然后使用这个兼容库的API。

例如：

```javascript
when($.ajax()).then()
```

在翻看`jQuery`坚持使用它们对`promises`实现的决定时，一个性能方面考虑的声明激发了我的好奇心，我决定做一个快速的性能测试。我使用了`Benchmark.js`,在`.then()`中用一个`success handler`测试了`deferred`对象的创建和解决。
结果如下：

```
    jQuery 91.6kb               when.js 1.04kb              Q.js 8.74kb
    9,979 ops/sec ±10.22%	    96,225 ops/sec ±10.10%	    2,385 ops/sec ±3.42%
    
```

    > 注意：使用`Closure Compiler`完成了最小化，但是没有用gzip压缩。

在执行完这些测试用例后，我发现了一篇揭秘了类似[`promise`库整体性能表现的更深度测试套件](https://github.com/cujojs/promise-perf-tests#test-results)的文章。
可能在现实应用中性能方面的差异微不足道，但是在我的`Benchmark.js`的测试中，`when.js`是毫无争议的胜者，从速度和文件大小来看，`when.js`是在需要考虑性能时一个极佳的选择。

众所周知，当选择一个`promise`库时，我们需要做一些权衡。我们到底需要使用哪个库，答案取决于你的具体用例和项目的需求，一些实现的特点如下：

* [`When.js`](https://github.com/cujojs/when) : 快速，轻量级的实现，有一些实用工具方法，v2.0版本对异步方案全面支持。

* [`Q.js`](https://github.com/kriskowal/q) : 可以在浏览器和`Node.js`中运行，提供健壮的API，完全兼容`Promise/A`.

* [`RSVP`](https://github.com/tildeio/rsvp.js) ：准系统实现，完全兼容`Promise/A`。

* [`jQuery`](http://api.jquery.com/category/deferred-object/) : 不兼容`Promise/A`但是被广泛实用。如果你已经在你的项目中实用了`jQuery`，很容易上手而且值得一试。

## 结论
`Promises` 提供给Javascript开发者一个处理异步事件的工具。现在我们对于什么是`promise`对象和`deferred`对象，它们是怎么工作的已经了解了一些，我们已经准备好深入下去看看具体怎么使用它们。在[第二部分](http://blog.mediumequalsmessage.com/promise-deferred-objects-in-javascript-pt2-practical-use)，我们将仔细看看在使用`promises`时一些常见的问题还有`jQuery` API的细节。同时，请随时参与[App.net](https://alpha.app.net/cwebbdesign)和[Hacker News](https://news.ycombinator.com/item?id=5499287)的讨论。

## 更多资源

### 书籍

* [Async Javascript](http://www.amazon.com/gp/product/B00AKM4RVG/ref=as_li_ss_tl?ie=UTF8&camp=1789&creative=390957&creativeASIN=B00AKM4RVG&linkCode=as2&tag=mediumequalsm-20)

### 文章

* [Promises/A Proposal](http://wiki.commonjs.org/wiki/Promises/A) 和 [Promises/A+ Spec](http://promises-aplus.github.com/promises-spec/)

* [You’re Missing the Point of Promises](https://gist.github.com/domenic/3889970)

* [#11010 (Make Deferred.then == Deferred.pipe like Promise/A) – jQuery Core - Bug Tracker](http://bugs.jquery.com/ticket/11010)

### JSFiddle

* [Promise/A forwarding](http://jsfiddle.net/briancavalier/4canN/) - jsFiddle documenting value and thrown error propagation in when.js

* [jQuery .then() and error propagation](http://jsfiddle.net/cwebbdesign/8BZhB/2/) - jsfiddle demonstrating value and thrown error propagation in jQuery



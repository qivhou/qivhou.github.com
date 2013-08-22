---
layout: post
category : translation
title: JavaScript中的Promise和Deferred对象 第二部分：实战
tags : [翻译, jQuery, Deferred, Promise, javascript]
---
{% include JB/setup %}


原文链接：[Promise & Deferred objects in JavaScript Pt.2: in Practice](http://blog.mediumequalsmessage.com/promise-deferred-objects-in-javascript-pt2-practical-use)

## 介绍

在[这篇博文的第一部分](http://blog.mediumequalsmessage.com/promise-deferred-objects-in-javascript-pt1-theory-and-semantics)，我花了很多时间查看`promises`和`deferreds`的理论知识：`promises`是什么和他们的行为方式是怎样的。现在是时候让我们实际的去深入并探索一些在`javascript`中使用`promises`和`deferreds`的方法和最佳实践。我将从一些基本的使用和`promises`的例子开始，然后深入到一些关于在`jQuery`中使用`promises`的具体的例子。

>注意：尽管`jQuery`与`Promise/A`提案在错误处理和与其他`promise`库一起工作的情况下，有一些显著的差异，但是代码示例将使用`jQuery`。因为第一部分在这个话题上已经讨论过并且包含了一些有深度的文章的链接，我将不会在这篇文章中再去谈它。`jQuery`的使用仍然在广泛传播着，而且对许多人来说它的实现是一个对`promises`使用的介绍，这两点让我相信理解jQuery中的`promises`是如何工作的是很有意义的。

## 序列模式
`deferred`是一个代表还未完成任务的对象，`promise`是一个代表未知值的对象。换句话说，`promises/deferreds`允许我们代表*简单*的任务，并可以很容易的组合起来代表复杂的任务和它们的工作流，而且可以允许对序列进行细粒度的控制。这意味着我们能并行的编写异步的`Javascript`和同步代码。另外，`promises`使得在多个异步任务中提取小的功能代码片段变得相对简单————例如加载动画，进度动画等例子。

让我们从一个3个普通序列模式的全局视图开始，`promises`使得堆栈模式，并行模式和顺序模式成为可能。

* 堆栈：在一个应用中，随时随地绑定多个处理程序到相同的`promise`事件。

```javascript
  var request = $.ajax(url);

  request.done(function () {
      console.log('Request completed');
  });

  // 在程序中的其他地方
  request.done(function (retrievedData) {
      $('#contentPlaceholder').html(retrievedData);
  });
```

* 并行任务：要求多个`promises`返回一个声明他们共同完成的`promise`。

```javascript
  $.when(taskOne, taskTwo).done(function () {
      console.log('taskOne and taskTwo are finished');
  });
```

* 顺序任务：按顺序执行任务。

```javascript
  var step1, step2, url;

  url = 'http://fiddle.jshell.net';

  step1 = $.ajax(url);

  step2 = step1.then(
    function (data) {
        var def = new $.Deferred();

        setTimeout(function () {
            console.log('Request completed');
            def.resolve();
        },2000);

      return def.promise();

  },
    function (err) {
        console.log('Step1 failed: Ajax request');
    }
  );
  step2.done(function () {
      console.log('Sequence completed')
      setTimeout("console.log('end')",1000);
  });
```
这些模式可以被组合或者独立使用来构建复杂的任务或者工作流。

## 常见用例
许多`promise`用例的例子是关于`Ajax`请求和UI动画的。事实上，`jQuery`甚至从`Ajax`请求默认返回`promises`。这对于那些完成时需要被以独特方式处理的异步任务来说是最理想的方式。然而，这并不意味着`promises`的使用应该限制在这些用例中。事实上，`promises`往往是在你需要调用回调函数时，值得考虑的一个工具，让我们看看我们可以使用`promises`的一些方式。

* Ajax

在`Ajax`请求中使用`promises`的例子可以在这篇文章中被找到，所以我将在这儿跳过这个例子。

* 定时

我们可以基于`timeout`函数创建一个`promise`。

```javascript
  function wait(ms) {
      var deferred = $.Deferred();
      setTimeout(deferred.resolve, ms);

     // 我们只需要返回promise而不是整个deferred
     return deferred.promise();
  }

  // 使用它
  wait(1500).then(function () {
      // 在这儿做一些操作
  });

```
* 动画

很显然下面的动画是完全没有用的，但是作为一个例子它告诉我们`promises`和动画是如何被一块使用的。

```javascript
    var fadeIn = function (el) {

      var promise = $(el).animate({
          opacity: 1
      }, 1500);

      // 动态创建并返回一个可观察的promise对象，它将在动画完成时被解决。
      return promise.promise();
    };

    var fadeOut = function(el) {

        var promise = $(el).animate({
            opacity: 0
        }, 1500);

        // 动态创建并返回一个可观察的promise对象
          return promise.promise();
    };

    // .随着设置的完成，我们可以做下面的任意一个

    // 并行
    $.when(
        fadeOut('div'), 
        fadeIn('div')
    ).done(function () {
        console.log('Animation finished');
        $('p').css('color', 'red');
    });

    // 或者
    // 链式
    fadeOut('div').then(function (el) {
        fadeIn(el); // returns a promise
    }).then(function (el) {
        fadeOut(el); // returns a promise
    });

```
* 用`$.when`同步并行任务

```javascript
    var promiseOne, promiseTwo, handleSuccess, handleFailure;

    // Promises
    promiseOne = $.ajax({ url: '../test.html' });
    promiseTwo = $.ajax({ url: '../test.html' });
      
      
    // Success 回调
    // .done() 将只在promise成功resolved的时候运行
    promiseOne.done(function () {
        console.log('PromiseOne Done');
    });

    promiseTwo.done(function () {
        console.log('PromiseTwo Done');
    });

    // $.when() 创建一个新的promise
    // ，它将被resolved如果两个内部的promises都被resolved
    // 将被rejected如果两个promises中的一个被rejected
    $.when(
        promiseOne,
        promiseTwo
    )
    .done(function () {
        console.log('promiseOne and promiseTwo are done');
    })
    .fail(function () {
        console.log('One of our promises failed');
    });

```

* 解耦事件和应用逻辑

我们也可以使用事件去触发`promises`的解决和失败，并且传递值，同时它允许我们解耦应用，DOM和事件逻辑([jsfiddle在这儿](http://jsfiddle.net/cwebbdesign/NEssP/2/))。

```javascript
var def, getData, updateUI, resolvePromise;

// The Promise and handler
def = new $.Deferred();

updateUI = function (data) {
    $('p').html('I got the data!');
    $('div').html(data);
};
getData = $.ajax({
          url: '/echo/html/', 
          data: {
              html: 'testhtml', 
              delay: 3
          }, 
          type: 'post'
    })
    .done(function(resp) {
        return resp;
    })
    .fail(function (error) {
        throw new Error("Error getting the data");
    });


// Event Handler
resolvePromise = function (ev) {
    ev.preventDefault();
    def.resolve(ev.type, this);
    return def.promise();
};

// Bind the Event
$(document).on('click', 'button', resolvePromise);

def.then(function() {
    return getData;   
})
.then(function(data) {
    updateUI(data);
})
.done(function(promiseValue, el) {
    console.log('The promise was resolved by: ', promiseValue, ' on ', el);
});


// Console output: The promise was resolved by: click on <button> </button>

```

## Gotcha's： 理解jQuery中的.then()

为了证明一些"Gotcha's"，最后这部分，我用到了一些我第一次接触`promises`时的例子。

让我们为下面的例子假定两个工具函数：

```javascript
// Utility Functions
function wait(ms) {
      var deferred = $.Deferred();
      setTimeout(deferred.resolve, ms);
      return deferred.promise();
}
function notifyOfProgress(message, promise) {
    console.log(message + promise.state());
}
```

首先，我尝试把`promises`链在一起，看起来像下面这样：

```javascript
// Naive attempt at working with .then()

// 创建两个新的`deferred`对象
var aManualDeferred = new $.Deferred(),
    secondManualDeferred = aManualDeferred.then(function () {
        console.log('1 started');

        wait(3500).done(function () {
            console.log('1 ended');
        });
    });

// After secondManualDeferred is resolved
secondManualDeferred.then(function () {
    console.log('2 started');

    wait(2500).done(function () {
        console.log('2 ended');
    });
});

// Resolve the first promise
aManualDeferred.resolve();
```

执行上面的代码，控制台输出是我没有使用`promises`时所期待的结果。

```
1 started
2 started
2 ended
1 ended
```

`jQuery`的API中描述`.then()`是可以链式操作并且返回一个新的`promise`,所以我的期望是不管我把什么封装在`.then()`中，把什么链在一起，它都将会按顺序发生，并且必须等待之前的任务完成才可以开始下一个任务。显然这并不是刚才发生的，为什么呢？

##`.then()`究竟是怎么工作的？
我们从`jQuery`的源码中发现：

* `.then()`总是返回一个新的`promise`

* `.then()`必须被传递一个函数

如果`.then()`没有被传递一个函数：

* 那个新的`promise`将具有和原`promise`一样的行为(这意味着它将立即被`resolved`或者`rejected`)

* `.then()`中的输入将会被执行，但是它将会被`.then()`忽略

如果`.then()`被传递了一个返回值是`promise`对象的函数

* 这个新的`promise`对象将具有和返回的`promise`对象一样的行为

```javascript
    var deferred = $.Deferred(),
        secondDeferred = deferred.then(function () {
          return $.Deferred(function (newDeferred) {
            setTimeout(function() {
              console.log('timeout complete');
            newDeferred.resolve();
          }, 3000);
        });
      }),
      thirdDeferred = secondDeferred.then(function () {
          console.log('thirdDeferred');
      });

    secondDeferred.done(function () {
        console.log('secondDeferred.done');
    });
    deferred.resolve();
```

* 如果`.then()`被传递了一个有返回值的函数，这个值将成为新对象的值

```javascript
    var deferred = $.Deferred(),
        filteredValue = deferred.then(function (value) {
          return value * value;
        });

    filteredValue.done(function (value) {
        console.log(value);
    });

    deferred.resolve(2); // 4
```
你可能已经看到了(如果你没有马上看到它)为什么我的版本不可用。我没有明确的从`.then()`返回一个`promise`，所以被`.then()`创建的新`promise`有着和它链的`promise`一样的值。

## 避免陷入回调地狱

我们知道我们需要传递给`.then()`一个函数使它能够完成它的任务，并且我们知道我们需要从`.then()`返回一个`promise`。所以我们可以像下面这样做：

```javascript
// Anti-pattern - Return to callback hell

var aManualDeferred = new $.Deferred();

aManualDeferred.then(function () {
    console.log('1 started');

    return wait(3500).then(function () {
        console.log('1 ended');
    }).then(function () {
        console.log('2 started');

        return wait(2500).done(function () {
            console.log('2 ended');
        });
    });
});

// Resolve the first promise
aManualDeferred.resolve();

```
这次成功了。不幸的是，它重新回到了回调地狱模式，但这正是`promises`应该帮助我们避免的事之一。幸运的是，有一些方法使我们不用深层嵌套函数就可以解决这个问题。当然，我们选择用怎样的方式去解决它视我们的具体情况而定。

### 避免广泛使用匿名的`promises`

比如，我们可以像下面这样做：

```javascript
// A chain
// Create new deferred objects
var aManualDeferred = $.Deferred();

aManualDeferred.then(function () {
    console.log('1 started');

    // We need to return this, we return a new promise which is resolved upon completion.
    return wait(3500);
})

.then(function () {
    console.log('1 ended');
})

.then(function () {
    console.log('2 started');
    return wait(2500);
})

.then(function () {
    console.log('2 ended');
});

// Resolve the first promise
aManualDeferred.resolve();
```
这个版本读起来相当不错，但是却有只用一个命名`promise`的缺点，它并不能让我们真正对过程中的每一个步骤进行细粒度的控制，而这正是很多情况下需要的。

### 松绑promises和他们的处理程序

假设我们想要避免深层嵌套函数，而且我们应该命名`promises`来使我们能够访问过程中的每一个步骤，那么这里的最终版本就是我们需要的。

```javascript
var aManualDeferred, secondManualDeferred, thirdManualDeferred;

// Create two new deferred objects
aManualDeferred = $.Deferred();

secondManualDeferred = aManualDeferred.then(function () {
    console.log('1 started');

    // We need to return this, we return a new promise which is resolved upon completion.
    return wait(3500);
})
.done(function () {
    console.log('1 ended');
});

thirdManualDeferred = secondManualDeferred.then(function () {
    console.log('2 started');
    return wait(2500);
})
.done(function () {
    console.log('2 ended');
});

// Check current state
thirdManualDeferred.notify(
    notifyOfProgress('thirdManualDeferred ', thirdManualDeferred)
);

// Resolve the first promise
aManualDeferred.resolve();

// Console output
// aManualDeferred pending
// secondManualDeferred pending
// 1 started
// 1 ended
// 2 started
// 2 ended
```
这个版本的优点是我们清楚地看到现在有三个步骤，我们可以得到每一个`promise`的状态并发送它进度的通知，或者之后不用重写代码就可以按需管理我们的序列。

## 上下文和数据传递

在早些的`Ajax`例子中，我们看到我们可以传递一个值给`.resolve()`和`.fail()`。如果一个`promise`用一个值被解决，它将把这个值作为自己进行返回。

```javascript
var passingData = function () {
    var def = new $.Deferred();

    setTimeout(function () {
        def.resolve('50');
    }, 2000);

   return def.promise();               
};

passingData().done(function (value) {
      console.log(value);
});
```

我们也可以设置`this`当我们解决一个`promise`对象时

```javascript
// Create an object
var myObject = {
    myMethod: function (myString) {
        console.log('myString was passed from', myString);
    }
};

// Create deferred
var deferred = $.Deferred();

// deferred.done(doneCallbacks [, doneCallbacks ])
deferred.done(function (method, string) {
    console.log(this); // myObject

    // myObject.myMethod(myString);
    this[method](string);
});

deferred.resolve.call(myObject, 'myMethod', 'the context');

=> myString was passed from the context

// We could also do this:
// deferred.resolveWith(myObject, ['myMethod', 'resolveWith']);
// but it's somewhat annoying to pass an array of arguments.

// => myString was passed from resolveWith
```
## 最佳实践
我曾尝试列举在这个过程中的一些最佳实践，但是为了清楚起见，允许我用一个个标题概括它们。十分坦率的说，它们中间的大多数采用了在使用`promises`的一些最佳实践：特别是[DRY](http://en.wikipedia.org/wiki/Don't_repeat_yourself)和[单一职责原则](http://en.wikipedia.org/wiki/Single_responsibility_principle)。

* 命名你的`promises`

```javascript
var step2 = step1.then()
```

* 通过从`.then()`中调用命名函数分离处理函数和`promise`逻辑，并且把功能分解为可重用的小单元。

```javascript
var someReusableFunctionality = function () {
    // do something
};

step2.then(someReusableFunctionality);
```

* 当有逻辑时，返回一个`promise`而不是一个`deferred`，这样使得没有人可以在无意中解决/拒绝这个`promise`

```javascript
step2.then(function() {
    // we don't want to give resolution / rejection powers 
    // to the wrong parties, so we just return the promise.
    return deferred.promise();
});
```

* 不要陷入嵌套回调地狱或者嵌套`promise`地狱

通过这些最佳实践，我们可以从`promises`中得到最大的收益。我们可以精心用可读的代码去解耦应用，获得对异步事件序列细粒度的控制，处理那些尚不存在的值和尚未完成的操作。

## jQuery相关

我想对`jQuery` API进行一个概括的总结，因为我的代码例子都集中在`jQuery`对`promises`的实现上。如果你正在使用一个不同的`promises`实现，你可能想要跳过这节。

### 笔记

* `deferred.always()`, `deferred.done()`, `deferred.fail()` 返回`deferred`对象。

* `deferred.then()`, `deferred.when()`, `deferred.promise()` 返回一个`promise`对象。

* `$.ajax`和`$.get`返回`promise`对象。

* 你可以用你希望继承的上下文调用`resolve`,而不用使用`.resolveWith()`和`.rejectWith()`。

* 传递`deferred.promise()`而不是`deferred`，因为`deferred`对象自己不能通过它被解决或者拒绝。

### [$.Deferred()](http://api.jquery.com/jQuery.Deferred/)

一个创造新`deferred`对象的构造函数。接受可选的初始化函数，初始化函数将在`deferred`创建后被立即执行。

### deferred.always()

返回`deferred`对象并且在解决或者拒绝后执行附加的函数。

```javascript
$.get("test.php")

// Execute regardless of resolution or success
.always(function() {
    alertCompletedRequest();
});
```
### deferred.then()

添加处理程序，这些处理程序将被在解决，拒绝或者进行中的时候被调用，返回一个`promise`。

```javascript
$.get("test.php")

// Execute regardless of resolution or success
.then(function() {
    alertSuccess(),
    alertFailure(),
    alertProgress();
});
```

### deferred.when()

基于多个`promises`的完成返回一个新的promise对象。如果任意`promise`被拒绝，`.when()`被拒绝，如果所有`promises`被解决，它被解决。值得注意的是一个非`promise`可以被传递给`.when()`，它将被像一个`resolved promise`对待。同样值得注意的是它将返回一个单值仅当所有其他`promises`解决为一个单值或者一个数组，否则它将解决为一个数组。

### deferred.resolve(optionalArgs) 或者 deferred.reject(optionalArgs)

解决或者拒绝`deferred`对象，调用处理程序(`.done()`,`.fail()`,`.always()`,`.then()`)，并把提供的所有参数和他们的上下文传递给被调用的函数。

```javascript
$('body').on('button', 'click', function() {

    // Can be passed a value which will be given to handlers
    deferred.resolve();
});
```

### deferred.promise()

返回`deferred`的`promise`对象。如果传递一个目标对象，`.promise()`将把`promise`方法附加到它的目标对象升上而不是创建一个新的对象。

### deferred.state()

对于调试和查询`deferred`对象的状态很有用。返回"pending","resoluved"或者"resolved"。

### deferred.always()

不管是`reject`还是`failure`，函数或者函数数组将被调用。

### deferred.done()

在`deferred`对象被解决后，函数或者函数数组将被调用。

### deferred.fail()

在`deferred`对象被拒绝后，函数或者函数数据将被调用。

### $.ajax()

发起`Ajax`请求并且返回一个`promise`对象。

## 结论

管理异步`Javascript`和编写解耦应用是有挑战的。我希望到现在为止，你已经对于什么是`promise`，你可以怎样使用它们，还有应该怎样去避免一些常见的陷阱有了一个更好的理解。仍然有很多的知识我没有能够在这两篇文章中覆盖到，我指的是你的库文档还有在两篇文章后提到的资源。当然，如果你有问题或者任何想法，请随时联系[App.net](https://alpha.app.net/cwebbdesign)和[Hacker News](https://news.ycombinator.com/item?id=5499287)。

### 作者注：
能将这两篇文章放在一起，我深深的感激其他一些人的努力。@dominic的文章[You're Missing the Point of Promises](https://gist.github.com/domenic/3889970) 还有@dominic和@rwaldron在[jQuery's .then()](http://bugs.jquery.com/ticket/11010)的交流真正的推动我深入研究`promise`是如何工作的。Trevor Burnham的书[Async Javascript](http://www.amazon.com/gp/product/B00AKM4RVG/ref=as_li_ss_tl?ie=UTF8&camp=1789&creative=390957&creativeASIN=B00AKM4RVG&linkCode=as2&tag=mediumequalsm-20),Brian Cavalier的[Async Programming](http://blog.briancavalier.com/),Jesse Haller的[Promise Pipeline in Javascript](http://sitr.us/2012/07/31/promise-pipelines-in-javascript.html), 当然[Promises/A](http://wiki.commonjs.org/wiki/Promises/A)和[Promises/A+](http://promises-aplus.github.com/promises-spec/)提案是非常宝贵的资源。最后，特别感谢[Rick Waldron](http://weblog.bocoup.com/)和`jQuery`的`.then()`实现的作者[Julian Aubourg](http://jaubourg.net/)在我准备这篇文章时回答我的问题。

## 更多资源

### 书
* [Async JavaScript](http://www.amazon.com/gp/product/B00AKM4RVG/ref=as_li_ss_tl?ie=UTF8&camp=1789&creative=390957&creativeASIN=B00AKM4RVG&linkCode=as2&tag=mediumequalsm-20)

### 文章

* [Async Programming Part 1: It’s messy](http://blog.briancavalier.com/async-programming-part-1-its-messy)

* [Async Programming Part 2: Promises](http://blog.briancavalier.com/?sort=&search=async%20programming%20part%202-)

* [Promise Pipelines in JavaScript](http://sitr.us/2012/07/31/promise-pipelines-in-javascript.html)

* [What’s the point of promises](http://www.kendoui.com/blogs/teamblog/posts/13-03-28/what-is-the-point-of-promises.aspx)

* [jQuery Documentation: Deferred Object](http://api.jquery.com/category/deferred-object/)

* [jQuery - deferred.js test suite](https://github.com/jquery/jquery/blob/master/test/unit/deferred.js)



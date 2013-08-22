---
layout : post
category : translation
title : 理解 jQuery.Deferred 和 Promise
tags : [翻译, jQuery, Deferred, Promise, javascript]
---
{% include JB/setup %}

原文链接：[Understanding JQuery.Deferred and Promise](http://joseoncode.com/2011/09/26/a-walkthrough-jquery-deferred-and-promise/)

请先阅读我写的有关`promise`的更准确的[新文章](http://joseoncode.com/2013/05/23/promises-a-plus/)。

    请注意这篇文章中有很多可以运行的javascript例子，但是它们可能并不能在RSS阅读器中工作。

jQuery1.5引入了"Deferred"概念，你可以通过[这儿](http://api.jquery.com/category/deferred-object/)更多了解它。

我发现这个概念在与javascript和ajax一起使用时非常强大，并且我觉得它会改变我们用js编写异步代码的方式。

通常，我们在javascript中习惯于使用把callbacks作为参数传递给函数的方式来处理异步代码。

```javascript

    $.ajax({
        url: "/echo/json/",
        data: {json: JSON.stringify({firstName: "Jose", lastName: "Romaniello"})} ,
        type: "POST",
        success: function(person){ 
            alert(person.firstName + " saved."); 
        },
        error: function(){ 
            alert("error!");
        } 
    });

```

正如你看到的，我们使用了(object-literal syntax)对象字面量表示法传递了2个(callbacks)回调函数(success和error)给`$.ajax`方法。

这样可以工作但它并不是一个标准的接口，它要求在post方法中做一些处理的工作而且我们不能够注册多个(callbacks)回调函数。

## 介绍$.Deferred
`deferred`对象是非常强大而且容易去理解的，它有两个重要的方法

*   resolve

*   reject

而且还有三个重要的事件和方式去附加回调函数

*   done

*   fail

*   always

一个最基本的例子将看上去像下面这样(你可以在这儿运行它)

```javascript
    var deferred = $.Deferred();

    deferred.done(function(value) {
       alert(value);
    });

    deferred.resolve("hello world");
```

如果你调用"reject"方法，附加的`fail callback`将被执行，不管`deferred`的状态是“resolved”还是“rejected”，`always callack`都会被执行。

另一个有趣的地方是如果你附加一个回调函数给一个状态已经是“resolved”的`deferred`,这个回调函数将会立即执行。

```javascript
    var deferred = $.Deferred();

    deferred.resolve("hello world");

    deferred.done(function(value) {
        alert(value);
    });
```

## Promise()方法

`Deferred`对象有另一个重要的方法叫做`Promise()`。这个方法返回一个和`Deferred`几乎有着一样接口的对象，但是它只有那些可以去附加回调函数的方法而没有那些可以把状态改变为“resoved”和“rejected”的方法。

这在你需要传递一些值到正在调用的API去交互的时候是很有用的，但是它并不能把`deferred`的状态改为“resolved”或者“rejected”。以下的这些代码将会运行失败因为`psomise`没有一个叫做"resolve"的方法。

```javascript
    function getPromise(){
        return $.Deferred().promise();
    }

    try{
        getPromise().resolve("a");
    }
    catch(err){
        alert(err);
    }
```

在jQuery中的`$.ajax` 方法会返回一个`Promise`对象, 所以你可以像下面这样操作：

```javascript
    var post = $.ajax({
        url: "/echo/json/",
        data: {json: JSON.stringify({firstName: "Jose", lastName: "Romaniello"})} ,
        type: "POST"
    });

    post.done(function(p){
        alert(p.firstName +  " saved.");
    });

    post.fail(function(){
        alert("error!");
    });
```

这段代码的功能和第一段代码一样，唯一的不同是你可以按你的需要去添加多个回调函数，这种写法很清晰，因为我们在方法中不需要额外的参数。另一方面你可以使用`promise`作为其他函数的返回值，对我来说最有趣的的是你可以使用`promise`对象做一些操作。

## Pipe()方法
> 弃用说明：在jQuery 1.8中，`deferred.pipe`方法被弃用了。`deferred.then`方法被用来替代它。

`Pipe`方法是很强大而且实用的，它允许你投影出一个`promise`。

按照之前的例子，我们可以像下面这样写一些代码：

```javascript
    var post = $.post("/echo/json/",
            {
                json: JSON.stringify({firstName: "Jose", lastName: "Romaniello"})
            }
        ).pipe(function(p){
            return "Saved " + p.firstName;
        });

    post.done(function(r){ alert(r); });
```

在这里我们做了一个结果的投影，它是一个"person"对象。所以，我们现在有了一个"Saved {firstname}"的`deferred`，而不是一个Person的`Deferred`。

`Pipe`方法另一个有趣的特性是你可以从`pipe`的回调函数内部返回一个`deferred`。想象一下我们有两个方法，一个通过内部id获取客户的SSN ，另一个是通过SSN得到客户的地址。

```javascript
    function getCustomerSSNById(customerId){
        return $.post("/echo/json/", {
                json: JSON.stringify({firstName: "Jose", lastName: "Romaniello", ssn: "123456789"})
        }).pipe(function(p){
            return p.ssn;
        });
    }

    function getPersonAddressBySSN(ssn){
        return $.post("/echo/json/", {
                json: JSON.stringify({
                    ssn: "123456789",
                    address: "Siempre Viva 12345, Springfield" })
        }).pipe(function(p){
            return p.address;
        });
    }

    function getPersonAddressById(id){
        return getCustomerSSNById(id)
               .pipe(getPersonAddressBySSN);  
    }


    getPersonAddressById(123)
        .done(function(a){
            alert("The address is " + a); 
        });
```

正如你所看到的，我添加了一个新方法"getPersonAddressById". 这个方法返回了一个`deferred`,它是两个方法的联合体，如果在`pipe`中的两个方法中的一个失败了，主的`deferred`将会失败。

`pipe`还有一些其他的用法，比如，你可以在pipe回调函数内把`deferred`状态改为“rejected”。

我想到的关于(pipes)管道的另一个有趣的用例是递归`deferred`。想象一下你在后端开始了一个异步操作，你需要做轮询去查看是否这个操作已经完成了，而且在这个操作完成的时候去做一些其他的事情。

```javascript
    //1: done, 2: cancelled, other: pending
    function getPrintingStatus(){
        var d = $.Deferred();
        $.post(
            "/echo/json/",
            {
                json: JSON.stringify( {status: Math.floor(Math.random()*8+1)} ),
                delay: 2
            }
        ).done(function(s){
            d.resolve(s.status);
        }).fail(d.reject); 
        return d.promise();
    }

    function pollUntilDone(){
        //do something
        return getPrintingStatus()
                .pipe(function(s){
                    if(s === 1 || s == 2) {
                        return s;  //if the status is done or cancelled return the status
                    }
                    //if the status is pending... call this same function
                    //and return a deferred...
                    return pollUntilDone();
                });
    }

    $.blockUI({message: "Loading..."});

    pollUntilDone()
        .pipe(function(s){ //project the status code to a meaningfull string.
                switch(s){
                case 1:
                    return "done";
                case 2:
                    return "cancelled";
                }  
        })
        .done(function(s){
            $.unblockUI();
            alert("The status is " + s);
        });

```

## 联合 promises 和 $.when

另一个非常有用的方法是`$.when`。这个方法接受任意数量的`promises`，并且返回一个主`deferred`，这个主`deferred`

*   当所有`promises`状态被为“resolved”的时候，它的状态为“resolved”。

*   当任何一个`promise`状态为“rejected”的时候，它的状态为“rejected”。

done的回调函数中有所有`promises`的结果。

下面是一个例子：
    
```javascript
    function getCustomer(customerId){
        var d = $.Deferred();
        $.post(
            "/echo/json/",
            {json: JSON.stringify({firstName: "Jose", lastName: "Romaniello", ssn: "123456789"})}
        ).done(function(p){
            d.resolve(p);
        }).fail(d.reject); 
        return d.promise();
    }

    function getPersonAddressBySSN(ssn){
        return $.post("/echo/json/", {
                json: JSON.stringify({
                    ssn: "123456789",
                    address: "Siempre Viva 12345, Springfield" })
        }).pipe(function(p){
            return p.address;
        });
    }

    $.when(getCustomer(123), getPersonAddressBySSN("123456789"))
        .done(function(person, address){
            alert("The name is " + person.firstName + " and the address is " + address);
        });

```

正如你在例子最后看到的，`$.when`返回了一个新的`deferred`，并且我们使用了两个返回值在done的回调函数中。

请注意我更改了`getCustomer`方法；这是因为ajax调用的`promise`把内容负载作为结果中的第一个元素，但是也包含其他元素比如说状态码。

这样你可以像下面这样混合使用`$.when`和`pipe`:

```javascript
    function getCustomer(customerId){
        var d = $.Deferred();
        $.post(
            "/echo/json/",
            {json: JSON.stringify({firstName: "Jose", lastName: "Romaniello", ssn: "123456789"})}
        ).done(function(p){
            d.resolve(p);
        }).fail(d.reject); 
        return d.promise();
    }

    function getPersonAddressBySSN(ssn){
        return $.post("/echo/json/", {
                json: JSON.stringify({
                    ssn: "123456789",
                    address: "Siempre Viva 12345, Springfield" })
        }).pipe(function(p){
            return p.address;
        });
    }

    $.when(getCustomer(123), getPersonAddressBySSN("123456789"))
        .pipe(function(person, address){
            return $.extend(person, {address: address});
        })
        .done(function(person){
            alert("The name is " + person.firstName + " and the address is " + person.address);
        });
```

另一个有趣的`$.when`操作符用例是当你需要在屏幕中加载若干东西，但是你只想要一个大的加载信息:

```javascript
    function getCustomer(customerId){
        var d = $.Deferred();
        $.post(
            "/echo/json/",
            {json: JSON.stringify({firstName: "Jose", lastName: "Romaniello", ssn: "123456789"}),
             delay: 4}
        ).done(function(p){
            d.resolve(p);
        }).fail(d.reject); 
        return d.promise();
    }

    function getPersonAddressBySSN(ssn){
        return $.post("/echo/json/", {
                 json: JSON.stringify({
                                ssn: "123456789",
                                address: "Siempre Viva 12345, Springfield" }),
                 delay: 2
        }).pipe(function(p){
            return p.address;
        });
    }


    function load(){
        $.blockUI({message: "Loading..."});
        var loadingCustomer = getCustomer(123)
                                .done(function(c){
                                    $("span#firstName").html(c.firstName)
                                });

        var loadingAddress = getPersonAddressBySSN("123456789")
                                .done(function(address){
                                    $("span#address").html(address)
                                });
        
        $.when(loadingCustomer, loadingAddress)
         .done($.unblockUI);
    }

    load();
```

还有有一些其他有用的方法和方式你可以用来结合`deferred`，我强烈推荐你读一读jquery中的相关文档。

欧了，我希望你觉得这篇文章对你有用。



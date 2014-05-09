---
layout : post
category : deep-into-grunt
title : Grunt源码解析之'/grunt/option.js'
tags : [Grunt, 源码分析, grunt.option]
---
{% include JB/setup %}

源码路径：https://github.com/gruntjs/grunt/blob/master/lib/grunt/option.js

该模块是一个简单的模块，代码很少而且没有调用外部模块。但是因为`options`遍布整个`grunt`源码，所以对其中的条理能做到心中有数还是极好的。

## 模块内部代码解析

首先，定义一个`data`变量用来存放option数据。

```javascript
// The actual option data.
var data = {};
```
接下来是常规写法，定义`option`对象，该对象实际是对一个函数的引用，该函数接受`key`和`value`参数。

```javascript
// Get or set an option value.
var option = module.exports = function(key, value) {
```
Note：上面这行代码看似很简单，但是相信大家看到过或者甚至在有很多模块中是按照下面这样写的，

```javascript
exports.func1 = function database_module(cfg) {return;}
```

究竟`module.exports`和`exports`是什么关系呢？很多人可能并不很清楚，我也是有个偶然的机会才去深究了一下。

实际上`module.exports`是作为`require`的调用实际返回的结果对象，`exports`只是在初始情况下一个指向`module.exports`对象的变量，它更像是一个快捷方式或者别名。

如果在代码中，你重新给`exports`进行了赋值，因为在赋值后它不再指向`module.exports`了，所以`module.exports`和`exports`将指向两个不同的对象，你在`exports`上的所有努力都不会在`require`这个模块时体现出来。所以如果你想要将一个新的对象或者函数的引用指向`exports`的话，你应该同时把这个引用指向`module.exports`，就像下面的[underscore][]中那样，我觉得这是一个最佳实践。

在[underscore][]中的写法如下：

```javascript
if (typeof exports !== 'undefined') {
    if (typeof module !== 'undefined' && module.exports) {
      exports = module.exports = _;
    }
    exports._ = _;
  } else {
    root._ = _;
  }
```
用例子来说明：

```javascript
//myModule.js

exports = function AConstructor() {}; // override the original exported object
exports.method2 = function () {}; // exposed to the new exported object

// method added to the original exports object which not exposed any more
module.exports.method3 = function () {}; 
```
当你用如下方法对`myModule.js`进行访问的时候，这个`mymodule`对象只有`method3`方法，而没有`method2`。

```javascript
var mymodule = require('./path/to/myModule');
```
如果你想要了解更多关于这个方面的信息的话，请查看[What is the purpose of Node.js module.exports and how do you use it?](http://stackoverflow.com/questions/5311334/what-is-the-purpose-of-node-js-module-exports-and-how-do-you-use-it)还有[module.exports vs exports in nodeJS](http://stackoverflow.com/questions/7137397/module-exports-vs-exports-in-nodejs)。

言归正传，继续看源码。

这个函数内部其实很简单，用正则`/^no-(.+)$/`去匹配`key`值，并将结果值给变量`no`。

```javascript
  var no = key.match(/^no-(.+)$/);
```
如果接受的参数是两个，那么这个可以看做是一个set函数，对`data`变量的`key`值属性进行赋值，值为`value`值，并将`value`值返回。

```javascript
  if (arguments.length === 2) {
    return (data[key] = value);
```
如果上面提到的正则匹配结果`no`变量不为`null`的话，那么对`data`变量的`no[1]`值属性与`false`进行比较，并将结果返回
。

```javascript
  } else if (no) {
    return data[no[1]] === false;
```
如果上述条件都不满足的话，取出`data`变量的`key`值属性并返回。

```javascript
} else {
    return data[key];
  }
};
```
上述的三种情况，分别用下面的例子来说明：

```javascript
var option = require(/path to/option);

...
//相关操作
...

//第一种情况
var result1 = option("target", "dev");
// dev

//第二种情况
var result2 = option("no-force");
// 在这个过程中option中的no变量的值如下
//[ 'no-force',
//  'force',
//  index: 0,
//  input: 'no-force' ]
//所以result2的值为 data["false"] === false 的表达式值

//第三种情况
var result3 = option("force");
//在这个过程中option中的no变量的值如为null
//所以result3的值为data["force"]

```

接下来，给`option`对象增加`init`方法。

### option.init

这个方法接受`obj`参数对`data`变量进行初始化，或者当`obj`为`undefined`时将`data`初始化为一个空对象`{}`。

```javascript
// Initialize option data.
option.init = function(obj) {
  return (data = obj || {});
};
```
接下来，给`option`对象增加`flags`方法。

### option.flags

这个方法用来将所有的`data`变量中的属性进行格式化输出：

```javascript
// List of options as flags.
option.flags = function() {
```
总体来说，这是一个大的链式操作。

首先，通过`Object.keys`[方法](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/keys)得到`data`变量的所有自有属性名，即`data`的所有自有key值。然后，用`Array.prototype.filter`[方法](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter)得到所有满足条件的数组项然后生成新的数组。

这个`filter`函数将过滤掉`data`变量中值为空数组的属性名，并将其余属性名生成新的数组。

```javascript
  return Object.keys(data).filter(function(key) {
    // Don't display empty arrays.
    return !(Array.isArray(data[key]) && data[key].length === 0);
```
接下来，对数组调用`Array.prototype.map`[方法](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map)。

```javascript
  }).map(function(key) {
```
在回调函数中，首先从`data`变量中将当前遍历到的属性的属性值取出并赋值给`val`变量。

然后，通过两个三元操作符对`val`的值进行判断然后得到相应内容，对各结果字符串进行拼接然后返回。

```javascript
    var val = data[key];
    return '--' + (val === false ? 'no-' : '') + key +
      (typeof val === 'boolean' ? '' : '=' + val);
  });
};
```
参看下面的例子，了解这个函数的运行情况：

```javascript
> var data = {force: true, write: false, target: "dev", filter: "isFile", cwd: "/usr/workspac
e" }

> Object.keys(data).filter(function(key) {
...     // Don't display empty arrays.
...     return !(Array.isArray(data[key]) && data[key].length === 0);
...   }).map(function(key) {
...     var val = data[key];
...     return '--' + (val === false ? 'no-' : '') + key +
...       (typeof val === 'boolean' ? '' : '=' + val);
...   });

[ '--force',
  '--no-write',
  '--target=dev',
  '--filter=isFile',
  '--cwd=/usr/workspace' ]
>
```

简言之，该方法就是遍历`data`变量中的各自有属性并且将对应的属性值格式化后以数组形式返回。

这里的格式化规则如下：将属性值为false的属性，按照'--no-属性名称'返回；将属性值为true的属性，按照'--属性名称'返回；将属性值不为boolean值也不是空数组的属性，按照'--属性值名称=属性值'返回。

[underscore]: http://underscorejs.org/docs/underscore.html "underscore"

---
layout : post
category : 源码分析
title : grunt-file 源码分析
tags : [Grunt, 源码分析]
---
{% include JB/setup %}

## grunt内部调用模块

```javascript
var grunt = require('../grunt');
```
有很多同学不知道怎么调用自己本地的模块，其实很简单，用`require`来引用模块相对路径或者绝对路径即可。

一般情况下，require方法接受以下几种参数的传递：

* 原生模块：http，fs，path等

* 相对路径的文件模块： ./mod 或者 ../mod

* 绝对路径的文件模块: /pathToModule/mod

* 通过npm install的非原生模块：glob，js-yaml等

更多深入内容可参考一篇专门介绍Node.js的模块机制的[大神旧文](http://www.infoq.com/cn/articles/nodejs-module-mechanism)。

## node.js内部调用模块

```javascript
var fs = require('fs');
var path = require('path');
```
* [path][]模块包含了一系列的处理和转换文件路径的工具方法，实际上几乎所有的方法都在是做字符串转换，而不检查路径是否真实有效。

* [fs][]模块通过简单封装标准的[posix][]函数来提供文件i/o方法。所有的方法都有同步和异步两种方式。

## 外部调用模块：

```javascript
file.glob = require('glob');
file.minimatch = require('minimatch');
file.findup = require('findup-sync');
var YAML = require('js-yaml');
var rimraf = require('rimraf');
var iconv = require('iconv-lite');
```
* [glob][]模块使用shell中的模式，利用\* 号之类的符号来进行文件匹配。
它是一个纯javascript的通配符实现，使用[minimatch][]库来完成匹配。

* [minimatch][]是一个袖珍的通配符匹配模块，[npm][]的内部匹配就是用它来完成的。它工作的原理是将通配符表达式转换为javascript的`regexp`正则表达式对象。

* [findup-sync][]用来在当前目录或者最近的父目录中发现匹配指定模式的第一个文件。

* [js-yaml][]是[yaml][]的解析器和javascript序列化工具，支持[yaml][]的1.2规范。

* [rimraf][]是转为nodejs打造的强制删除`rm -rf`。

* [iconv-lite][]是一个纯javascript的字符转换工具，广泛的在[grunt][]和[yeoman][]等项目中使用。


## 模块内部代码解析

### 变量win32
因为众所周知的原因，模块中第一句有实际意义的代码就是判断当前平台是否为windows，因为[process][]全局对象提供了`platform`属性，所以这里只是做了简单判断，如果是windows平台则返回`true`否则返回`false`。

```javascript
var win32 = process.platform === 'win32';
```

### 内部函数unixifypath
接下来的代码还是为了处理windows与\*nix的路径分隔符问题，如果是\*nix则直接返回参数路径，否则使用正则替换所有的`\`为'/'。（虽然地球人都知道，还是啰嗦一下，文件分隔符在\*nix系统中为`/`，但是在windows中是`\`）

```javascript
var unixifypath = function(filepath) {
  if (win32) {
    return filepath.replace(/\\/g, '/');
  } else {
    return filepath;
  }
};
```

### file.setbase
这是源码中出现的第一个接口函数，不得不说这短短的几行代码，可真是一字千金，通过封装node.js中全局对象[process][]的`chdir(directory)`[方法](http://nodejs.org/api/process.html#process_process_chdir_directory)来改变当前进程的工作目录。

___note: 这里的[process][]全局对象允许你获得或者修改当前node进程的设置。这里提到的当前进程的工作目录指的是`process.cwd()’，通常情况下（如果没有调用过process.chdir()）,当前工作目录即是你调用node'命令的目录。还有一点需要注意的是，一旦改变了当前工作目录，如果你们在改动之前将其赋值给其他变量的话，你就再也得不到那个初始的工作目录了。___

```javascript
file.setbase = function() {
  //调用path.join将参数列表中的内容组合在一起，返回一个结果路径
  var dirpath = path.join.apply(path, arguments);
  process.chdir(dirpath);
};
```

需要注意的有两点：1.`path.join([path1], [path2], [...])`的参数必须是字符串；2.通过`Function.protorype.apply()`方式来处理在这里是最方便的，这样可以直接将`setbase`的参数对象指定给`path.join`，而用不着去循环参数数组。更详细的说明可以参考[这里](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply)。

相同的用法如下示：

```javascript
/* min/max number in an array */
var numbers = [5, 6, 2, 3, 7];

/* using Math.min/Math.max apply */
var max = Math.max.apply(null, numbers); /* This about equal to Math.max(numbers[0], ...) 
                                            or Math.max(5, 6, ..) */
var min = Math.min.apply(null, numbers);

/* vs. simple loop based algorithm */
max = -Infinity, min = +Infinity;

for (var i = 0; i < numbers.length; i++) {
  if (numbers[i] > max)
    max = numbers[i];
  if (numbers[i] < min) 
    min = numbers[i];
}
```

### 内部函数processPatterns
用自定义函数`fn`处理那些指定的通配符表达式或者文件名称，然后将得到的非重结果集返回。

为了能更好的理解这段源码，将这个函数中用到的几个关键函数略做介绍：
[\_.flatten](http://underscorejs.org/#flatten)用来将任意深度的内嵌的数组拉平到一维数组；[\_.difference](http://underscorejs.org/#difference)将在第一个参数数组中但不在第二个参数数组中的值以数组形式返回，即两个参数数组求差集；[\_.union](http://underscorejs.org/#union)顾名思义是将参数数组中的所有值去重后以数组形式返回，即两个参数数组求合集。以上这三个函数都来自[underscore](http://underscorejs.org)，感兴趣的同学可以自行查看。


下面我们来分析这个函数，首先，定义`result`数组变量作为函数的返回值。

```javascript
var processPatterns = function(patterns, fn) {
  // Filepaths to return.
  var result = [];
```

然后，通过`_.flatten`将`pattens`中的内容都处理为一维数组然后循环处理，

```javascript
  // Iterate over flattened patterns array.
  grunt.util._.flatten(patterns).forEach(function(pattern) {

```
通过判断每一个`pattern`项是否以"!"开头，如果以"!"开头则`exclusion`变量为`true`，并将`pattern`中的"!"字符去除。

```javascript
    // If the first character is ! it should be omitted
    var exclusion = pattern.indexOf('!') === 0;
    // If the pattern is an exclusion, remove the !
    if (exclusion) { pattern = pattern.slice(1); }
```

接下来通过`fn`函数处理`pattern`项并且将返回值写入`matches`变量，根据`exclusion`标记，即是否以"!"开头对`matches`进行不同的处理，如果`exclusion`标记为`true`则将`result`与`matches`求差集，否则将`matches`与`result`求并集。

```javascript
    // Find all matching files for this pattern.
    var matches = fn(pattern);
    if (exclusion) {
      // If an exclusion, remove matching files.
      result = grunt.util._.difference(result, matches);
    } else {
      // Otherwise add matching files.
      result = grunt.util._.union(result, matches);
    }
  });
```
做完循环后，将result数组作为结果返回。

```javascript
  return result;
};
```
___Note:此函数在``file.match和``file.expand中被调用使用___


### file.match

通过调用`processPatterns`函数，将所有匹配一个或多个通配符样式或者文件路径的路径进行返回。

源码中使用了`grunt.util.kindOf`方法对`options`参数的类型进行了判断，当`options`参数不是一个对象时，将各参数值进行位移，保证`patterns`和`filepaths`参数值有效，并将`options`赋予一个空对象。

```javascript
file.match = function(options, patterns, filepaths) {
  if (grunt.util.kindOf(options) !== 'object') {
    filepaths = patterns;
    patterns = options;
    options = {};
  }
```
如果`patterns`和`filepaths`都为`null`的话，直接返回空数组。

```javascript
  // Return empty set if either patterns or filepaths was omitted.
  if (patterns == null || filepaths == null) { return []; }
```

当`patterns`或者`filepaths`不是数组时，将它们各自包装为一个长度为1的数组。

```javascript
  // Normalize patterns and filepaths to arrays.
  if (!Array.isArray(patterns)) { patterns = [patterns]; }
  if (!Array.isArray(filepaths)) { filepaths = [filepaths]; }
```

当`patterns`或者`filepaths`为空数组时，返回一个空数组。

```javascript
  // Return empty set if there are no patterns or filepaths.
  if (patterns.length === 0 || filepaths.length === 0) { return []; }
```
如果不符合上述种种的条件检测，接下来就会调用`processPatterns`函数，并将结果返回。

```javascript
  // Return all matching filepaths.
  return processPatterns(patterns, function(pattern) {
    return file.minimatch.match(filepaths, pattern, options);
  });
};
```
这里特别要注意的是参数列表中的这个匿名函数，虽然它只是对[minimatch][]模块中的`match(list, pattern, options)`方法进行了简单封装，但是要想使用好`file.match`方法，必须对`options`中的各种可能项及其对返回结果的影响有充分的了解，建议仔细查看[minimatch][]中对`options`的详细说明。

### file.isMatch

通过调用并判断`file.match`方法的返回结果来判断文件路径是否匹配一个或者多个通配符表达式，需要注意的是只要匹配通配符表达式中的任意一个，结果即为`true`。

```javascript
/ Match a filepath or filepaths against one or more wildcard patterns. Returns
// true if any of the patterns match.
file.isMatch = function() {
  return file.match.apply(file, arguments).length > 0;
};
```
这里唯一需要注意的是，参数格式需要与`file.match`保持一致，如果想要使用各种高大上的`options`选项的话，请仔细阅读[minimatch][]中各种说明。


[glob]: https://github.com/isaacs/node-glob "node-glob"
[minimatch]: https://github.com/isaacs/minimatch "minimatch"
[npm]: http://npmjs.org "npm"
[findup-sync]: https://github.com/cowboy/node-findup-sync "node-findup-sync"
[js-yaml]: https://github.com/nodeca/js-yaml "js-yaml"
[yaml]: http://zh.wikipedia.org/zh-cn/yaml "yaml"
[rimraf]: https://github.com/isaacs/rimraf "rimraf"
[iconv-lite]: https://github.com/ashtuchkin/iconv-lite "iconv-lite"
[grunt]: http://gruntjs.com "grunt"
[yeoman]: http://yeoman.io "yeoman"
[path]: http://nodejs.org/api/path.html "path"
[fs]: http://nodejs.org/api/fs.html "file system"
[posix]: http://zh.wikipedia.org/wiki/posix "可移植操作系统接口"
[process]: http://nodejs.org/api/process.html "process"



##参考资料

* [node.js入门 - 12.api：进程（process）](http://www.cnblogs.com/softlover/archive/2012/10/03/2707139.html)



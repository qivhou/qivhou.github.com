---
layout : post
category : deep-into-grunt
title : grunt.file 源码分析
tags : [Grunt, 源码分析]
---
{% include JB/setup %}

源码路径：https://github.com/gruntjs/grunt/blob/master/lib/grunt/file.js

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

___Note: 这里的[process][]全局对象允许你获得或者修改当前node进程的设置。这里提到的当前进程的工作目录指的是`process.cwd()’，通常情况下（如果没有调用过process.chdir()）,当前工作目录即是你调用node'命令的目录。还有一点需要注意的是，一旦改变了当前工作目录，如果你们在改动之前将其赋值给其他变量的话，你就再也得不到那个初始的工作目录了。___

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
___Note:此函数在`file.match`和`file.expand`中被调用使用___


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
// Match a filepath or filepaths against one or more wildcard patterns. Returns
// true if any of the patterns match.
file.isMatch = function() {
  return file.match.apply(file, arguments).length > 0;
};
```
这里唯一需要注意的是，参数格式需要与`file.match`保持一致，如果想要使用各种高大上的`options`选项的话，请仔细阅读[minimatch][]中各种说明。

### file.expand

这个方法通过调用`processPatterns`函数，以数组形式返回所有匹配指定的通配符表达式的文件路径。

接下来分析这个方法的源码，这里用到了`grunt.util`中的`toArray`方法，其实这个`toArray`方法只是封装了[underscore][]的`toArray`将参数对象`arguments`转换为一个数组对象。

```javascript
// Return an array of all file paths that match the given wildcard patterns.
file.expand = function() {
  var args = grunt.util.toArray(arguments);
```
因为这里只是用来处理`arguments`，如果不借助任何类库的话，这段代码可以写成下面这样：

```javascript
// Return an array of all file paths that match the given wildcard patterns.
file.expand = function() {
  var args = Array.prototype.slice.call(arguments);
```

接下来使用`grunt.util.kindOf`方法对第一个传入参数`args[0]`进行了判断，第一个参数是`object`类型的话，通过`Array.prototype.shift`方法将第一个参数赋值给`options`变量，并从`args`数组中去除了第一个参数值，否则的话，我们给`options`变量赋值一个空的对象`{}`。

```javascript
  // If the first argument is an options object, save those options to pass
  // into the file.glob.sync method.
  var options = grunt.util.kindOf(args[0]) === 'object' ? args.shift() : {};
```

如果在之前不对`arguments`进行数组化处理的话，其实也是可以的，我们可以像下面这样实现它：

```javascript
  // If the first argument is an options object, save those options to pass
  // into the file.glob.sync method.
  var options = grunt.util.kindOf(arguments[0]) === 'object' ? Array.prototype.slice.call(arguments, 0, 1) : {};
```

需要注意的是我们将`arguments`中除第一个元素以外剩下的其他元素都赋值给了`options`，但是`arguments`对象本身没有发生变化。如果我们想要获取除第一个元素外其他元素的话，我们可以使用`Array.prototype.slice.call(arguments, 1)`，但是这样的话我们至少需要多一行代码去赋值或者处理，比直接使用`Array.prototype.shift`略为麻烦。

实际上，将[arguments][]这个Array-like的对象转换为数组的情况可以经常看到，这样做的最大好处就是可以利用丰富的`Array`对象实例的方法来处理参数列表，对于[arguments][]的更多信息可以点击链接进行查看。

回到源码，如果`file.expand`只接受了一个参数的话，`args`还是那个`args`，我们可以将它看做是数组化了的`arguments`，代码逻辑也很简单。但是如果`file.expand`接受了两个或者更多参数的话，世界都变了。如果`arguments[1]`是一个数组的话，就把`arguments[1]`赋值给`patterns`变量，否则的话将除`arguments[0]`以外的元素数组赋值给`patterns`比昂梁。这里之所以没有像源码中那样使用`args`变量来说事，是因为经过上一行的`args.shift()`处理之后，`args`中只剩下了第一个参数以外的其他元素。

```javascript
  // Use the first argument if it's an Array, otherwise convert the arguments
  // object to an array and use that.
  var patterns = Array.isArray(args[0]) ? args[0] : args;
```
这里还有一点需要注意的就是`Array.isArray`，一般情况下如果`Array.isArray`不是原生支持的话，我们可以通过如下方法添加这个方法，当然这个在Grunt中是不存在问题的。

```javascript
if(!Array.isArray) {
  Array.isArray = function(arg) {
    return Object.prototype.toString.call(arg) === '[object Array]';
  };
}
```
接下来又是无穷无尽的域值判断，如果`patterns`数组长度为0的话直接返回空数组。

```javascript
  // Return empty set if there are no patterns or filepaths.
  if (patterns.length === 0) { return []; }
```

`processPatterns`这个老朋友又来了，不过与`file.match`中不同只是那个匿名函数，这次用到了`glob`模块的`glob.sync(pattern, [options])`方法来获得匹配的数组。话说`glob`模块中的同步匹配和异步匹配其实只差一个回调函数参数。当然要想玩转`glob`
模块的话，也有相当多的`options`选项需要熟悉，在使用`file.expand`之前切记看看[glob][]中的`Options`说明。

```javascript
  // Return all matching filepaths.
  var matches = processPatterns(patterns, function(pattern) {
    // Find all matching files for this pattern.
    return file.glob.sync(pattern, options);
  });

```

如果`options`中的`filter`选项为`true`的话，通过自定义的filter方法对从`processPatterns`返回的结果集进行过滤，否则的话直接将`processPatterns`结果集进行返回。

```javascript
  // Filter result set?
  if (options.filter) {
    matches = matches.filter(function(filepath) {

```

这里用到了`options.cwd`，顾名思义这个用来指定当前工作目录，并且通过`path.join`来组合文件路径。如果`options`中没有`cwd`的key值或者`cwd`的值为undifned的话，直接使用`filepath`。

```javascript
      filepath = path.join(options.cwd || '', filepath);
```

这是一个神奇的`options`, 在使用`file.expand`方法时，我们可以在`options`参数中指定`filter`函数，通过指定的`filter`函数来达到过滤的目的。当然如果指定的`filter`值不是一个函数时，直接通过[fs][]模块的`statSync`方法获得指定路径的`fs.Stats`对象，通过`options.filter`中传入的有效的[fs.Stats](http://nodejs.org/docs/latest/api/fs.html#fs_class_fs_stats)方法名来返回`true`或者`false`来完成过滤。

`fs.Stats`的合法方法名有如下这些：

* stats.isFile()

* stats.isDirectory()

* stats.isBlockDevice()

* stats.isCharacterDevice()

* stats.isSymbolicLink() (only valid with fs.lstat())

* stats.isFIFO()

* stats.isSocket()

所以，你可以通过在`options`中传入`{filter : "isFile"}`来只获得匹配的文件列表，或者传入`{filter: "isDirectory"}`来只获得匹配的目录列表。

如果无法获得有效的`fs.Stats`对象或者`options.filter`既不是一个函数也不是一个有效的[fs.Stats](http://nodejs.org/docs/latest/api/fs.html#fs_class_fs_stats)方法名的话，那么只能抛出异常被catch捕获进而返回`false`。等整个`matches`数组中的项目都经过过滤后，将`Array.prototype.filter`方法返回的新生成的数组作为结果返回。

对于[fs.Stats](http://nodejs.org/docs/latest/api/fs.html#fs_class_fs_stats)方法名

```javascript
      try {
        if (typeof options.filter === 'function') {
          return options.filter(filepath);
        } else {
          // If the file is of the right type and exists, this should work.
          return fs.statSync(filepath)[options.filter]();
        }
      } catch(e) {
        // Otherwise, it's probably not the right type.
        return false;
      }
    });
  }
  return matches;
};
```

`pathSeparatorRe`变量的正则表达式全局匹配`/`和`\`。

```javascript
var pathSeparatorRe = /[\/\\]/g;
```

`extDotRe`字变量中的`first`和`last`正则分别匹配第一个`.`号后的内容和最后一个`.`号后的内容。
```javascript
// The "ext" option refers to either everything after the first dot (default)
// or everything after the last dot.
var extDotRe = {
  first: /(\.[^\/]*)?$/,
  last: /(\.[^\/\.]*)?$/,
};
```
参见下面的这个例子，可能更好理解这个正则：

```javascript
> var extDotRe = {
...   first: /(\.[^\/]*)?$/,
...   last: /(\.[^\/\.]*)?$/,
... };

> var path = "c:\\aaa\\bbb\\abc.xyz.opq"

> path.match(extDotRe.first)
[ '.xyz.opq',
  '.xyz.opq',
  index: 14,
  input: 'c:\\aaa\\bbb\\abc.xyz.opq' ]

> path.match(extDotRe.last)
[ '.opq',
  '.opq',
  index: 18,
  input: 'c:\\aaa\\bbb\\abc.xyz.opq' ]

```
### file.expandMapping

该方法返回一个`src-dest`文件映射对象的数组。

该方法接受3个参数，分别是通配符表达式`patterns`，目标文件基础路径`destBase`以及配置选项`options`。

```javascript
// Build a multi task "files" object dynamically.
file.expandMapping = function(patterns, destBase, options) {
```
首先，通过[underscore][]的`_.defaults`[方法](http://underscorejs.org/#defaults)用`options`参数以及带有`extDot`和`rename`的字变量对象生成了一个新的`options`对象。如果在参数`options`对象中没有`extDot`和`rename`的属性，那么这两个属性将直接赋值给`options`变量，否则，将优先将参数`options`对象中的`extDot`和`rename`属性赋值给`options`变量。

这里的`rename`函数接受`destBase`和`destPath`两个参数，将通过`path.join`方法将这两个参数连接并返回结果。

```javascript
  options = grunt.util._.defaults({}, options, {
    extDot: 'first',
    rename: function(destBase, destPath) {
      return path.join(destBase || '', destPath);
    }
  });
```

定义`files`数组，以及`fileByDest`对象。并使用`patterns`和`options`变量调用`file.expand`方法得到匹配通配符表达式的文件目录数组，然后用`forEach`进行遍历操作。

```javascript
  var files = [];
  var fileByDest = {};
  // Find all files matching pattern, using passed-in options.
  file.expand(options, patterns).forEach(function(src) {
```
将数组项中路径值传递给`destPath`变量，如果在`options`对象中定义了`flatten`字段，则通过`path.basename`[方法](http://nodejs.org/api/path.html#path_path_basename_p_ext)获得`destPath`的最后一部分，并重新赋值。

如果`options`变量中有`ext`字段的话，将`destPath`变量中匹配`extDotRe[options.extDot]`正则表达式的部分替换为`options.ext`的属性值，并将替换后的结果重新赋值给`destPath`。

```javascript
    var destPath = src;
    // Flatten?
    if (options.flatten) {
      destPath = path.basename(destPath);
    }
    // Change the extension?
    if ('ext' in options) {
      destPath = destPath.replace(extDotRe[options.extDot], options.ext);
    }
```
通过`options.rename`方法，结合`destBase`，`destPath`以及`options`获得目标文件的路径值，并赋值给`dest`变量。

如果在`options`中有`cwd`字段且不为`undefined`，将`options.cwd`作为根路径拼接`src`变量值，并重新将值赋值给`src`变量。

紧接着，将处理完毕的`dest`和`src`中的文件分割符用正则`pathSeparatorRe`统一替换为`/`。

```javascript
    // Generate destination filename.
    var dest = options.rename(destBase, destPath, options);
    // Prepend cwd to src path if necessary.
    if (options.cwd) { src = path.join(options.cwd, src); }
    // Normalize filepaths to be unix-style.
    dest = dest.replace(pathSeparatorRe, '/');
    src = src.replace(pathSeparatorRe, '/');
```

如果`fileByDest`对象中不存在名为`dest`变量值的属性，则将`{src: [src], dest: dest}`对象压入`files`数组，并且将这个对象赋值给`fileByDest`变量的`dest`属性。如果存在的话，将当前的`src`变量值压入`fileByDest`对象中的`dest`变量值中的src数组。

```javascript
    // Map correct src path to dest path.
    if (fileByDest[dest]) {
      // If dest already exists, push this src onto that dest's src array.
      fileByDest[dest].src.push(src);
    } else {
      // Otherwise create a new src-dest file mapping object.
      files.push({
        src: [src],
        dest: dest,
      });
      // And store a reference for later use.
      fileByDest[dest] = files[files.length - 1];
    }
  });
  return files;
};
```

### file.mkdir

此方法的作用与`mkdir -p`一样，创建一个目录以及任何中间目录。（这里的中间目录指的是目标目录的父目录，例如：想要创建`/foo/bar/tmp/`这个目录，`/foo/bar/`这两级目录就是这里的中间目录）

___不熟悉[mkdir命令](http://linuxmanpages.com/man1/mkdir.1.php)的同学可以猛击[这里](http://linuxmanpages.com/man1/mkdir.1.php)看看这个模仿秀的本尊是什么样。___

查看源码我们会发现，这个方法接受两个参数，`dirpat`为目录的路径，`mode`为目录的预设访问权限。另外，如果全局的`grunt`对象中的`option`设置了`write`为`false`，那么这个方法将什么也不做直接返回。

```javascript
// Like mkdir -p. Create a directory and any intermediary directories.
file.mkdir = function(dirpath, mode) {
  if (grunt.option('no-write')) { return; }
```

如果调用时仅传入了`dirpath`，那么`mode`值其实为`undefined`，所以在下面语句中会重新给`mode`赋值。

```javascript
  // Set directory mode in a strict-mode-friendly way.
  if (mode == null) {
    mode = parseInt('0777', 8) & (~process.umask());
  }
```

通过调用[parseInt](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/parseInt)方法将8进制的`0777`转化为511，然后将与`(~process.umask())`进行位运算后的结果赋值给`mode`。

需要注意的是这里的[process.umask](http://nodejs.org/api/process.html#process_process_umask_mask)，它用来修改或者读取当前进程创建文件和文件夹的预设权限。因为文件系统和权限控制的不同，在`win32`平台上，`process.umask()`的值是0，即使做了`process.umask(022)`之类的设置，在使用时也只能取到0，所以在`win32`平台上，`mode`值就是511（8进制的777），而在\*nux系统中，`mode`需要通过计算得出，如果`process.umask`为002的话，通过计算`mode`的10进制值为509（8进制的775），如果`process.umask`为022的话，通过计算`mode`的10进制值为493（8进制的755），对于这里`mode`的计算方法，相信看了鸟哥对于`umask`的[说明](http://linux.vbird.org/linux_basic/0220filemanager.php#umask)就一下明白了，这里不在赘述。

接下来，通过使用`String.prototype.split()`[方法](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/split)把`pathSeparatorRe`正则表达式作为分隔符将传入的`dirpath`进行分割生成一个目录数组，紧接着通过链式操作调用`Array.prototype.reduce()`[方法](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce)，遍历操作目录数组中的元素。

```javascript
  dirpath.split(pathSeparatorRe).reduce(function(parts, part) {
```

在`reduce`方法中的回调函数其实很简单，`reduce`方法有两个参数，第一个是用来遍历处理数组项目的回调函数，第二个是第一次调用这个回调函数时传入的第一个参数（在这里这个值是''）。

这个回调函数内部逻辑很清晰：依次将目录数组中的项目进行拼接，然后通过`path.resolve`方法处理每次拼接完成的目录得到一个绝对路径并赋值给`subpath`变量，

```javascript
    parts += part + '/';
    var subpath = path.resolve(parts);
```
再通过调用`file.exists`判断当前目录是否存在，如果不存在的话则通过`fs.mkdirSync`方法创建权限为`mode`值的`subpath`目录，为了更友好的应对各种不可预见的情况，这里通过try...catch对可能发生的异常进行了捕获处理。

```javascript
    if (!file.exists(subpath)) {
      try {
        fs.mkdirSync(subpath, mode);
      } catch(e) {
        throw grunt.util.error('Unable to create directory "' + subpath + '" (Error code: ' + e.code + ').', e);
      }
    }
```

函数最后返回`parts`变量，用来给下一个项目拼接目录路径。通过这样处理之后，所有层级的目录，如果在执行前不存在的话，都会在这个过程中被创建。

```javascript
    return parts;
  }, '');
};
```

### file.recurse

见名知意，这个方法使用来递归处理指定目录的文件。方法接受三个参数，分别是根目录`rootdir`，处理文件的回调函数`callback`，以及指定的子目录`subdir`（用了`rootdir`和`subdir`两个参数来确定目录，一下让这个方法变得异常灵活了有木有？）

```javascript
// Recurse into a directory, executing callback for each file.
file.recurse = function recurse(rootdir, callback, subdir) {
```
如果指定了`subdir`参数并且不为''，那么通过`path.join`[方法](http://nodejs.org/api/path.html#path_path_join_path1_path2)对`rootdir`和`subdir`进行拼接并将值赋给`abspath`变量，否则的话直接将`rootdir`参数值赋给`abspath`变量。

```javascript
  var abspath = subdir ? path.join(rootdir, subdir) : rootdir;
```
通过`fs.readdirSync`[方法](http://nodejs.org/api/fs.html#fs_fs_readdirsync_path)得到`abspath`目录下的文件名数组（注意的是这个数组中不包含'.'还有'..'）,并且通过链式操作队数组中的每个元素进行处理，

```javascript
  fs.readdirSync(abspath).forEach(function(filename) {
```
首先，将目标目录`abspath`与文件名`filename`通过`path.join`[方法](http://nodejs.org/api/path.html#path_path_join_path1_path2)进行拼接，并将拼接后的文件绝对路径赋值给`filepath`变量，

```javascript
    var filepath = path.join(abspath, filename);
```

接下来通过`fs.statSync`[方法](http://nodejs.org/api/fs.html#fs_fs_statsync_path)得到当前路径的`fs.Stats`对象实例，并通过`isDirectory()`判断当前路径是否为一个目录。如果是一个目录的话，递归调用`recurse`方法，否则调用传入的回调函数`callback`。

```javascript
    if (fs.statSync(filepath).isDirectory()) {
      recurse(rootdir, callback, unixifyPath(path.join(subdir || '', filename || '')));
    } else {
      callback(unixifyPath(filepath), rootdir, subdir, filename);
    }
  });
};

```

在递归调用`recurse`时，用到了`unixifyPath`函数来处理路径中的分隔符，`rootdir`和`callback`参数内容及位置不变，只是针对第三个参数`subdir`进行了拼接处理。

Note: 这里的`subdir || ''`这种写法很常见，这个表达式表示：如果`subdir`不是`undefined`且不为空，则值为`subdir`否则值为''。类似下面这种三元运算符表达式，不过更精炼。

```javascript
    subdir ? subdir : "";
```
在直接调用`callback`函数时，4个参数的顺序分别为:具体文件路径`filepath`，根目录参数`rootdir`，子目录参数`subdir`以及具体文件名`filename`,在构造自己的回调函数时需要注意这个顺序及参数意义。


### file.defaultEncoding

默认字符编码为utf8，可以通过该接口进行改变。

```javascript
// The default file encoding to use.
file.defaultEncoding = 'utf8';
```

### file.preserveBOM

这里的BOM指的是[BOM（Byte Order Mark）][]，这个标记指定了在`file.read`过程中保留字节顺序标记还是遗弃它，因为默认值为`false`所以默认为遗弃。

```javascript
// Whether to preserve the BOM on file.read rather than strip it.
file.preserveBOM = false;
```
### file.read

读取并返回指定文件的内容。

方法有两个参数，`filepath`指定文件路径，`options`中包含相关配置。

```javascript
// Read a file, return its contents.
file.read = function(filepath, options) {
```
如果`options`参数在调用过程中被省略，则将一个空对象赋给`options`。

```javascript
  if (!options) { options = {}; }
```
接下来，定义了`contents`变量用来存放文件内容。并且通过`grunt.verbose`的API来输出当前状态。

```javascript
  var contents;
  grunt.verbose.write('Reading ' + filepath + '...');
```

___Note:这里的`grunt.verbose`实际上就是`grunt.log.verbose`，我们将在`grunt.log`
的源码分析中，对它进行分析。简言之就是当`grunt`运行时制定了`-v`或者`--verbose`参数时，此类日志就会输出到console。___

然后，通过`fs.readFileSync`方法来读取指定路径的文件内容，并将值赋予`contents`变量。

至于这里为什么要用`String`来进行字符串转换，细心的读者可能已经发现了在整个方法中，虽然对`options`参数进行了校验，但是并没有对`filepath`参数进行校验，这里使用`String`作为一种更安全的`toString`选择，虽然它任然调用了底层的`toString`方法，但是它也适用于`null`和`undefined`，所以如果`filepath`为undefined的话，异常将由`fs.readFileSync`抛出，然后统一在`catch`中进行捕获处理。

```javascript
  try {
    contents = fs.readFileSync(String(filepath));
```
如果`options.encoding`没有明确的指定为`null`的话，使用[iconv][]模块的`decode`方法对`contents`进行解码操作，如果`options.encoding`没有指定的话就是用全局的`file.defaultEncoding`值作为`decode`方法的编码参数。

```javascript
    // If encoding is not explicitly null, convert from encoded buffer to a
    // string. If no encoding was specified, use the default.
    if (options.encoding !== null) {
      contents = iconv.decode(contents, options.encoding || file.defaultEncoding);
```
___Note:注意这里的`options.encoding !== null`，因为这里用的是严格判断，因为`Null`和`Undefined`类型是`==`而不是`===`，所以如果`options.encoding`为`undefined`的话将依然因为满足条件进入这个代码块。___

接下来用到了上面提到的`file.preserveBOM`标记，如果该标记值为`false`并且`contents`变量的第一个字符是`0xFEFF`的话，通过`String.prototype.subString`[方法](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/substring)来舍弃第一个字符（`contents.charCodeAt(0)`）将之后的内容赋值给`contents`变量，达到舍弃[BOM（Byte Order Mark）][]的目的。

```javascript
      // Strip any BOM that might exist.
      if (!file.preserveBOM && contents.charCodeAt(0) === 0xFEFF) {
        contents = contents.substring(1);
      }
    }
```
当`grunt`运行时制定了`-v`或者`--verbose`参数时，输出操作完成的日志到console，并且将`contents`变量返回。如果在此过程中捕获到了异常则统一将`grunt.util.error`异常进行抛出。因为`file.read`操作通常为很关键的步骤，如果此过程失败或者异常的话，接下去的工作将无法进行，所以将异常抛出并终止`grunt`进程是可以理解的。

```javascript
    grunt.verbose.ok();
    return contents;
  } catch(e) {
    grunt.verbose.error();
    throw grunt.util.error('Unable to read "' + filepath + '" file (Error code: ' + e.code + ').', e);
  }
};
```

### file.readJSON

相比较`file.read`可以读取各种文件来说，这里对`file.read`进行了封装并且对文件内容进行JSON格式的解析，返回一个JSON对象。

与`file.read`一样，这个方法接受两个参数`filepath`和`options`。

```javascript
// Read a file, parse its contents, return an object.
file.readJSON = function(filepath, options) {
```
直接将参数传入`file.read`得到结果并赋值给`src`变量。

```javascript
  var src = file.read(filepath, options);
```
定义`result`变量，来存放JSON对象。并且调用`grunt.verbose`的`write`方法来输出详细日志。

```javascript
  var result;
  grunt.verbose.write('Parsing ' + filepath + '...');
```
接下来是重头戏，通过[JSON][]对象的`parse`方法对文件内容`src`进行JSON解析并将结果赋值给`result`变量。

如果这个过程中出现异常将统一在`catch`中捕获，通过`grunt.verbose.error`输出错误日志，然后通过`grunt.util.error`进行异常抛出。

```javascript
  try {
    result = JSON.parse(src);
    grunt.verbose.ok();
    return result;
  } catch(e) {
    grunt.verbose.error();
    throw grunt.util.error('Unable to parse "' + filepath + '" file (' + e.message + ').', e);
  }
};
```

### file.readYAML

读取一个[yaml][]文件，进行相应解析并返回YAML对象。

实际上这个方法和`file.readJSON`如出一辙，唯一不同的就是这里通过[js-yaml]模块的`load`方法对文件内容进行了解析。

```javascript
// Read a YAML file, parse its contents, return an object.
file.readYAML = function(filepath, options) {
  var src = file.read(filepath, options);
  var result;
  grunt.verbose.write('Parsing ' + filepath + '...');
  try {
    result = YAML.load(src);
    grunt.verbose.ok();
    return result;
  } catch(e) {
    grunt.verbose.error();
    throw grunt.util.error('Unable to parse "' + filepath + '" file (' + e.problem + ').', e);
  }
};
```

按我的理解，这里完全有必要抽象出一个`file.parse`方法如下：

```javascript
// Read a file, then parse it by your parse function, and return an relevant object.
file.parse = function(filepath, options, callback) {
  var src = file.read(filepath, options);
  var result;
  grunt.verbose.write('Parsing ' + filepath + '...');
  try {
    result = callback(src);
    grunt.verbose.ok();
    return result;
  } catch(e) {
    grunt.verbose.error();
    throw grunt.util.error('Unable to parse "' + filepath + '" file (' + e.problem + ').', e);
  }
};
```

`file.readYAML`和`file.readJSON`可以相应改写为下面这样：

```javascript
file.readJSON = function(filepath, options){
    return file.parse(filepath, options, JSON.parse);
}

file.readYAML = function(filepath, options){
    return file.parse(filepath, options, YAML.load);
}

```

### file.write

顾名思义，这个方法是用来写入文件的。

方法有三个参数，文件路径`filepath`，文件内容`contentx`以及配置选项`options`.。

```javascript
// Write a file.
file.write = function(filepath, contents, options) {
```
首先，处理`options`，这个之前已经提过，这里不再赘述。

```javascript
  if (!options) { options = {}; }
```

接下来，获取`grunt.option`中的`no-write`标记，如果在调用`grunt`时使用了`--write`命令行选项，则这里的`nowrite`变量值将为`true`，否则的话为`false`。

```javascript
  var nowrite = grunt.option('no-write');
```
然后，通过三元表达式，来输出正确的详细日志信息，如果`nowrite`为`true`则输出`Not actually writing ....`，否则的话则输出`Wrting...`。

```javascript
  grunt.verbose.write((nowrite ? 'Not actually writing ' : 'Writing ') + filepath + '...');
```

接下来，通过调用`file.mkdir`创建必需的目录结构，

```javascript
  // Create path, if necessary.
  file.mkdir(path.dirname(filepath));
```
然后，用`Buffer.isBuffer`[方法](http://nodejs.org/api/buffer.html#buffer_class_method_buffer_isbuffer_obj)对文件内容参数`contents`进行判断，如果不是一个`Buffer`实例的话，则通过[iconv][]模块的`encode`方法对`contents`变量进行编码操作，编码格式优先使用`options`中指定的`encoding`值，如果没有进行特殊指定的话则使用`file.defaultEncoding`值，这个值默认为`utf8`。

```javascript
  try {
    // If contents is already a Buffer, don't try to encode it. If no encoding
    // was specified, use the default.
    if (!Buffer.isBuffer(contents)) {
      contents = iconv.encode(contents, options.encoding || file.defaultEncoding);
    }
```
接下来的工作就简单了，如果`nowrite`变量值为`false`则通过`fs.writeFileSync`方法实际写入`contents`变量内容到`filepath`文件中，通过`grunt.verbose.ok`方法写出操作成功的日志信息，并且返回`true`。

```javascript
    // Actually write file.
    if (!nowrite) {
      fs.writeFileSync(filepath, contents);
    }
    grunt.verbose.ok();
    return true;
```
如果在这个过程中出现了任何异常的话，异常将被捕获并通过`grunt.verbose.error`方法写出操作失败的日志信息，最后通过`grunt.util.error`将异常抛出，终止当前的`grunt`进程。

```javascript
  } catch(e) {
    grunt.verbose.error();
    throw grunt.util.error('Unable to write "' + filepath + '" file (Error code: ' + e.code + ').', e);
  }
};
```

### file.copy

通过读取一个文件，并处理文件内容（可选的），最终将内容输出到指定的路径。

该方法有三个参数，分别为源文件路径`srcpath`，目标文件路径`destpath`，以及配置选项`options`。

```javascript
// Read a file, optionally processing its content, then write the output.
file.copy = function(srcpath, destpath, options) {
```
又是处理`options`，略去不表。
```javascript
  if (!options) { options = {}; }
```

如果在`options`对象中指定了一个`process`函数，并且`options`中的`noProcess`值不为`true`，并且在`options`中的`noProcess`中指定的通配符模式并不匹配当前的源文件路径`srcpath`，满足了上述种种条件的话，这个`process`变量值就为`true`，如果有一个条件不满足的话则为`false`。

```javascript
  // If a process function was specified, and noProcess isn't true or doesn't
  // match the srcpath, process the file's source.
  var process = options.process && options.noProcess !== true &&
    !(options.noProcess && file.isMatch(options.noProcess, srcpath));
```
如果`process`变量为`true`的话，则将`options`变量赋值给`readWriteOptions`变量，否则的话，将`encoding`值为`null`的字变量对象赋值给`readWriteOptions`。

然后，通过`file.read`方法读取文件路径为`srcpath`的文件内容并赋值给`contents`变量。需要注意的是当`encoding`值为`null`的时候，`contents`变量为一个字符串对象，否则的话，为一个`Buffer`对象。为什么呢？猛戳[这里](http://nodejs.org/api/fs.html#fs_fs_readfilesync_filename_options)，因为`file.read`方法实际上调用了`fs.readFileSync`方法来读取文件内容，所以这就很容易理解了。

```javascript
  // If the file will be processed, use the encoding as-specified. Otherwise,
  // use an encoding of null to force the file to be read/written as a Buffer.
  var readWriteOptions = process ? options : {encoding: null};
  // Actually read the file.
  var contents = file.read(srcpath, readWriteOptions);
```
如果`process`变量为`true`的话，我们会通过`options.process`方法对`contents`内容进行处理，这里传递给`process`方法的第一个参数是当前文件内容`contents`变量，第二个参数是源文件路径`srcpath`变量。如果在处理过程中有异常发生，异常将被捕获并通过`grunt.util.error`规范处理后统一抛出。

```javascript
  if (process) {
    grunt.verbose.write('Processing source...');
    try {
      contents = options.process(contents, srcpath);
      grunt.verbose.ok();
    } catch(e) {
      grunt.verbose.error();
      throw grunt.util.error('Error while processing "' + srcpath + '" file.', e);
    }
  }
```
如果通过处理后的`contents`值为`false`，将取消文件拷贝操作，否则的话通过`file.write`方法将文件内容写入`destpath`文件中。

```javascript
  // Abort copy if the process function returns false.
  if (contents === false) {
    grunt.verbose.writeln('Write aborted.');
  } else {
    file.write(destpath, contents, readWriteOptions);
  }
};
```

Note: 总结一下，如果想要通过`options.process`方法处理源文件内容后再拷贝到目标文件中，1. 自定义的`process`方法需要接受文件内容参数以及源文件路径参数，正常情况下返回处理后的文件内容，遇有异常情况需要取消拷贝操作的话返回`false`；2. 如果有部分文件并不需要通过`process`处理就直接拷贝到目标文件目录，需要将这部分文件的通配符表达式作为值指定给`options.noProcess`
。

### file.delete

递归删除目录及文件。

方法接受两个参数，第一个为文件路径`filepath`，第二个为配置选项`options`。

```javascript
// Delete folders and files recursively
file.delete = function(filepath, options) {
```
接下来直接看源码。

首先，通过`String`将`filepath`参数进行更安全的字符串转换。

```javascript
  filepath = String(filepath);
```
接下来，获取`grunt.option`中的`no-write`标记，如果在调用`grunt`时使用了`--no-write`命令行选项，则这里的`nowrite`变量值将为`true`，否则的话为`false`。

如果`options`变量是`undefined`的话，对`options`重新赋值，如果在调用`grunt`时使用了`--force`命令航选项，这里的`grunt.option('force')`将为`true`，则`options`变量值更改为`{force: true}`，否则的话为`{force : false}`。

```javascript
  var nowrite = grunt.option('no-write');
  if (!options) {
    options = {force: grunt.option('force') || false};
  }
```
根据`nowrite`变量的值，将`Not actually deleting ....` 或者`Deleting ...`作为详细日志输出。

```javascript
  grunt.verbose.write((nowrite ? 'Not actually deleting ' : 'Deleting ') + filepath + '...');
```
在正式开始删除操作前，对`filepath`的路径进行校验，调用`file.exists`方法检验`filepath`是否存在，如果不存在的话，通过`grunt.verbose.error`输出错误信息日志，并且通过`grunt.log.warn`
输出警告信息，最后返回`false`。

```javascript
  if (!file.exists(filepath)) {
    grunt.verbose.error();
    grunt.log.warn('Cannot delete nonexistent file.');
    return false;
  }
```
如果`filepath`通过了检验，但是`options.force`值为`false`的话，这意味着我们不能强制删除以下两种情况下的文件和目录：

* `filepath`是当前的工作目录CWD。例如，当前工作目录是`/grunt/workspace/`，这里的`filepath
`值也是`/grunt/workspace/`。

* `filepath`在当前工作目录CWD以外的目录中。例如，当前工作目录是`/grunt/workspace/`，但是`filepath`值却是`/gulp/workspace/`。

在遇到以上情况时，通过`grunt.verbose.error`输出错误信息，并且通过`grunt.fail.warn`方法输出相应的警告信息，最后返回`false`。

```javascript
  // Only delete cwd or outside cwd if --force enabled. Be careful, people!
  if (!options.force) {
    if (file.isPathCwd(filepath)) {
      grunt.verbose.error();
      grunt.fail.warn('Cannot delete the current working directory.');
      return false;
    } else if (!file.isPathInCwd(filepath)) {
      grunt.verbose.error();
      grunt.fail.warn('Cannot delete files outside the current working directory.');
      return false;
    }
  }
```
如果`options.force`为`true`，亦或`filepath`不在上述两种情况中，当`nowrite`变量为`false`时，通过[rimraf][]模块的`sync`方法对`filepath`进行删除，通过`grunt.verbose.ok`输出成功信息，并且返回结果`true`。

如果在这个过程中出现异常的话，异常会被捕获并且通过`grunt.util.error`对错误信息进行包装后进行抛出。

```javascript
  try {
    // Actually delete. Or not.
    if (!nowrite) {
      rimraf.sync(filepath);
    }
    grunt.verbose.ok();
    return true;
  } catch(e) {
    grunt.verbose.error();
    throw grunt.util.error('Unable to delete "' + filepath + '" file (' + e.message + ').', e);
  }
};
```
### file.exists

判断文件或目录是否存在，存在的话返回`true`。

这个方法接受一些列的参数，例如：`'/foo', 'bar', 'baz/asdf', 'quux', '..'`，因为在方法内部会通过调用`path.join`[方法](http://nodejs.org/api/path.html#path_path_join_path1_path2)连接各参数并将结果赋值给`filepath`变量。

然后，通过调用`fs.existsSync`[方法](http://nodejs.org/api/fs.html#fs_fs_existssync_path)
判断`filepath`是否存在，并将结果进行返回。

```javascript
// True if the file path exists.
file.exists = function() {
  var filepath = path.join.apply(path, arguments);
  return fs.existsSync(filepath);
};
```

### file.isLink

判定给定的路径是否表示链接，返回`true`或者`false`。

该方法与上述方法`file.exists`一样接受一些列的参数，并且调用`path.join`[方法](http://nodejs.org/api/path.html#path_path_join_path1_path2)连接各参数并将结果赋值给`filepath`变量。

然后，通过调用`file.exists`方法判断路径文件是否存在，如果不存在则返回`false`，否则，调用`fs.lstatSync`[方法](http://nodejs.org/api/fs.html#fs_fs_lstatsync_path)并对返回的`fs.Stats`[对象](http://nodejs.org/api/fs.html#fs_class_fs_stats)实例进行链式操作，将`isSymbolicLink`方法的结果进行返回。

```javascript
// True if the file is a symbolic link.
file.isLink = function() {
  var filepath = path.join.apply(path, arguments);
  return file.exists(filepath) && fs.lstatSync(filepath).isSymbolicLink();
};
```

### file.isDir

判断给定的路径是否是一个目录，返回`true`或者`false`。

该方法与上述方法`file.exists`一样接受一系列的参数，并且调用`path.join`[方法](http://nodejs.org/api/path.html#path_path_join_path1_path2)连接各参数并将结果赋值给`filepath`变量。


然后，与`file.isLink`方法处理逻辑一样，只不过最后将`isDirectory`方法的结果进行返回。

```javascript
// True if the path is a directory.
file.isDir = function() {
  var filepath = path.join.apply(path, arguments);
  return file.exists(filepath) && fs.statSync(filepath).isDirectory();
};
```

### file.isFile

判断给定的路径是否是一个文件，返回`true`或者`false`。

该方法与上述方法`file.exists`一样接受一系列的参数，并且调用`path.join`[方法](http://nodejs.org/api/path.html#path_path_join_path1_path2)连接各参数并将结果赋值给`filepath`变量。

然后，与`file.isLink`方法处理逻辑一样，只不过最后将`isFile`方法的结果进行返回。

```javascript
// True if the path is a file.
file.isFile = function() {
  var filepath = path.join.apply(path, arguments);
  return file.exists(filepath) && fs.statSync(filepath).isFile();
};
```
### file.isPathAbsolute

判断给定的路径是否是一个绝对路径，返回`true`或者`false`。

该方法与上述方法`file.exists`一样接受一系列的参数，并且调用`path.join`[方法](http://nodejs.org/api/path.html#path_path_join_path1_path2)连接各参数并将结果赋值给`filepath`变量。

该方法的逻辑很简单，将获得的`filepath`的绝对路径与`filepath`比较，如果相等就返回`true`，否则返回`false`。

```javascript
// Is a given file path absolute?
file.isPathAbsolute = function() {
  var filepath = path.join.apply(path, arguments);
  return path.resolve(filepath) === filepath.replace(/[\/\\]+$/, '');
};
```

源码中通过调用`path.resolve`[方法](http://nodejs.org/api/path.html#path_path_resolve_from_to)获得`filepath`的绝对路径，并将`filepath`结尾处的路径分隔符替换删除对两者进行比较，最后将比较结果返回。

之所以要比较删除了结尾处路径分隔符的`filepath`，是因为通过`path.resolve`[方法](http://nodejs.org/api/path.html#path_path_resolve_from_to)得到的绝对路径是不带最后一个路径分隔符的。

可参看例子：

```javascript
path.resolve('/foo/bar', './baz')
// returns
'/foo/bar/baz'

path.resolve('/foo/bar', '/tmp/file/')
// returns
'/tmp/file'

path.resolve('wwwroot', 'static_files/png/', '../gif/image.gif')
// if currently in /home/myself/node, it returns
'/home/myself/node/wwwroot/static_files/gif/image.gif'

```

### file.arePathsEquivalent

判断指定的路径是否都引用的是同一路径。

该方法可以接受多个参数，内部的逻辑是获得第一个参数的绝对路径，然后依次与其他参数的绝对路径进行比较，如果在比较过程中有一个不相等，则返回`false`，否则在比较结束后返回`true`。

```javascript
// Do all the specified paths refer to the same path?
file.arePathsEquivalent = function(first) {
  first = path.resolve(first);
  for (var i = 1; i < arguments.length; i++) {
    if (first !== path.resolve(arguments[i])) { return false; }
  }
  return true;
};
```
### file.doesPathContain

判断指定的路径是否都属于指定的父路径。

___Note：这里只做判断，并不检测是否每一个路径都真实存在。___

该方法接受多个参数，这里的父路径指的是传入的第一个参数的绝对路径。

源码解析如下：

首先，通过`path.resolve`[方法](http://nodejs.org/api/path.html#path_path_resolve_from_to)获得第一个参数的绝对路径并把值赋给`ancestor`。

```javascript
// Are descendant path(s) contained within ancestor path? Note: does not test
// if paths actually exist.
file.doesPathContain = function(ancestor) {
  ancestor = path.resolve(ancestor);
```
接下来，遍历`arguments`中其他的参数，调用`path.relative`[方法](http://nodejs.org/api/path.html#path_path_relative_from_to)得到父路径与参数的绝对路径的相对路径，并将这个相对路径赋值给`relative`变量。

```javascript
  var relative;
  for (var i = 1; i < arguments.length; i++) {
    relative = path.relative(path.resolve(arguments[i]), ancestor);
```
如果`relative`变量值为''（这意味着当前参数的绝对路径与`ancestor`路径相同），或者`relative`变量值中包含字母或者数字或者下划线或者汉字（这意味着当前参数的绝对路径不被包含在`ancestor`路径中），这两种情况任一个满足，则返回`false`，否则将在`arguments`遍历结束后返回`true`。

```javascript

    if (relative === '' || /\w+/.test(relative)) { return false; }
  }
  return true;
};
```

为了更好的说明这个问题，请参看下面的例子：

```javascript
> var aa = path.resolve("c:\\aaa\\ddd\\eee")
> var bb = path.resolve("c:\\aaa\\xxx\\yyy")
> path.relative(aa, aa)
''
> path.relative(aa, bb)
'..\\..\\xxx\\yyy'
> var cc = path.resolve("c:\\aaa\\xxx\\yyy\\xxx\\x.yz")
> path.relative(cc, bb)
'..\\..'
> /\w+/.test(path.relative(cc, bb))
false
```
### file.isPathCwd

判断给定的路径是否是CWD（Current Working Directory，当前工作目录）。

该方法的逻辑如下：将`process.cwd()`和通过`fs.realpathSync`[方法](http://nodejs.org/api/fs.html#fs_fs_realpathsync_path_cache)得到的`filepath`的真实路径作为参数，调用了`file.arePathsEquivalent`方法并将结果进行返回。

```javascript
// Test to see if a filepath is the CWD.
file.isPathCwd = function() {
  var filepath = path.join.apply(path, arguments);
  try {
    return file.arePathsEquivalent(process.cwd(), fs.realpathSync(filepath));
  } catch(e) {
    return false;
  }
};
```
### file.isPathInCwd

判断给定的路径是否包含在CWD（Current Working Directory，当前工作目录）中。

该方法的逻辑非常简单，只是将`process.cwd()`和通过`fs.realpathSync`[方法](http://nodejs.org/api/fs.html#fs_fs_realpathsync_path_cache)得到的`filepath`的真实路径作为参数，调用了`file.doesPathContain`方法，并将结果进行了返回。

```javascript
// Test to see if a filepath is contained within the CWD.
file.isPathInCwd = function() {
  var filepath = path.join.apply(path, arguments);
  try {
    return file.doesPathContain(process.cwd(), fs.realpathSync(filepath));
  } catch(e) {
    return false;
  }
};
```

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
[underscore]: http://underscorejs.org "underscore"
[arguments]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions_and_function_scope/arguments "arguments"
[BOM（Byte Order Mark）]: http://zh.wikipedia.org/zh-cn/%E4%BD%8D%E5%85%83%E7%B5%84%E9%A0%86%E5%BA%8F%E8%A8%98%E8%99%9F "字节顺序标记"
[JSON]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON "JSON"

##参考资料

* [深入浅出Node.js (三)： 深入Node.js的模块机制](http://www.infoq.com/cn/articles/nodejs-module-mechanism)

* [node.js入门 - 12.api：进程（process）](http://www.cnblogs.com/softlover/archive/2012/10/03/2707139.html)



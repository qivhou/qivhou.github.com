---
layout : post
category : deep-into-grunt
title : Grunt源码解析之'/grunt/template.js'
tags : [Grunt, 源码分析, grunt.template]
---
{% include JB/setup %}

源码路径：https://github.com/gruntjs/grunt/blob/master/lib/grunt/template.js

乍一看，又是template，这年头template简直是太多太多了，从[tmpl][]到[_.template][]到[dot][]，再到国内模板引擎中的[arttemplate][]，至于如何选择可谓仁者见仁智者见智，在[jsperf][]上的各种template的性能比较，包括国内开发者提供的[引擎渲染速度竞赛](http://aui.github.io/arttemplate/test/test-speed.html)都可以作为选择的参考，甚至有[template-engine-chooser](http://garann.github.io/template-chooser/)这样的选择器帮助你去选择，如果遇到相关template选择的问题可自行查看选择。

但是grunt中的这个template其实并没有重复造轮子，它只是使用了[lo-dash][]的template engine，然后又在模块上封装了一些常用的工具方法。

言归正传，让我们开始源码解析。

## 模块内部代码解析

在模块起始处，还是先引入了`grunt`核心模块。

```javascript
var grunt = require('../grunt');
```
紧接着，定义了`template`对象变量作为模块返回的对象。


### template变量

定义`template`变量，作为该模块返回的对象。

```javascript
// the module to be exported.
var template = module.exports = {};
```
### template.date

这里引入了[dateformat][]模块，并指向`template.date`。

```javascript
// external libs.
template.date = require('dateformat');
```
[dateformat][]模块将steven levithan的[javascript date format](http://blog.stevenlevithan.com/archives/date-time-format)中提到的日期格式化函数进行了node.js化的打包封装并进行了一些修改，它提供的日期格式化功能如下示：

```javascript
var dateformat = require('dateformat');
    var now = new date();

    // basic usage
    dateformat(now, "dddd, mmmm ds, yyyy, h:mm:ss tt");
    // saturday, june 9th, 2007, 5:46:21 pm

    // you can use one of several named masks
    dateformat(now, "isodatetime");
    // 2007-06-09t17:46:21

    // ...or add your own
    dateformat.masks.hammertime = 'hh:mm! "can\'t touch this!"';
    dateformat(now, "hammertime");
    // 17:46! can't touch this!

    // when using the standalone dateformat function,
    // you can also provide the date as a string
    dateformat("jun 9 2007", "fulldate");
    // saturday, june 9, 2007

    // note that if you don't include the mask argument,
    // dateformat.masks.default is used
    dateformat(now);
    // sat jun 09 2007 17:46:21

    // and if you don't include the date argument,
    // the current date and time is used
    dateformat();
    // sat jun 09 2007 17:46:22

    // you can also skip the date argument (as long as your mask doesn't
    // contain any numbers), in which case the current date/time is used
    dateformat("longtime");
    // 5:46:22 pm est

    // and finally, you can convert local time to utc time. simply pass in
    // true as an additional argument (no argument skipping allowed in this case):
    dateformat(now, "longtime", true);
    // 10:46:21 pm utc

    // ...or add the prefix "utc:" or "gmt:" to your mask.
    dateformat(now, "utc:h:mm:ss tt z");
    // 10:46:21 pm utc

    // you can also get the iso 8601 week of the year:
    dateformat(now, "w");
    // 42

    // and also get the iso 8601 numeric representation of the day of the week:
    dateformat(now,"n");
    // 6
```
### template.today

通过将当前时间`new date()`作为`template.date`的参数并进行封装，打造了`template.today`方法。

该方法仅接受一个`format`参数，并根据不同的`format`返回指定格式的当前时间。

```javascript
// format today's date.
template.today = function(format) {
  return template.date(new date(), format);
};
```

### alldelimiters变量

作为模块内的公有变量，用来存放各种分隔符配置。

```javascript
// template delimiters.
var alldelimiters = {};
```

### template.adddelimiters

该方法接受`name`，`opener`，`closer`三个参数，用来向`alldelimiters`变量对象添加相关的分隔符配置。

```javascript
// initialize template delimiters.
template.adddelimiters = function(name, opener, closer) {
```
首先，在`alldelimiters`对象中添加值为空对象"{}"的属性`name`，并且定义变量`delimiters`指向该对象，同时为该对象添加"opener"和"closer"属性，值分别为`opener`和`closer`。

```javascript
  var delimiters = alldelimiters[name] = {};
  // used by grunt.
  delimiters.opener = opener;
  delimiters.closer = closer;
```
接下来，将`delimiters.operner`值中的每一个字符都替换为"\\\\原字符"，并将替换后的结果赋值给`a`变量。将字符串"([\\\\s\\\\s]+?)"与`delimiters.closer`值中的每一个字符替换为"\\\\原字符"的结果进行连接，将结果赋值给`b`变量。

```javascript
  // generate regexp patterns dynamically.
  var a = delimiters.opener.replace(/(.)/g, '\\$1');
  var b = '([\\s\\s]+?)' + delimiters.closer.replace(/(.)/g, '\\$1');
```
__用例子看看以上这段代码的功效：__

```javascript
> var delimiters = {opener: "<%", closer: "%>"};
> var a = delimiters.opener.replace(/(.)/g, '\\$1');
> a
'\\<\\%'
> var b = '([\\s\\s]+?)' + delimiters.closer.replace(/(.)/g, '\\$1');
> b
'([\\s\\s]+?)\\%\\>'
```
接下来，为`delimiters`变量增加`lodash`属性，属性值中包含由变量`a`和`b`组合成的三组正则表达式，分别对应`evaluate`,`interpolate`和`escape`。

```javascript
  // used by lo-dash.
  delimiters.lodash = {
    evaluate: new regexp(a + b, 'g'),
    interpolate: new regexp(a + '=' + b, 'g'),
    escape: new regexp(a + '-' + b, 'g')
  };
};
```
__调用`template.adddelimiters`来设置默认的名为"config"的配置，这里的模板分隔符使用了[_.template][]中用到的"<%"和"%>"__

```javascript
// the underscore default template syntax should be a pretty sane default for
// the config system.
template.adddelimiters('config', '<%', '%>');
```
关于添加完毕后的`delimiters`变量，我们可以看一下它现在的状态：

```javascript
> template.adddelimiters('config', '<%', '%>');

> alldelimiters
{ config:
   { opener: '<%',
     closer: '%>',
     lodash:
      { evaluate: /\<\%([\s\s]+?)\%\>/g,
        interpolate: /\<\%=([\s\s]+?)\%\>/g,
        escape: /\<\%-([\s\s]+?)\%\>/g 
      } 
   } 
}

```
### template.setdelimiters

设置并返回预定义的分隔符对象。

这个方法仅接受一个参数`name`用来匹配`alldelimiters`中的预定义配置项。如果当前指定的`name`配置不存在则使用默认的"config"配置。

```javascript
// set lo-dash template delimiters.
template.setdelimiters = function(name) {
  // get the appropriate delimiters.
  var delimiters = alldelimiters[name in alldelimiters ? name : 'config'];

```
获得的配置选项`delimiters.lodash`值将被设置到`grunt.util._.templatesettings`，这个设置[\_.templateSettings][]用来指定模板匹配时用到的各种正则表达式。

配置选项`delimiters`则作为返回值返回。

```javascript
  // tell lo-dash which delimiters to use.
  grunt.util._.templatesettings = delimiters.lodash;
  // return the delimiters.
  return delimiters;
};
```
### template.process

该方法通过调用`grunt.util._.template`将数据用`tmpl`模板进行渲染。

方法接受两个参数，`tmpl`为模板对象，`options`则是一个可能包含着`data`，`delimiter`的配置选项对象。

```javascript
// process template + data with lo-dash.
template.process = function(tmpl, options) {
```
对于`options`的判断和处理不再赘述。

```javascript
  if (!options) { options = {}; }
```
接下来，通过调用`template.setdelimiters`获取预定义的，配置名称`options.delimiters`的分隔符对象。当`options.delimiters`为`undefined`或不存在该名称的配置选项时，将得到名称为`config`的预定义分隔符对象。

```javascript
  // set delimiters, and get a opening match character.
  var delimiters = template.setdelimiters(options.delimiters);
```
然后，通过调用[Object.create][]创建一个指定原型的对象`data`，这里的原型按照`options.data`，'grunt.config.data'和`{}`顺序来判断决定。当`options.data`不为`undefined`时，对象原型为`options.data`；否则，如果`grunt.config.data`不为`undefined`时则对象原型为`grunt.config.data`，如果前两者都是`undefined`的话，则原型只能为`{}`。

```javascript
  // clone data, initializing to config data or empty object if omitted.
  var data = Object.create(options.data || grunt.config.data || {});
```
对于不熟悉`Object.create`的同学来说，其实它的实现很简单（__注意：下面的实现只是涵盖了创建一个指定原型的对象这个最主要的用例，并没有对`Object.create(proto [, propertiesObject ])`中的第二个参数即指定相关属性的情形进行考虑。__），请看下面的代码：

```javascript
if (typeof Object.create != 'function') {
    (function () {
        var F = function () {};
        Object.create = function (o) {
            if (arguments.length > 1) { throw Error('Second argument not supported');}
            if (o === null) { throw Error('Cannot set a null [[Prototype]]');}
            if (typeof o != 'object') { throw TypeError('Argument must be an object');}
            F.prototype = o;
            return new F();
        };
    })();
}
```
如果新创建的`data`变量中并未指定"grunt"属性，则将模块开始处引入的`grunt`模块对象赋值给`data.grunt`。

```javascript
  // expose grunt so that grunt utilities can be accessed, but only if it
  // doesn't conflict with an existing .grunt property.
  if (!('grunt' in data)) { data.grunt = grunt; }
```
紧接着，将当前的`tmpl`模板赋值给`last`变量，用来与每次模板渲染的结果进行比较，来确定当前模板是否全部渲染完毕。

```javascript
  // keep track of last change.
  var last = tmpl;
```
在这里，渲染的核心逻辑是：

1. 查找当前模板字符串`tmpl`中是否有指定的开始分隔符`delimiters.opener`，如果没有的话，说明不需要进行渲染退出`while`子句，否则进行第二步。

2. 通过调用`grunt.util._.template`即[\_.template][]来用`data`渲染`tmpl`模板，并将渲染后的结果重新赋值给`tmpl`。

3. 通过比较渲染前的模板`last`与渲染后的模板`tmpl`来确认是否当前模板已经渲染结束。如果两者相等，说明没有再次渲染的必要，直接跳出`while`字句，否则，需要准备下一轮的判断及处理，即步骤四。

4. 将`tmpl`变量重新赋值给`last`变量。然后返回步骤一，重新进行再一轮的判断及处理。

```javascript
  try {
    // as long as tmpl contains template tags, render it and get the result,
    // otherwise just use the template string.
    while (tmpl.indexof(delimiters.opener) >= 0) {
      tmpl = grunt.util._.template(tmpl, data);
      // abort if template didn't change - nothing left to process!
      if (tmpl === last) { break; }
      last = tmpl;
    }
```
综上述，有两个条件可以证实渲染已结束并跳出`while`子句:

* 其一是：模板中再没有指定的开始分隔符`delimiters.opener`，这点很容易让人理解；

* 其二是：即使模板中存在着指定的开始分隔符`delimiters.opener`，但是又进行了一次渲染后发现跟没有进行渲染前的结果是一样的，那么只能选择结束渲染，因为这个开始分隔符可能在模板中有别的意义，或者缺少结束分隔符导致在渲染的过程中没有进行有效的替换，但总之我们需要停止再一次的渲染，因为如果这样下去会陷入一个无休止的渲染死循环。

当然在渲染的过程中，可能会出现各种异常，这里所有渲染过程中的异常都会被捕获并进行统一处理。

这里对[lo-dash][]（或者[underscore.js][]中的1.3.3）版本中关于模板中出现的`\n`或者`\r`字符抛出的异常进行了特殊处理，并且通过`grunt.log`对该异常及修复方法进行了说明。

```javascript
  } catch (e) {
    // in upgrading to lo-dash (or underscore.js 1.3.3), \n and \r in template
    // tags now causes an exception to be thrown. warn the user why this is
    // happening. https://github.com/documentcloud/underscore/issues/553
    if (string(e) === 'syntaxerror: unexpected token illegal' && /\n|\r/.test(tmpl)) {
      grunt.log.errorlns('a special character was detected in this template. ' +
        'inside template tags, the \\n and \\r special characters must be ' +
        'escaped as \\\\n and \\\\r. (grunt 0.4.0+)');
    }
```
对于所有的异常来说，这里都会对异常信息`e.message`进行再次的简单封装，并且通过`grunt.warn`结合`grunt.fail.code`中指定的错误码将异常信息在console中显示。

```javascript
    // slightly better error message.
    e.message = 'an error occurred while processing a template (' + e.message + ').';
    grunt.warn(e, grunt.fail.code.template_error);
  }
```
这里需要注意的是，所有渲染过程中出现的异常并没有影响到整个`grunt`任务执行，最后一次的`tmpl`变量值将会通过`grunt.util.normalizelf`进行换行符处理，并将处理后的结果作为最终的渲染结果进行返回。

```javascript
  // normalize linefeeds and return.
  return grunt.util.normalizelf(tmpl);
};
```

[tmpl]: https://github.com/borismoore/jquery-tmpl "tmpl"
[jsperf]: http://jsperf.com "jsperf"
[dot]: https://github.com/olado/dot "dot"
[_.template]: http://lodash.com/docs/#template "_.template"
[lo-dash]: http://lodash.com/ "lo-dash"
[underscore.js]: http://underscorejs.org "underscore.js"
[dateformat]: https://github.com/felixge/node-dateformat "dateformat"
[artTemplate]: https://github.com/aui/artTemplate "artTemplate"
[_.templateSettings]: http://lodash.com/docs/#templateSettings "_.templateSettings"
[Object.create]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create "Object.create"

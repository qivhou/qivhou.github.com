---
layout : post
category : translation
title : 使用Grunt,Gulp或者Broccoli进行特定环境下的代码构建
tags : [翻译, Grunt, Gulp, Broccoli, 代码构建, 特定环境]
---
{% include JB/setup %}

[原文地址](http://addyosmani.com/blog/environment-specific-builds-with-grunt-gulp-or-broccoli/)

![Image credit: http://www.flickr.com/photos/brianneudorff/9109932159/](http://addyosmani.com/blog/wp-content/uploads/2014/02/9109932159_efae795c6f_z.jpg)

通常dev, staging，还有生产环境的项目版本会有很大的不同。其中一个重要的原因是我们需要更改（scripts/styles/templates）资源文件的路径，更改生成的标记文件或者与环境相关的内容，以及更改特定环境目标的信息。幸亏Grunt，Gulp以及Broccolli微系统中有很多的构建相关的任务，我们可以借助它们来完成构建。

今天，我将提到三种解决这个问题的方法————字符串替换（String Replacement），条件注释（Conditional Comments）以及模板变量（Template Variables）。最终你选择的方法将很大程度上依赖于_你最愿意把条件性的逻辑放在哪里_。

## 最简单的选择 - 字符串/正则表达式替换

对于特定环境的构建来说，最简单的选择是使用字符串或者正则表达式来查找，删除/替换在HTML文件中的代码块。你的构建文件中可以存放大量的针对目标的设置，其中的每一块都可以被替换为你想要的内容。

例如，在Grunt中可以非常容易的使用[grunt-string-replace](https://github.com/erickrdch/grunt-string-replace)，将`source.js`替换为`build.js`来构建生产环境，示例如下：

```javascript
module.exports = function(grunt) {
  grunt.initConfig({
    'string-replace': {
      prod: {
        src: './app/**/*.html',
        dest: './dist/',
        options: {
          replacements: [{
            pattern: 'source.js',
            replacement: 'build.js'
          }]
        }
      }
    }
  });
  grunt.loadNpmTasks('grunt-string-replace');
  grunt.registerTask('default', ['string-replace']);
};

```

   （[grunt-replace](https://github.com/outaTiME/grunt-replace)也可以用来完成相同的工作）

通常，文本替换方式都是能解决问题的，它只需要少量的配置，而且被Stephen Sawchuk（Yeoman和Bower项目的贡献者）这样的开发者使用，Stephen通过文本替换方式发现并改善了他的设置中一些过于复杂的部分。你可以在下面这个[dist](https://gist.github.com/stephenplusplus/523314c4bdda1a8ad1c0)中了解到他是怎样在项目中使用字符串替换的。

如果你没有在使用Grunt，[gulp-replace](https://github.com/lazd/gulp-replace)和[broccoli-replace](https://github.com/outaTiME/broccoli-replace)都是不错的
选择。

在仅需要更新或者移除跨文件的字符串集这样的简单场景中，字符串替换方案可以很好的满足你的需要。但是当你需要跨文件的进行大量替换工作，而且需要来回在源文件和构建文件中跳转来确定究竟哪一部分需要被替换的场景下，这种方式会变得很难维护。

在你的源码中定义你的替换代码块可能是一个更好的方案。这个想法将我们引入了下一个方案。

## 条件注释

使用环境条件注释最基本的思路是：你定义必需的逻辑和信息（比如：路径信息）将一个以HTML注释形式的字符串（为特定目标）替换为其他内容，这个注释中会包含构建任务相关的一些配置。按照这个思路，你可以很容易的基于文件来编写可读的解决方案。这里有一些选项：

Grunt:

* [grunt-targethtml](https://npmjs.org/package/grunt-targethtml/)

* [grunt-preprocess](https://npmjs.org/package/grunt-preprocess/)

* [grunt-processhtml](https://npmjs.org/package/grunt-processhtml/)

Gulp:

* [gulp-preprocess](https://npmjs.org/package/gulp-preprocess/)

* [gulp-processhmlt](https://npmjs.org/package/gulp-processhtml/)

前三个选项中的注释样式在可读性和详细程度上各不相同，但是它们有效的达到了同样的结果。所有选项都可以很容易的输出目标/特定环境代码块，但是让我们先比较下`targethtml`和`preprocess`: 

一个`grunt-targethtml`例子如下 （Dev环境 vs. Production环境）

Dev环境:

```html
<!--(if target dev)><!-->
 <link rel="stylesheet" href="dev.css" />
<!--<!(endif)-->
<!--(if target dev)><!-->
  <script src="dev.js"></script>
  <script>
    var less = { env:'development' };
  </script>
<!--<!(endif)-->
```

Production环境:

```html
<!--(if target prod)><!-->
  <link rel="stylesheet" href="release.css">
<!--<!(endif)-->
<!--(if target prod)><!-->
   <script src="release.js"></script>
<!--<!(endif)-->
```

同样的也来看一下`grunt-preprocess`

Dev环境:

```html
<!-- @if NODE_ENV='production' -->
 <link rel="stylesheet" href="dev.css">
<!-- @endif -->
<!-- @if NODE_ENV='production' -->
<script src="dev.js"></script>
<script>
  var less = { env:'development' };
</script>
<!-- @endif -->
```

Production环境:

```html
<!-- @if NODE_ENV='production' -->
<link rel="stylesheet" href="release.css">
<!-- @endif -->
<!-- @if NODE_ENV='production' -->
<script src="release.js"></script>
<!-- @endif -->
```

正如我们看到的，用注释方式指定我们的逻辑是相当直接的，我一直在用这种方式而且幸运的是至今没有收到太多的抱怨（基于样式上的考虑，我个人倾向于使用`grunt-targethtml`）。

然而对条件注释方式来说，最大的一个弊端是在你的标记文件中将会有大量的构建相关的逻辑。一些开发者可能不喜欢这样，但是就我个人而言，还是挺喜欢这些可读性强的代码的。这也意味着我不需要再返回到Gruntfile中去配置将要如何构建。

之前我一共提到了3个Grunt任务，那么`grunt-processhtml`又是怎样的情况的？嗯，它表现的更专业一些，在构建过程中你可以使用模板变量和其他一些很棒的技巧来包含完全不同的内嵌文件。

例如，这里你可以只在'dist'目标运行的时候才将这个类改为'production'：

```html
<!-- build:[class]:dist production -->
<html class="debug_mode">
<!-- /build -->
```
或者将一个代码块为指定的目标替换为一个特定的文件：

```html
<!-- build:include:dev dev/content.html -->
This will be replaced by the content of dev/content.html
<!-- /build -->
```

再或者根据你的目标移除掉一个代码块：

```html
<!-- build:remove -->
<p>This will be removed when any processhtml target is done</p>
<!-- /build -->
<!-- build:remove:dist -->
<p>But this one only when doing processhtml:dist target</p>
<!-- /build -->
```

似乎有很多可行的任务方案来解决相似的问题，我在这节里将再多讲一个。如果你需要在当前条件注释块中配置注释块格式的能力，去看看[grunt-devcode](https://npmjs.org/package/grunt-devcode/)。一些简单的配置样例如下：

Gruntfile:

```javascript
devcode :
    {
      options :
      {
        html: true,        // html files parsing?
        js: true,          // javascript files parsing?
        css: true,         // css files parsing?
        clean: true,
        block: {
          open: 'condition', // open code block
          close: 'conditionend' // close code block
        },
        dest: 'dist'
      },
```

页面标记：

```html
<!-- condition: !production -->
  <li class="right">
    <a href="#settings" data-toggle="tab">
      Settings
    </a>
  </li>
<!-- conditionend -->
```

很明显，这种方式将解决你的问题。但是如果你想要受益于在条件注释中已经提供的占位符，而且需要把你的构建文件作为用于替换的字符串数据，又该怎么办呢？让我们继续向下看...


## 在HTML中模板变量

编译时模板可以让你使用在构建文件中（比如：Gruntfile）特定目标的数据（字符串，对象，或者其他你需要的）来替换源码文件中定义的占位符字符串（例如：<%- title%>）。这种方式与许多带有条件判断的HTML注释方式不同，在那些方式中你注入的数据并不是在目标文件本身指定的。对于这种方式，你有如下选择：

Grunt：

* [grunt-template](https://npmjs.org/package/grunt-template/)

* [grunt-include-replace](https://npmjs.org/package/grunt-include-replace/)


Gulp：

* [gulp-template](https://npmjs.org/package/gulp-template/)

`grunt-template`（我推荐）基本上是一个`grunt.template.process()`的简单封装。它主要针对基于目标的模板字符串场景，你可以通过`data`对象来改变基于dev/prod目标的路径或者内容。一个用于演示的例子如下示：

index.html.tpl

```html
<title><%- title %></title>
```

Gruntfile.js

```javascript
module.exports = function(grunt) {
    grunt.initConfig({
        'template': {
            'dev': {
                'options': {
                    'data': {
                        'title': 'I <3 JS in dev'
                    }
                },
                'files': {
                    'dev/index.html': ['src/index.html.tpl']
                }
            },
            'prod': {
                'options': {
                    'data': {
                        'title': 'I <3 JS in prod'
                    }
                },
                'files': {
                    'dist/index.html': ['src/index.html.tpl']
                }
            }
        }
    });
    grunt.loadNpmTasks('grunt-template');
    grunt.registerTask('default', [
        'template:prod'
    ]);
    grunt.registerTask('dev', [
        'template:dev'
    ]);
};
```
`grunt-include-replace` 的做法与另一种不同格式的模板字符串（<title>@@title</title>）类似。

## 结束语

我还没有遇到过为了构建特定环境而绞尽脑汁考虑一个理想解决方案的场景，但是我和一些共同工作的开发者在之前的项目中发现了上述这些有用的方案。如果你有一些解决这个问题的更有效或者可维护的方法，请随时分享它们。我希望这篇文章中的内容至少对有些人是有用的。

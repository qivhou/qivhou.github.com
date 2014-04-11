---
layout : post
category : translation
title : 理解MVVM - JavaScript开发者指南
tags : [翻译, MVVM, MVC, MVP, KnockoutJS, Backbone]
---
{% include JB/setup %}

[原文地址](http://addyosmani.com/blog/understanding-mvvm-a-guide-for-javascript-developers/)

![Model View ViewModel](http://addyosmani.com/blog/wp-content/uploads/2012/04/modelview.jpg)

MVVM(模型 视图 视图模型)是一种基于MVC和MVP的架构模式，这种模式试图更清晰的将UI开发从应用的业务逻辑和行为中分离开来。基于这个目的，这种模式的很多实现都利用了声明方式数据绑定（declarative data bindings）从而使得视图（Views）从其他层的分离成为可能。

这一点有利于UI和其他开发工作在同一个代码库中同时展开。UI开发者在文档标记（HTML）中编写到视图模型（ViewModel）的绑定，而模型（Model）和视图模型（ViewModel）由负责应用逻辑的开发者维护。

## MVVM的由来

MVVM最初由微软为了使用Windows Presentation Foundation（[WPF](http://en.wikipedia.org/wiki/Windows_Presentation_Foundation)）和[Silverlight](http://www.microsoft.com/silverlight/)而定义，在2005年由[John Grossman](http://blogs.msdn.com/b/johngossman/)发表的一篇关于Acalon（WPF的代号）的文章中进行了官方声明。并且把它作为MVC的一种替代方案在Adobe Flex社区也很普遍。

现在让我们看看组成MVVM的三个组件。

## Model

像其他MV\*大家庭的成员一样，MVVM中的模型（Model）代表着我们的应用将与之合作的特定域的数据或者信息。用户账户是一个典型的特定域数据的例子（例如：姓名，avatar，email），音乐曲目也是（例如： 标题，年代，专辑）。

模型（Models）掌控信息，但是一般并不控制行为，它们不会格式化信息或者在浏览器中影响数据的呈现，这不是它们的职责范围。相反，数据的格式化由视图（View）控制，而被认为是业务逻辑的行为应该被封装在与模型（Model）互动的另一层--视图模型（ViewModel）中。

这个规则唯一的例外是数据校验，对于模型（Model）来说校验被用来定义或者更新已有模型的数据是可以被接受的。（例如：一个输入的email地址是否匹配一个特定的正则表达式？）

在KnockoutJS中，模型（Models）符合上述的定义，但是模型往往发起Ajax请求服务器端服务来读取和写入模型数据。

如果我们构建一个简单的Todo应用，一个基于KnockoutJS模型的单个Todo项目可能会如下所示：

```javascript

var Todo = function (content, done) {
    this.content = ko.observable(content);
    this.done = ko.observable(done);
    this.editing = ko.observable(false);
};

```

注意：你可能已经注意到了，在上面的代码片段中我们在KnockoutJS的命名空间ko下调用了一个`observables()`方法。在KnockoutJS中，`observables`是特定的JavaScript对象，它能自动检测依赖关系并且通知相关订阅者变化信息。这一点使得我们可以在模型（Model）的属性值发生改变的时候同步模型（Model）和视图模型（ViewModel）。

## View

和在MVC中一样，这里的视图（View）实际上是应用中唯一与用户互动的部分，它是一个展现视图模型（ViewModel）状态的互动用户界面。在这个意义上，MVVM中的视图（View）被认为是主动的而非被动的，但是这句话该怎么理解呢？

一个被动的视图（View）由控制器（controller）进行操控，并不真正的了解我们应用中的模型。MVVM中的主动的视图（View）包含数据绑定，事件以及行为，这些行为要求必须对模型（Model）和视图模型（ViewModel）有所了解。

有一点我们需要牢记：视图（View）不负责处理状态，它负责把状态同步给视图模型（ViewModel）。

KnockoutJS中的视图（View）只是一个带有声明方式绑定的HTML文档，这些声明方式的绑定将它链接到对应的视图模型（ViewModel）。KnockoutJS视图（View）展现视图模型（ViewModel）中的信息，传递指令到视图模型（ViewModel）（例如：用户点击一个元素）并且随着视图模型（ViewModel）的变动进行更新。由视图模型(ViewModel)中数据生成标记的模板（Templates）也被用于此目的。

看看下面这个简单的初始化例子，我们可以了解到在JavaScript的MVVM框架KnockoutJS中是如何定义视图模型（ViewModel）以及在标记中进行相关绑定的。

视图模型（ViewModel）：

```javascript
var aViewModel = {
    contactName: ko.observable('John');
};

```

视图（View）：

```html
<input id="source" data-bind="value: contactName, valueUpdate: 'keyup'" /></p>
<div data-bind="visible: contactName().length > 10">
    You have a really long name!
</div>

```

我们的输入文本框（source）从`contactName`获得初始值，只要contactName变化时便会自动更新这个值。因为数据绑定是双向的，在文本框中的输入也将相应更新`contactName`，所以这个值是一直同步的。

在这个KnockoutJS的实现中，包含了`You have a really long name! `文本的`<div>`标签有一个简单的验证（再一次以数据绑定的形式）。如果输入超过了10个字符，它将会显示，否则将一直隐藏。

我们回到Todo应用看一个更高阶的例子。一段包含了所有必需数据绑定的KnockoutJS的视图代码如下所示：

```html
<div id="todoapp">
    <header>
        <h1>Todos</h1>
        <input id="new-todo" type="text" data-bind="value: current, valueUpdate: 'afterkeydown', enterKey: add"
               placeholder="What needs to be done?"/>
    </header>
    <section id="main" data-bind="block: todos().length">
        <input id="toggle-all" type="checkbox" data-bind="checked: allCompleted">
        <label for="toggle-all">Mark all as complete</label>
        <ul id="todo-list" data-bind="foreach: todos">
           <!-- item -->
            <li data-bind="css: { done: done, editing: editing }">
                <div class="view" data-bind="event: { dblclick: $root.editItem }">
                    <input class="toggle" type="checkbox" data-bind="checked: done">
                    <label data-bind="text: content"></label>
                    <a class="destroy" href="#" data-bind="click: $root.remove"></a>
                </div>
                <input class="edit" type="text"
                       data-bind="value: content, valueUpdate: 'afterkeydown', enterKey: $root.stopEditing, selectAndFocus: editing, event: { blur: $root.stopEditing }"/>
            </li>
        </ul>
    </section>
</div>

```
请注意，这个html基础布局是比较直接的，包含了一个添加新项目的输入框（new-todo），设置项目完成的切换开关，以及一个带着`li`格式的Todo项目模板的列表（todo-list）。

上述在标记中的数据绑定分析如下：

* 输入框`new-todo`对`current`属性有数据绑定，`current`属性存储当前被加入的项目值。我们的视图模型（ViewModel）（稍后展示）观察`current`属性并且对`add`事件也有绑定。当enter键按下后，`add`事件被触发，然后我们的视图模型（ViewModel）准备`current`的值并把它加入Todo列表中。

* 复选框`toggle-all`在点击后可以标记所有当前项目为已完成。如果选中，它将会触发`allCompleted`事件，我们可在视图模型（ViewModel）看到这部分。

* `li`项目有一个名为`done`的类。当一个任务标记为完成后，CSS类`editing`将被相应标记。如果双击这个项目，`$root.editItem`回调函数将被执行。

* 带有CSS类`toggle`的复选框展现`done`属性的状态。

* 标签包含Todo项目的文本值（`content`）。

* 还有一个删除按钮，当你点击后将调用`$root.remove`回调函数。

* 编辑模式下的输入框将保留Todo项目值`content`，`enterKey`事件将设置`editing`属性为true或者false。

## ViewModel

视图模型（ViewModel）可以被看做一个专门做数据变换的控制器（Controller）。它把模型信息转换为视图信息，从视图（View）传递指令到模型（Model）。

例如，想象下我们有一个unix格式的日期属性（例如：1333832407），而不是一般意义上用户认为的日期（例如：04/07/2012 @ 5:00pm）。这里很有必要在显示时将它转换为显示格式，我们的模型（Model）只是简单的持有原始格式的数据。我们的视图（View）包含格式化后的日期，我们的视图模型（ViewModel）作为两者之间的中间人。

在这个意义上，视图模型（ViewModel）可能看起来更像是一个模型（Model）而不是一个视图（View），但是它确实处理了大多数视图（View）显示的逻辑。视图模型（ViewModel）还可以暴露帮助保持视图（View）状态的方法，基于在视图（View）上的动作更新模型（Model）并且触发视图（View）上的事件。

简而言之，视图模型（ViewModel）位于我们的UI层之下。它（从模型（Model）中）提供需要的数据给视图（View）并且可以被看做是我们视图（View）找寻数据和行为的源头。

KnockoutJS将视图模型（ViewModel）阐释为可以在UI上进行的数据和操作的代表。不是UI本身也不是持续存在的数据模型，而是一个可以容纳用户交互中尚未保存的数据的层。Knockout的视图模型（ViewModel）由JavaScript对象实现，没有涉及HTML标记。这种实现的抽象方式允许它们保持简单，意味着更多复杂的行为可以被更容易的在上层按需管理。

部分Todo应用的KnockoutJS视图模型（ViewModel）如下所示：

```javascript
// our main ViewModel
    var ViewModel = function (todos) {
        var self = this;
    // map array of passed in todos to an observableArray of Todo objects
    self.todos = ko.observableArray(ko.utils.arrayMap(todos, function (todo) {
        return new Todo(todo.content, todo.done);
    }));
    // store the new todo value being entered
    self.current = ko.observable();
    // add a new todo, when enter key is pressed
    self.add = function (data, event) {
        var newTodo, current = self.current().trim();
        if (current) {
            newTodo = new Todo(current);
            self.todos.push(newTodo);
            self.current("");
        }
    };
    // remove a single todo
    self.remove = function (todo) {
        self.todos.remove(todo);
    };
    // remove all completed todos
    self.removeCompleted = function () {
        self.todos.remove(function (todo) {
            return todo.done();
        });
    };
    // writeable computed observable to handle marking all complete/incomplete
    self.allCompleted = ko.computed({
        //always return true/false based on the done flag of all todos
        read:function () {
            return !self.remainingCount();
        },
        //set all todos to the written value (true/false)
        write:function (newValue) {
            ko.utils.arrayForEach(self.todos(), function (todo) {
                //set even if value is the same, as subscribers are not notified in that case
                todo.done(newValue);
            });
        }
    });
    // edit an item
    self.editItem = function(item) {
        item.editing(true);
    };
 ..

```
以上我们只是简单提供了一些添加，编辑和删除项目的方法，以及标记所有遗留项目为已完成的逻辑。注意：与之前的例子相比，唯一真正的区别是我们的视图模型（ViewModel）是一个可观察的数组（observable arrays）。在KnockoutJS中，如果我们想要在单个对象上检测并响应对象变化，我们会使用`observables`。然而，如果我们想要在事物集合上检测并响应对象变化，我们应该使用`observableArray`。下面这个更简单一点的例子，将展示如何使用可观察数组：

```javascript
// Define an initially an empty array
var myObservableArray = ko.observableArray();
// Add a value to the array and notify our observers
myObservableArray.push(‘A new todo item’);

```
注意：如果你感兴趣的话，你可以从[TodoMVC](http://todomvc.com/)得到上述基于Knockout.js的Todo应用的完整代码。

### 概括：视图和视图模型

视图（View）和视图模型（ViewModel）通过数据绑定和事件机制进行沟通。正如我们在最初的那个视图模型（ViewModel）例子中看到的，视图模型（ViewModel）不只暴露了模型（Model）属性而且也获得了模型的其他方法以及诸如校验这样的特性。

我们的视图（Views）处理它们自己的用户界面事件，并在必要情况下将其映射到视图模型（ViewModel）中。模型（Models）和视图模型（ViewModel）中的属性通过双向数据绑定进行同步和更新。

### 概括：视图模型和模型

虽然在MVVM模式中视图模型（ViewModel）经常会全面负责模型（Model）事务，然而这个关系中的一些细微之处也很值得注意。视图模型（ViewModel）不但可以为了数据绑定的目的来暴露模型（Model）或者模型（Model）的属性，而且也包含获取和维护那些在视图（View）中暴露出的属性的接口。

## 优点和缺点

现在，你肯定希望对MVVM是什么以及它是如何工作的有一个更深入的了解。让我们回顾下这个模式的优点和缺点先。

### 优点

* MVVM有助于使UI开发和功能开发更容易并行进行。

* 抽取视图（View），从而减少在视图层之后的层中所要求的业务逻辑（或者粘合工作）的代码量。

* 视图模型（ViewModel）相比事件驱动代码，可以更容易的进行单元测试。

* 视图模型（ViewModel）更像一个模型（Model）而不是一个视图（View），它可以在没有UI自动化和交互的情况下测试。

### 缺点

* 对于简单的用户界面来说，MVVM有杀鸡用牛刀的嫌疑。

* 虽然数据绑定可以是声明方式的而且用起来不错，但是它比我们只需要设置断点就可以调试的指令代码更难调试。

* 数据绑定在一个复杂的应用中会创建很多标记。你也不希望在开发结束时发现绑定本身比要绑定的对象还要复杂。

* 在更大一些的应用中，很难设计一个好的可以得到一般化必要量的视图模型（ViewModel）。

##松散数据绑定的MVVM

看到MVC或者MVP背景的JavaScript开发者审阅MVVM并且抱怨MVVM关注点分离的事实并不是一件稀奇的事情。也就是说，内联数据绑定的数量由视图的HTML标记所维护。

我必须承认，当我第一次看到MVVM的实现时（例如：KnockoutJS，Knockback）,我感到惊讶的是居然有开发者希望回到那些把JavaScript逻辑与HTML标记混杂一气，而且已经被证明会很难维护的老路上。然而，现在的情况是MVVM对为什么这样做有很好的理由，（我们在之前已经聊到了），包括使得设计人员能够更容易的从他们的标记绑定到应用逻辑。

对于开发者中的纯粹主义者，如果知道在KnockoutJS 1.3版本中出现的自定义绑定提供者（custom binding providers）特性将大幅减少我们对于数据绑定的依赖，而且这个特性将会在之后所有版本中得到支持，你们肯定会非常高兴的。

默认情况下，KnockoutJS有一个数据绑定提供者（data-binding provider），它将搜索所有像下面这个例子中一样拥有`data-bind`属性的元素。

```html

<input id="new-todo" type="text" data-bind="value: current, valueUpdate: 'afterkeydown', enterKey: add" placeholder="What needs to be done?"/>

```

当这个提供者定位了拥有这个属性的元素后，将对元素进行语义分析并且使用当前上下文将它转变为一个绑定对象。这是KnockoutJS一直遵循的工作方式，允许你以声明方式添加绑定到那些KnockoutJS绑定数据的元素。

一旦你开始构建复杂的视图（View），你可能最终会被大量的元素及属性在标记中的绑定给折腾疯了。然而有了自定义绑定提供者（custom binding providers），这已经不再是一个问题了。

绑定提供者主要对两件事情感兴趣：

* 当被扔给一个Dom节点时，这个节点有任何数据绑定吗？

* 如果这个节点通过了第一个问题，这个绑定对象在当前数据上下文中是什么样子的？

绑定提供者实现了两个函数方法：

* `nodeHasBindings` : 这个函数接受一个DOM节点参数，这个节点不一定是一个元素对象。

* `getBindings` : 返回应用到当前上下文的绑定对象。

一个绑定提供者的骨架看起来像下面这样：

```javascript

var ourBindingProvider = {
    nodeHasBindings: function(node) {
        // returns true/false
    },
    getBindings: function(node, bindingContext) {
        // returns a binding object
    }
};

```

在我们完善这个提供者之前，让我们先简要的讨论一下数据绑定属性中的逻辑。

如果在使用Knockout的MVVM时，你对视图（View）被应用逻辑过度束缚的情况不满意，你可以改变这一点。我们可以实现一些类似于CSS类的东西通过名称来指定绑定。Knockmeout.net的Ryan Niemeyer之前建议使用`data-class`来避免数据类（data classes）与展示类（presentation classes）发生冲突，按照这个方案，我们的`nodeHasBings`函数应该看起来像下面这样：

```javascript

// does an element have any bindings?
function nodeHasBindings(node) {
    return node.getAttribute ? node.getAttribute("data-class") : false;
};

```

接下来，我们需要一个有意义的`getBindings()`函数。既然我们坚持使用CSS类（CSS classes）的方案，那么为什么不想办法支持空格分隔类（space-separated classes）在不同的元素间共享绑定规范呢？

让我们先展望下我们的绑定会是什么样子的。我们创建一个对象去持有绑定，这个对象的属性名需要匹配我们在数据类中希望使用的键值。

注意：将一个KnockoutJS应用从传统数据绑定转换为一个使用自定义绑定提供者提供的无碍绑定（unobstrusive bindings），并不是一个很复杂的工作。我们只是简单的抽取所有数据绑定属性，使用数据类属性对它们进行替换，然后把我们的绑定像下面这样指定给一个对象：

```javascript

var viewModel = new ViewModel(todos || []);
var bindings = {
        newTodo:  {
            value: viewModel.current,
            valueUpdate: 'afterkeydown',
            enterKey: viewModel.add
        },
        taskTooltip :  { visible: viewModel.showTooltip },
        checkAllContainer :  {visible: viewModel.todos().length },
        checkAll: {checked: viewModel.allCompleted },
        todos: {foreach: viewModel.todos },
        todoListItem: function() { return { css: { editing: this.editing } }; },
        todoListItemWrapper: function() { return { css: { done: this.done } }; },
        todoCheckBox: function() {return { checked: this.done }; },
        todoContent: function() { return { text: this.content, event: { dblclick: this.edit } };},
        todoDestroy: function() {return { click: viewModel.remove };},
        todoEdit: function() { return {
            value: this.content,
            valueUpdate: 'afterkeydown',
            enterKey: this.stopEditing,
            event: { blur: this.stopEditing } }; },
        todoCount: {visible: viewModel.remainingCount},
        remainingCount: { text: viewModel.remainingCount },
        remainingCountWord: function() { return { text: viewModel.getLabel(viewModel.remainingCount) };},
        todoClear: {visible: viewModel.completedCount},
        todoClearAll: {click: viewModel.removeCompleted},
        completedCount: { text: viewModel.completedCount },
        completedCountWord: function() { return { text: viewModel.getLabel(viewModel.completedCount) }; },
        todoInstructions: {visible: viewModel.todos().length}
    };
    ....

```

上面的代码片段少了2行代码--事实上我们依然需要`getBindings`方法，这个方法将遍历我们数据类属性的每一个键值然后为他们构建结果对象。如果我们检测到绑定对象是一个函数，我们会使用当前上下文`this`和当前数据调用该函数。完整的自定义绑定提供者如下示：

```javascript

// We can now create a bindingProvider that uses
    // something different than data-bind attributes
    ko.customBindingProvider = function(bindingObject) {
        this.bindingObject = bindingObject;
        //determine if an element has any bindings
        this.nodeHasBindings = function(node) {
            return node.getAttribute ? node.getAttribute("data-class") : false;
        };
      };
    // return the bindings given a node and the bindingContext
    this.getBindings = function(node, bindingContext) {
        var result = {};
        var classes = node.getAttribute("data-class");
        if (classes) {
            classes = classes.split(' ');
            //evaluate each class, build a single object to return
            for (var i = 0, j = classes.length; i < j; i++) {
               var bindingAccessor = this.bindingObject[classes[i]];
               if (bindingAccessor) {
                   var binding = typeof bindingAccessor == "function" ? bindingAccessor.call(bindingContext.$data) : bindingAccessor;
                   ko.utils.extend(result, binding);
               }
            }
        }
        return result;
    };
};

```

因此，我们的`bindings`对象的最后几行可以这样写：

```javascript

   // set ko's current bindingProvider equal to our new binding provider
    ko.bindingProvider.instance = new ko.customBindingProvider(bindings);
    // bind a new instance of our ViewModel to the page
    ko.applyBindings(viewModel);
})();

```

我们在这里所做的都是为了更高效的为绑定处理定义构造函数，该构造函数接受一个对象参数（bindings），用它来查找我们的绑定。 之后我们可以用数据类（data-classes）重写应用视图如下：

```html
<div id="create-todo">
                <input id="new-todo" data-class="newTodo" placeholder="What needs to be done?" />
                <span class="ui-tooltip-top" data-class="taskTooltip" style="display: none;">Press Enter to save this task</span>
            </div>
            <div id="todos">
                <div data-class="checkAllContainer" >
                    <input id="check-all" class="check" type="checkbox" data-class="checkAll" />
                    <label for="check-all">Mark all as complete</label>
                </div>
                <ul id="todo-list" data-class="todos" >
                    <li data-class="todoListItem" >
                        <div class="todo" data-class="todoListItemWrapper" >
                            <div class="display">
                                <input class="check" type="checkbox" data-class="todoCheckBox" />
                                <div class="todo-content" data-class="todoContent" style="cursor: pointer;"></div>
                                <span class="todo-destroy" data-class="todoDestroy"></span>
                            </div>
                            <div class="edit">
                                <input class="todo-input" data-class="todoEdit"/>
                            </div>
                        </div>
                    </li>
                </ul>
            </div>


```

Neil Kerkin使用上面这些代码构建了一个完整的TodoMVC样例应用，可以点击[这里](http://jsfiddle.net/nkerkin/hmq7D/light/)进行查看。

尽管上面这一番云里雾里的解释可能会让你误以为这是一项工作量很大的工作，但是既然你已经有了一个写好的通用的`getBindings`方法，接下来你只需要在你的KnockoutJS应用中简单的重用它，使用数据类而不是严格的数据绑定。这样做的最终的目的是希望你将视图中的数据绑定转移到一个绑定的对象，从而拥有一个更清晰的HTML标记文档。

## MVC Vs. MVP Vs. MVVM

MVP和MVVM都是MVC的衍生框架。MVC与它的衍生框架最大的不同在于每一层对于其他层的依赖以及层次间是如何紧密结合的。

在MVC中，视图（View）位于我们架构的顶端，接下来是控制器（Controller）。模型（Model）位于控制器之下，所以视图和控制器，控制器和模型相互关联。这样的话，我们的视图可以直接访问模型。然而依据我们应用的不同复杂度，把完整的模型暴露给视图会有种种安全和性能的损耗。MVVM试图去避免这些问题。

在MVP中，控制器（Controller）的角色被呈现器（Presenter）所取代，呈现器（Presenter）和视图（View）位于同一级，监听视图（View）和模型（Model）的事件并且调解它们之间的行为。与MVVM不同的是，它没有一个把视图绑定到视图模型的机制，所以我们依赖于每个视图实现的接口，通过这个接口呈现器（Presenter）与视图进行互动。

MVVM允许我们创建一个包含状态和逻辑信息的类视图的模型子集（View-specific subsetf of a Model），避免了把整个模型暴露给视图。与MVP中的呈现器（Presenter）不同，视图模型并不要求对视图进行引用。视图可以绑定到视图模型的属性上，通过这个属性将模型中的数据暴露给视图。正如我们之前提到的，这个视图的抽象意味着代码中更少的逻辑依赖。

然而这样做的一个缺点是，在视图（View）和视图模型（ViewModel）间需要一定程度的转化，这将引发性能上的开销。这种转化的复杂性可能有所不同，有可能是简单的数据拷贝，也可能是比较复杂的数据操作，例如我们想要它成为我们希望看到的视图（View）形式。MVC没有这个问题，因为整个模型是现成的，这类转化操作可以被避免。

## Backbone.js Vs. KnockoutJS

了解MVC，MVP以及MVVM之间的细微差别是很重要的，但是最终开发者将会问到：基于已经学到的这些内容，他们应该使用KnockoutJS还是Backbone？以下这些信息也许有助于找到答案：

* 这两个库是为不同的目的设计的，通常不能简单的说选择MVC还是MVVM

* 如果你主要关心的数据绑定和双向沟通，那么KnockoutJS无疑是你的不二选择。几乎存储在DOM节点中的所有属性和值可以通过这个方式映射到JavaScript对象。

* Backbone擅长与RESTful服务进行整合，而KnockoutJS模型是简单的JavaScript对象,
开发者只能自己编码实现模型更新。

* KnockoutJS专注于自动化UI绑定，但是如果想让Backbone也做到这一点的话，需要更多更细致的自定义代码。对Backbone本身而言，这不是一个问题，因为它故意没有涉及这一UI邻域，然而Knockback却试图解决这个问题。

* 有了KnockoutJS，我们可以绑定我们自定义的函数到视图模型（ViewModel）的`observables`JavaScript对象，任何时间只要`observable`发生改变这个函数就会执行。这一点使得我们可以拥有和Backbone相同水平的灵活性。

* Backbone内置了一个强大的路由解决方案，但KnockoutJS并不提供路由方案。然而使用Ben Alman的[BBQ插件](http://benalman.com/projects/jquery-bbq-plugin/)或者像Miller Medeiros的[Crossroads](http://millermedeiros.github.com/crossroads.js/)这样的独立路由系统，这个问题很容易被解决。

总之，我个人觉得KnockoutJS更适合小一些的应用而Backbone的特性使得它在构建所有复杂应用的时候都能有不错的表现。尽管如此，许多开发者已经使用这两个框架去开发了不同复杂度的一些应用，我建议在决定你的项目要使用的框架之前，可以先小规模的试用下这两个框架。


## 参考文献/进阶阅读

* [The Advantages Of MVVM](http://www.silverlightshow.net/news/The-Advantages-of-MVVM.aspx)

* [SO:What are the problems with MVVM?](http://stackoverflow.com/questions/883895/what-are-the-problems-of-the-mvvm-pattern)

* [MVVM Explained](http://www.codeproject.com/Articles/100175/Model-View-ViewModel-MVVM-Explained)

* [How does MVVM compare to MVC?](http://www.quora.com/Pros-and-cons-of-MVVM-framework-and-how-I-can-campare-it-with-MVC)

* [Custom bindings in KnockoutJS](http://www.knockmeout.net/2011/09/ko-13-preview-part-2-custom-binding.html)

* [Exploring Knockout with TodoMVC](http://gratdevel.blogspot.co.uk/2012/02/exploring-todomvc-and-knockoutjs-with.html)








































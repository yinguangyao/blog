---
title: underscore源码剖析之整体架构
date: 2018-03-16 21:08:56
tags:
- underscore
- 前端
- 编程
categories: [前端, underscore]
---
最近打算好好看看underscore源码，一个是因为自己确实荒废了基础，另一个是underscore源码比较简单，比较易读。
本系列打算对underscore1.8.3中关键函数源码进行分析，希望做到最详细的源码分析。
今天是underscore源码剖析系列第一篇，主要对underscore整体架构和基础函数进行分析。
## 基础模块 ##
首先，我们先来简单的看一下整体的代码：
```
// 这里是一个立即调用函数，使用call绑定了外层的this(全局对象)
(function() {

  var root = this;
  
  // 保存当前环境中已经存在的_变量（在noConflict中用到）
  var previousUnderscore = root._;
  
  // 用变量保存原生方法的引用，以防止这些方法被重写，也便于压缩
  var ArrayProto = Array.prototype, ObjProto = Object.prototype, FuncProto = Function.prototype;

  var
    push             = ArrayProto.push,
    slice            = ArrayProto.slice,
    toString         = ObjProto.toString,
    hasOwnProperty   = ObjProto.hasOwnProperty;

  var
    nativeIsArray      = Array.isArray,
    nativeKeys         = Object.keys,
    nativeBind         = FuncProto.bind,
    nativeCreate       = Object.create;

  var Ctor = function(){};
  // 内部实现省略
  var _ = function(obj) {};
  
    // 这里是各种方法的实现（省略）
    
  // 导出underscore方法，如果有exports则用exports导出，如果    没有，则将其设为全局变量
  if (typeof exports !== 'undefined') {
    if (typeof module !== 'undefined' && module.exports) {
      exports = module.exports = _;
    }
    exports._ = _;
  } else {
    root._ = _;
  }
  
  // 版本号
  _.VERSION = '1.8.3';
  
  // 用amd的形式导出
  if (typeof define === 'function' && define.amd) {
    define('underscore', [], function() {
      return _;
    });
  }
}.call(this))
```
<!-- more -->
## 全局对象 ##
这段代码整体比较简单，不过我看后来的underscore版本有一些小改动，主要是将var root = this;替换为下面这句：
```
var root = typeof self == 'object' && self.self === self && self || typeof global == 'object' && global.global === global && global || this;
```
这里增加了对self和global的判断，self属性可以返回对窗口自身的引用，等价于window，这里主要是为了兼容web worker，因为web worker中是没有window的，global则是为了兼容node，而且在严格模式下，立即执行函数内部的this是undefined。

## void(0) ? undefined ##
扫一眼源码，我们会发现在源码中并没有见到undefined的出现，反而是用void(0)或者void 0来代替的，那么这个void到底是什么？为什么不能直接用undefined呢？
关于void的解释，我们可以看这里：[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/void)
void 运算符通常只用于获取 undefined的原始值，一般使用void(0)，因为undefined不是保留字，在低版本浏览器或者局部作用域中是可以被当做变量赋值的，这样就会导致我们拿不到正确的undefined值，在很多压缩工具中都是将undefined用void 0来代替掉了。
其实这里不仅是void 0可以拿到undefined，还有其他很多方法也可以拿到，比如0["ygy"]、Object.\__undefined\__、Object.\__ygy\__，这些原理都是访问一个不存在的属性，所以最后一定会返回undefined

## noConflict ##
也许有时候我们会碰到这样一种情况，\_已经被当做一个变量声明了，我们引入underscore后会覆盖这个变量，但是又不想这个变量被覆盖，还好underscore提供了noConflict这个方法。
```
_.noConflict = function() {
    root._ = previousUnderscore;
    return this;
};
var underscore = _.noConflict();
```
显而易见，这里正常保留原来的\_变量，并返回了underscore这个方法（this就是\_方法）
## \_ ##
接下来讲到了本文的重点，关于\_方法的分析，在看源码之前，我们先熟悉一下\_的用法。
这里总结的是我日常的用法，如果有遗漏，希望大家补充。
一种是直接调用\_上的方法，比如\_.map([1, 2, 3])，另一种是通过实例访问原型上的方法，比如\_([1, 2, 3]).map()，这里和jQuery的用法很像，\$.extend调用jQuery对象上的方法，而\$("body").click()则是调用jQuery原型上的方法。

既然\_可以使用原型上面的方法，那么说明执行_函数的时候肯定会返回一个实例。

这里来看源码：
```
// instanceof 运算符用来测试一个对象在其原型链中是否存在一个构造函数的 prototype 属性。
// 我这里有个不够准确但容易理解的说法，就是检查一个对象是否为另一个构造函数的实例，为了更容易理解，下面将全部以XXX是XXX的实例的方式来说。
var _ = function(obj) {
    // 如果obj是_的实例（这种情况我真的没碰到过）
    if (obj instanceof _) return obj;
    // 如果this不是_构造函数的实例，那就以obj为参数 new一个实例（相等于修改了_函数）
    if (!(this instanceof _)) return new _(obj);
    // 对应_([1,2,3])这种情况
    this._wrapped = obj;
  };
```
我先从源码上来解释，这里可以看出来\_是一个构造函数，我们都知道，我既可以在构造函数上面增加方法，还可以在原型上面增加方法，前者只能通过构造函数本身访问到，后者由于原型链的存在，可以在构造函数的实例上面访问到。
```
var Person = function() {
    this.name = "ygy";
    this.age = 22;
}
Person.say = function() {
    console.log("hello")
}
Person.prototype.say = function() {
    console.log("world")
}
var ygy = new Person();
Person.say(); // hello
ygy.say(); // world
```
所以我们平时用的\_.map就是Person.say()这种用法，而\_([1, 2, 3]).map则是ygy.say()这种用法。

在继续讲这个之前，我们再来复习一下原型的知识，当我们new一个实例的时候到处发生了什么？

首先，这里会先创建一个空对象，这个空对象继承了构造函数的原型（或者理解为空对象上增加一个指向构造函数原型的指针\__proto\__），之后会根据实例传入的参数执行一遍构造函数，将构造函数内部的this绑定到这个新对象中，最后返回这个对象，过程和如下类似：
```
var ygy = {};
ygy.__proto__ = Person.prototype 
// 或者var ygy = Object.create(Person.prototype)
Person.call(ygy);
```
这样就很好理解了，要是想调用原型上面的方法，必须先new一个实例出来。我们再来分析\_方法的源码：
\_接收一个对象作为参数，如果这个对象是\_的一个实例，那么直接返回这个对象。（这种情况我倒是没见过）
    
如果this不是\_的实例，那么就会返回一个新的实例new \_(obj)，这个该怎么理解？
我们需要结合例子来看这句话，在\_([1, 2, 3])中，obj肯定是指[1, 2, 3]这个数组，那么this是指什么呢？我觉得this是指window，不信你直接执行一下上面例子中的Person()？你会发现在全局作用域中是可以拿到name和age两个属性的。
    
那么既然this指向window，那么this肯定不是\_的实例，所以this instanceof \_必然会返回false，这样的话就会return一个new \_([1, 2, 3])，所以\_([1, 2, 3])就是new \_([1, 2, 3])，从我们前面对new的解释来看，这个过程表现如下：
```
var obj = {}
obj.__proto__ = _.prototype
// 此时_函数中this的是new _(obj)，this instanceof _是true，所以就不会重新return一个new _(obj)，这样避免了循环调用
_.call(obj) // 实际上做了这一步： obj._wrapped = [1, 2, 3]
```
这样我们就理解了为什么\_([1, 2, 3]).map中map是原型上的方法，因为\_([1, 2, 3])是一个实例。

我这里再提供一个自己实现的_思路，和jQuery的实现类似，这里就不作解释了：
```
var _ = function(obj) {
    return new _.prototype.init(obj)
}
_.prototype = {
    init: function(obj) {
    	this.__wrapped = obj
    	return this
    },
    name: function(name) {
        console.log(name)
    }
}
_.prototype.init.prototype = _.prototype;
var a = _([1, 2, 3])
a.name("ygy"); // ygy
```
underscore中所有方法都是在\_方法上面直接挂载的，并且用mixin方法将这些方法再一次挂载到了原型上面。不过，由于篇幅有限，mixin方法的实现会在后文中给大家讲解。
如果本文有错误和不足之处，希望大家指出。
<head>
    <script src="//cdn1.lncld.net/static/js/3.0.4/av-min.js"></script>
    <script src='//unpkg.com/valine/dist/Valine.min.js'></script>
</head>
<body>
    <div id="comment"></div>
</body>
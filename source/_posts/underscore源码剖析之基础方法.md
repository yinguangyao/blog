---
title: underscore源码剖析之基础方法
date: 2018-03-17 22:10:16
tags:
- underscore
- 前端
- 编程
categories: [前端, underscore]
---
本文是underscore源码剖析系列的第二篇，主要介绍underscore中一些基础方法的实现。
## mixin ##
在上篇文章[underscore整体架构分析][1]中，我们讲过**\_**上面的方法有两种挂载方式，一个是挂载到**\_**构造函数上以\_.map(arr)的形式直接调用**（在后文上统称构造函数调用）**，另一种则是挂到\_.prototype上以\_(arr).map()的形式被实例调用**（在后文上统称原型调用）**。

翻一遍underscore源码你会发现underscore中的方法都是直接挂到\_构造函数上实现的，但是会通过mixin方法来将\_上面的方法扩展到\_.prototype上面，这样这些方法既可以直接调用，又可以通过实例来调用。
```
_.mixin = function(obj) {
    // 遍历obj上所有的方法
    _.each(_.functions(obj), function(name) {
        // 保存方法的引用
        var func = _[name] = obj[name];
        _.prototype[name] = function() {
            // 将一开始传入的值放到数组中
            var args = [this._wrapped];
            // 将方法的参数一起push到数组中（这里处理的很好，保证了func方法参数的顺序）
            push.apply(args, arguments);
            // 这里先用apply方法执行了func，并将结果传给了result
            return result(this, func.apply(_, args));
        };
    });
};

_.mixin(_);
```
从这段代码中我们可以看出，mixin方法将\_上的所有方法通过遍历的形式挂载到了\_.prototype上面。
<!-- more -->
细心观察一下，构造函数调用和原型调用的区别在哪里？
没错，区别就在于调用方式和传参，构造函数调用时一般会把要处理的值当做第一个参数传入，而原型调用的时候会把要处理的值传入\_构造函数来创建一个实例。
```
var arr = [1, 2, 3]
var func = function(item) {
    console.log(item);
}
// 构造函数调用时arr被传入第一个参数
_.each(arr, func)
// 原型调用的时候，arr被当做参数传给_方法来创建一个实例
_(arr).each(func)
// 链式调用，和上面类似
_.chain(arr).each(func)
```
从上一节中我们知道，在创建一个\_的实例时，会用this.\_wrapped将传入的值保存起来，所以在mixin里面这一句：var args = [this.\_wrapped];是将我们传给\_的值放到args数组第一项中，之后再将arguments也放入args数组中，借助apply方法执行当前遍历的方法（在这个例子中是each），这个时候传给each方法的是arr和func，正好和原来直接\_.each调用each传入参数的顺序是一样的（underscore中的方法第一项基本上都是要处理的数据）。
## 链式调用 ##
那么上面最后return result(this, func.apply(\_, args))，result又是做什么的呢？

首先来看result源码：
```
var result = function(instance, obj) {
    // 首先判断是否使用链式调用，如果是，那就继续将刚刚执行后返回的结果链式调用一下，如果不是，则直接返回执行后的结果
    return instance._chain ? _(obj).chain() : obj;
};
_.chain = function(obj) {
    // 创建一个实例
    var instance = _(obj);
    // 给这个实例加个_chain属性来表明这是链式调用
    instance._chain = true;
    return instance;
};
```
我们知道underscore中也是有和jQuery类似的链式调用，来看一下链式调用的例子：
```
var arr = [1, 2, 3]
var newArr = _.chain(a).map(function(item) {
    return item + 1
}).filter(function(item) {
    return item > 2
}).value()
```
链式调用的关键在于每次执行方法后都需要返回一个实例，以确保能够继续调用其他方法。
chain方法会用传入的obj创建一个\_的实例，这个实例可以调用原型上的方法。从上面mixin的实现来看，每次调用原型方法后会将执行后的结果传给result方法，在result内部会判断你是否使用了链式调用（chain），如果是链式的，那么就会将返回结果链式化（传入chain中创建新的实例）。
链式调用一定要在结尾执行value方法，不然最后返回的是一个对象（最后一次创建的\_实例）
## 数组函数 ##
underscore构造方法上面并没有直接对push、pop、shift等数组方法进行实现，但是链式调用的时候往往需要用到这些方法，所以在原型上对这些方法做了一些封装，实现方法和mixin类似，这里不再多做解释。
```
_.each(['pop', 'push', 'reverse', 'shift', 'sort', 'splice', 'unshift'], function(name) {
    var method = ArrayProto[name];
    _.prototype[name] = function() {
    var obj = this._wrapped;
    method.apply(obj, arguments);
    // 这句是好像是为了解决ie上的bug？
    if ((name === 'shift' || name === 'splice') && obj.length === 0) delete obj[0];
        return result(this, obj);
    };
});

_.each(['concat', 'join', 'slice'], function(name) {
    var method = ArrayProto[name];
    _.prototype[name] = function() {
        return result(this, method.apply(this._wrapped, arguments));
    };
});
```
  [1]: https://segmentfault.com/a/1190000013789060?_ea=3463450
  <head> 
    <script src="//cdn1.lncld.net/static/js/3.0.4/av-min.js"></script>
    <script src='//unpkg.com/valine/dist/Valine.min.js'></script>
</head>
<body>
    <div id="comment"></div>
</body>
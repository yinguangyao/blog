---
title: underscore源码剖析之数组遍历函数分析（一）
date: 2018-03-19 22:11:23
tags:
- underscore
- 前端
- 编程
categories: [前端, underscore]
---
这是underscore源码剖析系列第三篇文章，主要介绍underscore中each、map、filter、every、reduce等我们常用的一些遍历数组的方法。


## **each** ##

在underscore中我们最常用的就是each和map两个方法了，这两个方法一般接收三个参数，分别是数组/对象、函数、上下文。
```
// iteratee函数有三个参数，分别是item、index、array或者value、key、obj
_.each = _.forEach = function(obj, iteratee, context) {
    // 如果不传context，那么each方法里面的this就会指向window
    iteratee = optimizeCb(iteratee, context);
    var i, length;
    // 如果是类数组，一般来说包括数组、arguments、DOM集合等等
    if (isArrayLike(obj)) {
        for (i = 0, length = obj.length; i < length; i++) {
            iteratee(obj[i], i, obj);
        }
    // 一般是指对象
    } else {
        var keys = _.keys(obj);
        for (i = 0, length = keys.length; i < length; i++) {
            iteratee(obj[keys[i]], keys[i], obj);
        }
    }
    return obj;
};
```
each函数的源码很简单，函数内部会使用isArrayLike方法来判断当前传入的第一个参数是类数组或者对象，如果是类数组，直接使用访问下标的方式来遍历，并将数组的项和index传给iteratee函数，如果是对象，则先获取到对象的keys，再进行遍历后将对象的value和key传给iteratee函数
<!-- more -->
不过在这里，我们主要分析optimizeCb和isArrayLike两个函数。
### optimizeCb ###
```
    // 这个函数主要是给传进来的func函数绑定context作用域。
	var optimizeCb = function (func, context, argCount) {
	    // 如果没有传context，那就直接返回func函数
		if (context === void 0) return func;
		// 如果没有传入argCount，那就默认是3。这里是根据第二次传入的参数个数来给call函数传入不同数量的参数
		switch (argCount == null ? 3 : argCount) {
			case 1: return function (value) {
				return func.call(context, value);
			};
			case 2: return function (value, other) {
				return func.call(context, value, other);
			};
			// 一般是each、map等
			case 3: return function (value, index, collection) {
				return func.call(context, value, index, collection);
			};
			// 一般是reduce等
			case 4: return function (accumulator, value, index, collection) {
				return func.call(context, accumulator, value, index, collection);
			};
		}
		// 如果参数数量大于4
		return function () {
			return func.apply(context, arguments);
		};
	};
```
其实我们很容易就看出来optimizeCb函数只是帮func函数绑定context的，如果不存在context，那么直接返回func，否则则会根据第二次传给func函数的参数数量来判断给call函数传几个值。
这里有个重点，为什么要用这么麻烦的方式，而不直接用apply来将arguments全部传进去？
原因是call方法的速度要比apply方法更快，因为apply会对数组参数进行检验和拷贝，所以这里就对常用的几种形式使用了call，其他情况下使用了apply，详情可以看这里：[call和apply][1]

### isArrayLike ###
关于isArrayLike方法，我们来看underscore的实现。（这个延伸比较多，如果没兴趣，可以跳过）
```
// 一个高阶函数，返回对象上某个具体属性的值
var property = function (key) {
	return function (obj) {
		return obj == null ? void 0 : obj[key];
	};
};

// 这里有个ios8上面的bug，会导致类似var pbj = {1: "a", 2: "b", 3: "c"}这种对象的obj.length = 4; jQuery中也有这个bug。
// https://github.com/jashkenas/underscore/issues/2081 
// https://github.com/jquery/jquery/issues/2145
// MAX_SAFE_INTEGER is 9007199254740991 (Math.pow(2, 53) - 1).
// http://ecma-international.org/ecma-262/6.0/#sec-number.max_safe_integer
var MAX_ARRAY_INDEX = Math.pow(2, 53) - 1;

// 据说用obj["length"]就可以解决？我没有ios8的环境，有兴趣的可以试试
var getLength = property('length');

// 判断是否是类数组，如果有length属性并且值为number类型即可视作类数组
var isArrayLike = function (collection) {
	var length = getLength(collection);
	return typeof length == 'number' && length >= 0 && length <= MAX_ARRAY_INDEX;
};
```
在underscore中，只要带有length属性，都可以被认为是类数组，所以即使是{length: 10}这种情况也会被归为类数组。
我个人感觉这样写其实太过片面，我还是更喜欢jQuery里面isArrayLike方法的实现。
```
function isArrayLike(obj) {
	// Support: real iOS 8.2 only (not reproducible in simulator)
	// `in` check used to prevent JIT error (gh-2145)
	// hasOwn isn't used here due to false negatives
	// regarding Nodelist length in IE
	var length = !!obj && "length" in obj && obj.length,
		type = toType(obj);
	// 排除了obj为function和全局中有length变量的情况
	if (isFunction(obj) || isWindow(obj)) {
		return false;
	}
	return type === "array" || length === 0 ||
		typeof length === "number" && length > 0 && (length - 1) in obj;
}
```
jQuery中使用in来解决ios8下面那个JIT的错误，同时还会排除obj是函数和window的情况，因为如果obj是函数，那么obj.length则是这个函数参数的个数，而如果obj是window，那么我在全局中定义一个var length = 10，这个同样也能获取到length。

最后的三个判断分别是：

 1. 如果obj的类型是数组，那么返回true
 2. 如果obj的length是0，也返回true。即使是{length: 0}这种情况，因为在调用isArrayLike的each和map等方法中会在for循环里面判断length，所以也不会造成影响。
 3. 最后这个(length - 1) in obj我个人理解就是为了排除{length: 10}这种情况，因为这个可以满足length>0和length==="number"的情况，但是一般情况下是无法满足最后(length - 1) in obj的，但是NodeList和arguments这些却可以满足这个条件。
## map ##
说完了each，我们再来说说map，map函数其实和each的实现很类似，不过不一样的一个地方在于，map函数的第二个参数不一定是函数，我们可以什么都不传，甚至还可以传个对象。
```
var arr = [{name:'Kevin'}, {name: 'Daisy', age: 18}]
var result1 = _.map(arr); // [{name:'Kevin'}, {name: 'Daisy', age: 18}]
var result2 = _.map(arr, {name: 'Daisy'}) // [false, true]
```
所以这里就会对传入map的第二个参数进行判断，整体来说map函数的实现比each更加简洁。
```
_.map = _.collect = function (obj, iteratee, context) {
		// 因为在map中，第二个参数可能不是函数，所以用cb，这点和each的实现不一样。
		iteratee = cb(iteratee, context);
		// 如果不是类数组（是对象），则获取到keys
		var keys = !isArrayLike(obj) && _.keys(obj),
			length = (keys || obj).length,
			results = Array(length);
		// 这里根据keys是否存在来判断传给iteratee是key还是index
		for (var index = 0; index < length; index++) {
			var currentKey = keys ? keys[index] : index;
			results[index] = iteratee(obj[currentKey], currentKey, obj);
		}
		return results;
	};
```
### cb ###
我们来看看map函数中这个cb函数到底是什么来历？
```
_.identity = function (value) {
	return value;
};
var cb = function (value, context, argCount) {
    // 如果value不存在
	if (value == null) return _.identity;
	// 如果传入的是个函数
	if (_.isFunction(value)) return optimizeCb(value, context, argCount);
	// 如果传入的是个对象
	if (_.isObject(value)) return _.matcher(value);
	return _.property(value);
};
```
cb函数在underscore中一般是用在遍历方法中，大多数情况下value都是一个函数，我们结合上面map的源码和例子来看。
 1. 如果value不存在，那就对应上面的_.map(obj)的情况，map中的iteratee就是_.identity函数，他会将后面接收到的obj[currentKey]直接返回。
 2. 如果value是一个函数，就对应_.map(obj, func)这种情况，那么会再调用optimizeCb方法，这里就和each的实现是一样的
 3. 如果value是个对象，对应_.map(obj, arrts)的情况，就会比较obj中的属性是否在arr里面，这个时候会调用_.matcher函数
 4. 这种情况一般是用在_.iteratee函数中，用来访问对象的某个属性，具体看这里：[iteratee函数][2]
### matcher ###
那么我们再来看matcher函数，matcher函数内部对两个对象做了浅比较。
```
_.matcher = _.matches = function (attrs) {
    // 将attrs和{}合并为一个对象（避免attrs为undefined）
	attrs = _.extendOwn({}, attrs);
	return function (obj) {
		return _.isMatch(obj, attrs);
	};
};
// isMatch方法会对接收到的attrs对象进行遍历，同时比较obj中是否有这一项
_.isMatch = function (object, attrs) {
	var keys = _.keys(attrs), length = keys.length;
	// 如果object和attr都是空，那么返回true，否则object为空时返回false
	if (object == null) return !length;
	// 这一步没懂是为了做什么？
	var obj = Object(object);
	for (var i = 0; i < length; i++) {
		var key = keys[i];
		if (attrs[key] !== obj[key] || !(key in obj)) return false;
	}
	return true;
};
```
matcher是个高阶方法，他会将两次接收到的对象传给isMatch函数来进行判断。首先是以attrs为被遍历的对象，通过对比obj[key]和attrs[key]的值，只要obj中的值和attrs中的不想等，就会返回false。
这里还会排除一种情况，如果attrs中对应key的value正好是undefined，而且obj中并没有key这个属性，这样obj[key]和attrs[key]其实都是undefined，这里使用！==来比较必然会返回false，实际上两者应该是不想等的。
所以使用in来判断obj上到底有没有key这个属性，如果没有，也会返回false。如果attrs上面所有属性在obj中都能找到，并且两者的值正好相等，那么就会返回true。
这也就是为什么_.map([{name:'Kevin'}, {name: 'Daisy', age: 18}], {name: 'Daisy'}); 会返回 [false, true]。
### 重写each ###
each和map实现原理基本上一样，不过map更加简洁，这里可以用map的形式重写一下each
```
_.each = _.forEach = function (obj, iteratee, context) {
		iteratee = optimizeCb(iteratee, context);
		var keys = !isArrayLike(obj) && _.keys(obj),
			length = (keys || obj).length,
			results = Array(length);
		for (var index = 0; index < length; index++) {
			var currentKey = keys ? keys[index] : index;
			iteratee(obj[currentKey], currentKey, obj);
		}
		return obj;
	};
```
## filter、every、some、reject ##
这几种方法的实现和上面的each、map类似，这里就不多做解释了，有兴趣的可以自己去看一下。

  [1]: https://segmentfault.com/q/1010000007894513
  [2]: http://www.bootcss.com/p/underscore/#iteratee
  <head>
    <script src="//cdn1.lncld.net/static/js/3.0.4/av-min.js"></script>
    <script src='//unpkg.com/valine/dist/Valine.min.js'></script>
</head>
<body>
    <div id="comment"></div>
</body>
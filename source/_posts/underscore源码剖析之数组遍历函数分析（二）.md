---
title: underscore源码剖析之数组遍历函数分析（二）
date: 2018-03-20 22:13:12
tags:
- underscore
- 前端
- 编程
categories: [前端, underscore]
---
## 用法 ##

上一篇主要介绍了each和map这些函数的实现，本篇继续分析reduce的源码。
在underscore中有reduce和reduceRight两个方法，reduce是从数组或对象第一项开始遍历，reduceRight则是从最后一项开始遍历。
```
var arr = [1, 2, 3, 4];
_.reduce(arr, function(result, item) {
    result += item;
    console.log(result); // 1, 3, 6, 10
    return result;
}, 0)
_.reduceRight(arr, function(result, item) {
    result += item;
    console.log(result); // 4, 7, 9, 10
    return result;
}, 0)
```
reduce函数有四个参数，分别是list（数组或对象）、iteratee（函数）、memo（初始值）、context（上下文），func函数则会接收四个参数，分别是memo执行后的结果、数组项/对象value、index/key，list，具体用法可以看这里：[reduce的用法][1]
<!-- more -->
## createReduce实现 ##

```
_.reduce = _.foldl = _.inject = createReduce(1);
_.reduceRight = _.foldr = createReduce(-1);
// createReduce会根据dir的值来控制遍历方向
function createReduce(dir) {
	function iterator(obj, iteratee, memo, keys, index, length) {
		// 从第index个值开始迭代，dir为1的时候从左往右跌代，dir为-1的时候从右往左跌代        
		// 将每次返回的值赋给memo，以便于下次继续调用，直到走完这个循环，最后返回memo
		for (; index >= 0 && index < length; index += dir) {
			var currentKey = keys ? keys[index] : index;
			memo = iteratee(memo, obj[currentKey], currentKey, obj);
		}
		return memo;
	}
	// _.reduce传参分别是对象/数组, 函数，初始值，上下文
	// 根据dir的值来判断index的值（从第一个元素还是最后一个元素开始迭代）
	// 如果memo有值，那么就从obj第一个元素迭代
	// 如果没有传入memo和context两个值的时候，需要对memo赋个初始值
	// 也就是说将第一个元素作为初始值给memo，从obj第二个元素开始迭代
	// 或者将最后一个元素作为初始值给memo，从obj倒数第二个元素开始迭代
	return function (obj, iteratee, memo, context) {
	    // 调用optimizeCb，返回一个绑定了context上下文的iteratee
		iteratee = optimizeCb(iteratee, context, 4);
		
		var keys = !isArrayLike(obj) && _.keys(obj),
			length = (keys || obj).length,
			index = dir > 0 ? 0 : length - 1;
		// 如果没有memo和context
		if (arguments.length < 3) {
			// 如果这里没有传入第三个初始值，那么会把obj里面第一个或最后一个值赋过去
			memo = obj[keys ? keys[index] : index];
			index += dir;
		}
		return iterator(obj, iteratee, memo, keys, index, length);
	};
}
```
createReduce函数是一个高阶函数，接收了一个dir，通过dir来判断是从list是从左往右遍历还是从右往左遍历，之所以用1和-1是为了后续遍历的时候，直接让index加上dir来实现循环，如果是从左往右遍历，那么dir为1，在for循环中index每次加1，通过访问index或key[index]就可以实现从左往右遍历。如果是从右往左遍历，那么dir为-1，在for循环中index每次减1，这样就实现了从右往左遍历。

如果没有给reduce传memo这个初始值，则会将list中的第一项或者最后一项赋给memo，之后的迭代从第二项或者倒数第二项开始。

一共就是下面四种情况：

 1. 如果dir为1，也就是调用reduce的时候，如果传入了memo，那么会从list的第一项开始遍历，将memo和循环中当前的项传给iteratee函数执行，并且将执行结束后的结果重新赋值给memo，这样一直迭代下去，得到最后的memo值。
 2. 如果dir为1，并且没有传入memo，那么会将list中的第一项赋给memo，之后从第二项再开始遍历list。
 3. 如果dir为-1，也就是调用reduceRight的时候，如果传入了memo，那么会从list的最后一项开始遍历。
 4. 如果dir为-1，并且没有传入memo，那么会将list中的最后一项赋给memo，之后从倒数第二项再开始遍历list。
## 应用场景 ##
reduce可以有很多很巧妙的用法，某些情况下代替each、map循环也会更加方便，比如计算一个数组中所有项之和的时候，那么reduce还有其他什么用法呢？
### 深层取值 ###
我们在平时写代码的时候经常会遇到给对象内部某个属性赋值，但是这个属性在很深的层级里面，如果直接使用.来赋值，只要前面有一个是undefined，后面必报错。
```
var obj = {
    info1: {
        age: 20
    }
}
var age = obj.info1.age
var name = obj.info2.name // Cannot read property 'name' of undefined
```
我们想获取obj下面info2属性的name，但是因为obj里面没有info2，所以obj.info2是undefined，我们访问的是undefined.name，这个时候肯定会报错的，我们只能用obj.info2 && obj.info2.name的丑陋形式来取值，如果是更深的层级，就需要写更多的&&，所以我们可以使用reduce来封装一个get方法来优雅的深层取值。
```
var get = function(obj, attrArr) {
    return _.reduce(attrArr, function(obj, attr) {
        return obj && obj[attr]
    }, obj)
}

```
我们找个例子来试试看：
```
var obj = {
	country: {
		name: "china",
		city: {
			name: "shanghai",
			street: {
				name: "changning"
			}
		}

	}
}

get(obj, ["name", "name"]) // undefined
get(obj, ["country", "city", "name"]) // "shanghai"
get(obj, ["country", "city", "street", "name"]) // "changning"
```
### 数组扁平化 ###
这是之前在一篇博客里面看到的面试题，怎么将下面数组拍平？
```
    var arr = [1, [2, [3, [4]]]]
    var arr2 = [[1, 2, 3], [4], [5 ,6 , [7, 8]]]
```
我们可以用reduce这样写。
```
    var flatten = function(arr) {  
        return _.reduce(arr, function(originArr, item) {
            return originArr.concat(Array.isArray(item) ? flatten(item) : item)
        }, [])
    }
```
这里主要是对数组里面的项进行判断，如果是数组，那么就进行递归，一直到返回一个非数组的值，这样时候将这些值用一个空数组concat起来，最后就得到了一个新的扁平的数组。

  [1]: http://www.bootcss.com/p/underscore/#reduce
  <head>
    <script src="//cdn1.lncld.net/static/js/3.0.4/av-min.js"></script>
    <script src='//unpkg.com/valine/dist/Valine.min.js'></script>
</head>
<body>
    <div id="comment"></div>
</body>
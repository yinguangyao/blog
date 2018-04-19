---
layout: underscore
title: throttle节流函数分析
date: 2018-03-22 22:14:47
tags:
- underscore
- 前端
- 编程
categories: [前端, underscore]
---
这是underscore源码剖析系列第五篇，今天来聊一下throttle和debounce两个函数。
## throttle节流函数 ##

Javascript中的函数大多数情况下都是用户调用执行的，但是在某些场景下不是用户直接控制的，在这些场景下，函数会被频繁调用，容易造成性能问题。
比如在window.onresize事件和window.onScroll事件中，由于用户可以不断地触发，这会导致函数短时间内频繁调用，如果函数中有复杂的计算，很容易就造成性能的问题。
这些场景下最主要的问题是触发频率太高，1s内可以触发数次，但是大多数情况下我们并不需要那么高的触发频率，可能只要在500ms内触发一次，这样其实我们可以用setTimeout来解决，在这期间的触发都忽略掉。
我们可以先尝试着自己实现一个节流函数：
```
  // 自己实现的简单节流函数
function throttle (func, time) {
	var timeout = null,
		context = null,
		args = null
	return function() {
	    context = this
		args = arguments
		// 只要timeout函数存在，所有调用都无视
		if(timeout) return;
		timeout = setTimeout(function() {
			func.apply(context, args)
			clearTimeout(timeout)
			timeout = null
		}, time||500)
	}
}
```
我们实现了一个简单的节流函数，但是还不够完整，如果我想在第一次触发的时候立即执行怎么办？如果我想禁用掉最后一次执行怎么办？underscore中实现了一个比较完整的节流函数。
<!-- more -->
```
// options是一个对象，如果options.leading为false，就是禁用第一次触发立即调用
// 如果options.trailing为false，则是禁用第一次执行
_.throttle = function (func, wait, options) {
		// 一些初始化操作
		var context, args, result;
		var timeout = null;
		var previous = 0;
		if (!options) options = {};
		var later = function () {
			// 如果禁用第一次首先执行，返回0否则就用previous保存当前时间戳
			previous = options.leading === false ? 0 : _.now();
			// 解除引用
			timeout = null;
			result = func.apply(context, args);
			// 看到一种说法是在func函数里面重新给timeout赋值，会导致timeout依然存在，所以这里会判断!timeout
			if (!timeout) context = args = null;
		};
		return function () {
		    // 获取当前调用时的时间（ms）
			var now = _.now();
			// 如果previous为0并且禁用了第一次执行，那么将previous设置为当前时间
			// 这里用全等来避免undefined的情况
			if (!previous && options.leading === false) previous = now;
			// 还要wait时间才会触发下一次func
			var remaining = wait - (now - previous);
			context = this;
			args = arguments;
			// remaining小于0有两种情况，一种是上次调用后到现在已经到了wait时间
			// 一种情况是第一次触发的时候并且options.leading不为false，previous为0，因为now记录的是unix时间戳，所以会远远大于wait
			// remaining大于wait的情况我自己不清楚，但看到一种说法是客户端系统时间被调整过，可能会出现now小于previous的情况
			// 这两种情形下会立即执行func函数，并把previous设置为now
			if (remaining <= 0 || remaining > wait) {
				if (timeout) {
				    // 清除定时器
					clearTimeout(timeout);
					timeout = null;
				}
				// previous保存当前触发的时间戳
				previous = now;
				result = func.apply(context, args);
				if (!timeout) context = args = null;
			// 如果timeout不存在（当前定时器还存在）
			// 并且options.trailing不为false，这个时候会重新设置定时器，remaining时间后执行later函数
			} else if (!timeout && options.trailing !== false) {
				timeout = setTimeout(later, remaining);
			}
			return result;
		};
	};

```
这段代码看着不多，但是让我纠结了很久，运行的时候主要会有以下几种情况。

### 没有传leading和trailing ###

 1. 第一次触发函数的时候，由于previous为0，而now又非常大，所以会导致remaining为负值，满足下面第一个if判断，所以会立即执行func函数（第一次触发时立即调用）并且用previous记录当前时间戳
 2. 第二次触发的时候由于previous记录了前一次的时间戳，所以now - previous几乎为0，这个时候满足else if里面的判断，会设置一个定时器，这个定时器在remaining时间后执行，所以只要在remaining时间内不管我们再怎么频繁触发，由于不会满足两个if里面的条件，所以都不会执行func，一直到remaining后才会执行func
 3. 之后每次触发都会重复走2的流程

### options.leading: false ###
这种情况和上面情况类似，不过区别在于第一次触发的时候。
由于满足!previous && options.leading === false这个条件，所以previous会被设置为now，这个时候remaining等于wait，所以会走else if的分支，这样就会重复前一种情况下步骤2的流程

### options.trailing: false ###

 1. 由于没有设置leading为false，所以第一次触发就会立即执行一次func
 2. 第二次触发的时候，由于previous保存了上次时间戳，所以remaining <= wait，但是又因为options.trailing为false，这样就不会走if的任何一个分支，一直到now-previous大于wait的时候（也就是过了wait时间后），这样会满足if第一个分支的条件，func会立即被执行一次
 3. 之后重复步骤2

### trailing和leading都为false ###
最好不要这么写，因为会导致一个bug的出现，如果我们在一段时间内频繁触发，这个是没什么问题，但如果我们最后一次触发后停止等待ait时间后再重新开始触发，这时候的第一次触发就会立即执行func，leading为false并没有生效。


不知道有没有人和我一样有这两个疑问，leading为false的时候，真的只是在第一次调用的时候有区别吗？trailing是怎么做到禁用最后一次执行的？
这两个问题让我昨晚睡觉前都还在纠结，还好今天在segmentfault上面有热心的用户帮我解答了。
请直接看第一个回答以及下面的评论区：[关于underscore源码中throttle函数的疑惑？][1]
## leading带来的不同表现 ##
GDUTxxZ大神给了一段代码，执行后不同的表现让我印象深刻。
```
var _now = new Date().getTime()
var throttle = function(func, wait, options) {
  var context, args, result;
  var timeout = null;
  var previous = 0;
  if (!options) options = {};
  var later = function() {
    previous = options.leading === false ? 0 : new Date().getTime();
    timeout = null;
    result = func.apply(context, args);
    if (!timeout) context = args = null;
  };
  return function() {
    console.log(`函数${++i}在${new Date().getTime() - _now}调用`)
    var now = new Date().getTime();
    if (!previous && options.leading === false) previous = now;
    var remaining = wait - (now - previous);
    context = this;
    args = arguments;
    // 如果超过了wait时间，那么就立即执行
    if (remaining <= 0 || remaining > wait) {
      if (timeout) {
        clearTimeout(timeout);
        timeout = null;
      }
      previous = now;
      result = func.apply(context, args);
      if (!timeout) context = args = null;
    } else if (!timeout && options.trailing !== false) {
      timeout = setTimeout(later, remaining);
    }
    return result;
  };
};
var i = 0
var test = throttle(() => {
  console.log(`函数${i}在${new Date().getTime() - _now}执行`)
}, 1000, {leading: false})

setInterval(test, 3000)
```
我将传入leading和没传入leading的情况作了以下比较。
leading为false时：
![leading为false][6]
没有传入leading时：
![leading为true][7]
当两次触发间隔时间大于wait时间的时候，很明显leading为false的时候总会在调用后延迟wait后执行func，而不传leading的时候两者是同时的，调用test的时候就直接运行了func。原本应该是callback => wait => callback

一般情况下当然不会有这种极端情况存在，但是可能出现这种情况。如果在scroll事件中，我们滚动一段距离后停止了，等wait ms后再开始滚动，这个时候如果leading为false，依然会延迟wait时间后执行，而不是立即执行，这也是为什么同时设置leading和trailing为false的时候会出现问题。

## 为什么是禁用最后一次调用 ##
trailing为false时到底是怎么禁用了最后一次调用？这个也一直让我很纠结。同样的，我也写了一段代码，比较了一下两次运行后的不同结果。
```
var _now = new Date().getTime()
var throttle = function(func, wait, options) {
var context, args, result;
var timeout = null;
var previous = 0;
if (!options) options = {};
var later = function() {
previous = options.leading === false ? 0 : new Date().getTime();
timeout = null;
result = func.apply(context, args);
if (!timeout) context = args = null;
};
return function() {
console.log(`函数${++i}在${new Date().getTime() - _now}调用`)
var now = new Date().getTime();
if (!previous && options.leading === false) previous = now;
var remaining = wait - (now - previous);
context = this;
args = arguments;
// 如果超过了wait时间，那么就立即执行
if (remaining <= 0 || remaining > wait) {
  if (timeout) {
    clearTimeout(timeout);
    timeout = null;
  }
  previous = now;
  result = func.apply(context, args);
  if (!timeout) context = args = null;
} else if (!timeout && options.trailing !== false) {
  timeout = setTimeout(later, remaining);
}
return result;
};
};
var i = 0
var test = throttle(() => {
console.log(函数${i}在${new Date().getTime() - _now}执行)
}, 1000, {trailing: false})
window.addEventListener("scroll", test)
```
trailing为false时：
![trailing为false][8]

没有设置trailing时：
![没有设置trailing][9]

这两张图很明显的不同就是设置了trailing的时候，最后一次总是"执行"，而未设置trailing最后一次总是"调用"，少了一次执行。

我们可以假设在一种临界的场景下，比如在倒数第二次执行func后的 (wait-1) 的时间内。
如果设置了trailing，因为无法走setTimeout，所以只能等待wait时间后才能立即调用func，所以在（wait-1）的时间内无论我们触发了多少次都不会执行func函数。
如果没有设置trailing，那么肯定会走setTimeout，在这个期间触发的第一次就会设置一个定时器，等到wait时间后自动执行func函数，到（wait-1）的这段时间内不管我们触发了多少次，反正第一次触发的时候就已经设置了定时器，所以到最后一定会执行一次func函数。

## 总结 ##
很久以前就使用过throttle函数，自己也实现过简单的，但是看到underscore源码后才发现原来还会有这么多令人充满想象的场景，自己所学的这点知识真的是皮毛。
我知道自己平时叙述比较罗嗦，语言又比较无聊，希望大家可以理解，如果看完还不懂，建议结合下面的参考链接。
本文有错误和不足之处，也希望大家能够指出。

## 参考链接：##

 1. [关于underscore源码中throttle函数的疑惑？][2]
 2. [ underscore 函数节流的实现][3]
 3. [ Underscore之throttle函数源码分析以及使用注意事项
][4]
 4. [浅谈 Underscore.js 中 _.throttle 和 _.debounce 的差异][5]

 


  [1]: https://segmentfault.com/q/1010000013899949
  [2]: https://segmentfault.com/q/1010000013899949?_ea=3493310
  [3]: https://github.com/hanzichi/underscore-analysis/issues/22
  [4]: http://www.easyui.info/archives/1853.html
  [5]: https://blog.coding.net/blog/the-difference-between-throttle-and-debounce-in-underscorejs
  [6]: https://pic1.zhimg.com/80/v2-7f14598aced570beef335ef7485c56f0_hd.jpg
  [7]: https://pic2.zhimg.com/80/v2-1bedfe844882f6c1967b4ec9d5029f3f_hd.jpg
  [8]: https://pic1.zhimg.com/80/v2-e901c58912a573086e6db8bd36b0c2cf_hd.jpg
  [9]: https://pic4.zhimg.com/80/v2-25ae6fdaa9472253d9b1fbd383edd54a_hd.jpg

  <head>
    <script src="//cdn1.lncld.net/static/js/3.0.4/av-min.js"></script>
    <script src='//unpkg.com/valine/dist/Valine.min.js'></script>
</head>
<body>
    <div id="comment"></div>
</body>
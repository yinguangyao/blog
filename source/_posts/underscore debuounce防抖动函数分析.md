---
title: underscore debuounce防抖动函数分析
date: 2018-03-23 22:15:06
tags:
- underscore
- 前端
- 编程
categories: [前端, underscore]
---
本文是underscore源码剖析系列第六篇文章，上节我们介绍了throttle节流函数的实现，这节将会介绍一下节流函数的兄弟 —— debounce防抖动函数。
throttle函数是在高频率触发的情况下，为了防止函数的频繁调用，将其限制在一段时间内只会调用一次。而debounce函数则是在频繁触发的情况下，只在触发的最后一次调用一次，想像一下如果我们用手按住一个弹簧，那么只有等到我们把手松开，弹簧才会弹起来，下面我用一个电梯的例子来介绍debounce函数。
## 电梯 ##

假如我下班的时候去坐电梯，等了一段时间后，电梯正准备关上门下降，这个时候一个同事走了过来，电梯门被打开，这样电梯就会继续等一段时间，如果中间一直有人进来，那么电梯就一直不会下降，直到最后一个人进来后过了一定时间后还没有下一个人进来，这时电梯才会下降。
## 应用场景 ##
除了电梯，事实上我们还有很多应用场景，比如我用键盘不断输入文字，我希望等最后一次输入结束后才会调用接口来请求展示联想词，如果每次输入一个字的时候就会调用接口，这样调用未免太过于频繁了。
<!-- more -->
未使用debounce的输入：
![未使用debounce][2]
使用debounce的输入：
![使用debounce][3]
## 简单的debounce ##
知道debounce的工作原理了，我们可以先自己实现一个比较简单的debounce函数。
```
function debounce(func, wait) {
	var timeout,
		args, 
		context
	var later = function() {
		func.apply(context, args)
		timeout = context = args = null
	}
	return function() {
		context = this
		args = arguments
		// 每次触发都清理掉前一次的定时器
		clearTimeout(timeout)
		// 只有最后一次触发后才会调用later
		timeout = setTimeout(later, wait)
	}
}
```
麻雀虽小，五脏俱全。
不过这个函数还是有很多问题，比如每次触发都要反复的清除和设置定时器，我们来看一下underscore的实现。
## underscore debounce ##
```
// debounce函数传入三个参数，分别是要执行的函数func，延迟时间wait，是否立即执行immediate
// 如果immediate为true，那么就会在wait时间段一开始就执行一次func，之后不管触发多少次都不会再执行func
// 在类似不小心点了提交按钮两下而提交了两次的情况下很有用
_.debounce = function (func, wait, immediate) {
		var timeout, args, context, timestamp, result;

		var later = function () {
		    // 这个是最关键的一步，因为每次触发的时候都要记录当前timestamp
		    // 但是later是第一次触发后wait时间后执行的，_now()减去第一次触发时的时间当然是等于wait的
		    // 但是如果后续继续触发，那么_.now() - timestamp肯定会小于wait
		    // last是执行later的时间和上一次触发的时间差
			var last = _.now() - timestamp;
            // 如果在later执行前还有其他触发，那么就会重新设置定时器
            // last >= 0应该是防止客户端系统时间被调整
			if (last < wait && last >= 0) {
				timeout = setTimeout(later, wait - last);
			// 如果last大于等于wait，也就是说设置timeout定时器后没有再触发过
			} else {
				timeout = null;
				// 这个时候如果immediate不为true，就会立即执行func函数，这也是为什么immediate为true的时候只会执行第一次触发
				if (!immediate) {
					result = func.apply(context, args);
					// 解除引用
					if (!timeout) context = args = null;
				}
			}
		};

		return function () {
			context = this;
			args = arguments;
			// 每次触发都用timestamp记录时间戳
			timestamp = _.now();
			// 第一次进来的时候，如果immediate为true，那么会立即执行func
			var callNow = immediate && !timeout;
			// 第一次进来的时候会设置一个定时器
			if (!timeout) timeout = setTimeout(later, wait);
			if (callNow) {
				result = func.apply(context, args);
				context = args = null;
			}

			return result;
		};
	};
```
underscore的实现比较巧妙，为了防止出现我们上面那种不停地清除、设置定时器的情况，underscore则会在每次触发时都记录时间戳，在wait时间后定时器触发执行later函数时计算当前时间和时间戳之差，如果小于wait时间，那么在之后肯定还有其他触发，这时再重新设置定时器，这样不仅解决了上面的问题，也保证了最后一次触发后wait时间才会执行func。

同时，在我们的基础之上，underscore加入了immediate参数。如果传入的immediate为true，那么只会在第一次进来的时候立即执行。很明显在上面代码中func执行只有两处，一个是callNow判断里面，一个是!immediate判断里面，所以这样保证了后续触发不会再执行func。

**参考链接：**
1、[浅谈throttle以及debounce的原理和实现
][1]


  [1]: https://segmentfault.com/a/1190000010983733
  [2]: https://pic1.zhimg.com/v2-b702de1108e20324c9fa8c492825a14a_b.gif
  [3]: https://pic3.zhimg.com/v2-86ef7d20bd65b88884538ade33dfc3c8_b.gif
  <head>
    <script src="//cdn1.lncld.net/static/js/3.0.4/av-min.js"></script>
    <script src='//unpkg.com/valine/dist/Valine.min.js'></script>
</head>
<body>
    <div id="comment"></div>
</body>
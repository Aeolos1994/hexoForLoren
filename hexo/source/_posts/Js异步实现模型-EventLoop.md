---
title: Js异步模型-EventLoop
date: 2018-06-14 17:41:20
tags: JavaScript
---
众所周知，JS作为一门脚步语言，具有**事件驱动**、**异步**、**单线程**等特点，今天我们就浅析一下关于JS如何做到异步的以及相关周边问题；
<!--more-->
### 如何实现异步操作
JS若要实现异步操作，大家最容易想到的可能会是setTimeout或者Ajax，今天我们就从实现最简单的setTimeout入手；
	
	setTimeout(function () {
		//do something here
	},1000);
	...
	
	=>
	
	
	setTimeout(function () {
		//do something here
	},0);
	...

上面这部分代码就是最简单的一个setTimeout运用了，它表示在一秒之后执行一些指令。如果我们将setTimeout的第二个参数，即1000修改为0，那么它会立即执行吗？答案是不会的，因为setTimeout是异步的（准确的说，设置定时器这个动作是同步的，但是timeout触发的事件是异步的，后边我们详细的论述它）。异步操作将会在所有同步操作完成后执行，所以如果整个任务中，设置定时器后的同步任务如果执行耗时超过一秒，将会延迟setTimeout的回调执行；
顺便一提，**即使整个任务只有一个setTimeout且定时器时长设置为0，回调函数也无法像同步方法一样立马执行**，至少会延迟**4ms**，这是由HTML5规范规定的。

>In modern browsers, setTimeout()/setInterval() calls are throttled to a minimum of once every 4ms when successive calls are triggered due to callback nesting (where the nesting level is at least a certain depth), or after certain number of successive intervals.

### event-loop
那么这个过程中，到底发生了什么呢？ 要想知道发生了什么，有一个概念或者说机制必须要知道，那就是event-loop;

>   Each 'thread' gets its own event loop, so each
 	web worker gets its own, so it can execute
 	independently, whereas all windows on the same 
 	origin share an event loop as they can 
 	synchronously communicate. The event loop runs 
 	continually, executing any tasks queued. 
 	
翻译一下：**每个线程都有自己的event-loop(事件循环)，因为每个web worker由于都有自己的线程，所以都有自己event-loop，这样才能保证web worker可以相互独立的运行，`同源 `的所有窗口共享一个event-loop，所以它们才可以同步的进行通信(比如postMessage)。event-loop不断循环并执行队列中的任务。**

> #### NOTE:
> 这里所说的同源不是指同源策略的同源。指的是`window.open`、`iframe`、`window.frame`等方法打开的同源窗口，具体细节可参考[Shared Event-loop for Same-Origin Windows](http://hassansin.github.io/shared-event-loop-among-same-origin-windows)
 

> 所以，为了协调事件，用户交互，脚本，渲染，网络等，用户代理必须使用本节所述的event loop。



是不是有点晦涩难懂，下面结合event-loop我用一段代码结合图文详细解释一下代码的执行过程来进行理解：

	function sthBeforeTimeout () {
		console.log('before');
	}
	
	setTimeout(function () {
		console.log('timeout');
	})
	
	function sthAfterTimeout () {
		console.log('after')
	}
	
	sthBeforeTimeout();
	sthAfterTimeout();

![Event Loop](http://images2015.cnblogs.com/blog/903320/201705/903320-20170516235510557-6476946.png)

1. 所有声明式函数都在代码执行之前预处理，所以在真正的代码执行时，所以第一个被压入Stack(主线程)中的方法为setTimeout，它触发了一个定时器，定时器到时触发timeout事件，并将其push到event queue中；
2. 执行方法 sthBeforeTimeout
3. 执行方法 sthAfterTimeout
4. event-loop开始循环检查event queue，发现有待处理的事件则按照FIFO的原则将其处理，处理方式为检查事件是否有关联方法或回调，如有则push到Stack中并执行它；

上述简化后的过程，就是event-loop实现异步的过程，关键点就在于，**event-loop是在所有的同步代码执行完成后，才去检查event-queue**；

### macroTask & microTask
说到这里，仿佛我们已经大概明白js的异步实现原理，但是不妨看一下下面的代码，猜一下在**Chrome**控制台中打印的会是什么？

	console.log('start');
	
	setTimeout(function () {
		console.log('timeout')
	})
	
	Promise.resolve().then(function(){
		console.log('then1');
	}).then(function () {
		console.log('then2');
	})
	
	console.log('end');
	
按照我们的正常理解，同步代码会先执行毫无疑问，然后是异步的，event-loop按照FIFO原则，循环的拿出相对早的那个事件并执行对应回调，那么执行顺序将会是’timeout‘、’then1‘、'then2'；

但是你实践了就会发现，并不是这样的，打印顺序是
	
	start
	end
	then1
	then2
	timeout
	
为何会这样子呢，不是说好的FIFO么，这里就涉及更深入的原理和一个概念**task(macroTask)宏任务**与**microTask微任务**；首先明确，event-loop有两种任务(笔者认为其实准确说是事件)队列，一种是macroTask queue，队列里对应的任务就是marcoTask，另一种就是microTask queue也就对应microTask；

那么既然是两种queue，js又是单线程的，肯定要分出个高下，也就是到底谁先被读取、被执行。根据HTML规范:
>If the stack of script settings objects is now empty, perform a microtask checkpoint.
>	— HTML: Cleaning up after a callback step 3

![Event Loop](https://github.com/aooy/aooy.github.io/blob/master/blog/issues5/img/application1.jpg?raw=true)

如上图所示也就是macroTask被执行完毕，将会去检查微任务队列，看是否有需要处理的事件。而且根据定义还能看出整个代码的执行被浏览器认为是一个macroTask。那么看到这里，就可以意识到Promise.then和setTImeout是属于不同的任务种类的了，在Chrome中，Promise.then是作为一个微任务存在的，在macroTask队列检查完成后没有待执行的任务，就去检查微任务队列，发现有Promise的完成事件存在，则去执行then方法对应的回调，然后再次检查微任务队列，直到任务队列被清空。整理并细化一下规范则event-loop执行顺序如下表：

	1.MacroTask: 在tasks队列中选择最老的一个task，如果没有可选的任务，则跳到下边的microtasks步骤。
	2.将上边选择的task设置为正在运行的task。
	3.运行被选择的task。
	4.将event loop的currently running task变为null。
	5.从task队列里移除前边运行的task。
	6.Microtasks: 执行microtasks任务检查点。
		(1).将microtask checkpoint的flag设为true。
		(2).Microtask queue handling: 如果event loop的microtask队列为空，直接跳到第八步（Done）。
		(3).在microtask队列中选择最老的一个任务。
		(4).将上一步选择的任务设为event loop的currently running task。
		(5).运行选择的任务。
		(6).将event loop的currently running task变为null。
		(7).将前面运行的microtask从microtask队列中删除，然后返回到第二步（Microtask queue handling）。
		(8).Done: 每一个environment settings object它们的 responsible event loop就是当前的event loop，会给environment settings object发一个 rejected promises 的通知。
		(9).清理IndexedDB的事务。
		(10).将microtask checkpoint的flag设为flase。.
	7.更新渲染（Update the rendering）...
	8.如果这是一个worker event loop，但是没有任务在task队列中，并且WorkerGlobalScope对象的closing标识为true，则销毁event loop，中止这些步骤，然后进行定义在Web workers章节的run a worker。
	9.返回到第一步。

**所以，Chrome中then1、then2先于timeout被打印的关键点就在于，then为微任务，setTimeout为宏任务，在同步代码作为宏任务执行完毕之后，event-loop将会先去检查微任务队列并执行回调；**

在上面的关于打印的问题，我一直在强调在chrome中，是因为规范中并未明确定义Promise.then到底是macroTask还是microTask，所以各个浏览器的event-loop实现方式可能不一致，若浏览器实现是将then作为宏任务的话，那么就会是timeout先于then1、then2打印了；

### task origin

上面说了两种task queue，那么我们再深入一些，每种task queue到底内部是什么样子的呢；首先还是一坨规范甩在下面，虽然枯燥晦涩，但对于我们对于底层实现原理的理解还是很有好处的。

>An event loop has multiple task sources which guarantees execution order within that source (specs such as IndexedDB define their own), but the browser gets to pick which source to take a task from on each turn of the loop. This allows the browser to give preference to performance sensitive tasks such as user-input.

如规范所描述的每个event-loop都有多个marcoTask queue，这是因为宏任务有多个任务源，每个任务源都有一个对应的marcoTask queue，每个任务源对应的queue都可以保证是FIFO的，在多个marcoTask queue中规范允许浏览器主动选择先执行哪个queue，并以此实现对于用户操作的优先响应。而每个event-loop都只有一个microTask queue，如下图总结。

![event-loop](https://github.com/aooy/aooy.github.io/blob/master/blog/issues5/img/eventLoop.jpg?raw=true)

让我们最后探究一下，到底不同任务类型的任务源是怎么区分的
	
	macroTask Origin
		1. setTimeout
		2. setInterval
		3. setImmediate
		4. I/O
		5. UI rendering

对于 **microTask Origin** HTML规范中并没有明确说明，根据浏览器实现和参考一些资料我们普遍认为下列为微任务源
	
	microTask Origin
		1.process.nextTick
		2.promises
		3.Object.observe
		4.MutationObserver
		
### 总结
以上叙述基本就是event-loop的原理，懂了event-loop的工作方式，就明白了runtime为现代浏览器的js是如何实现异步的了；以一种通俗易懂方法解释的话可以这么形容：

> 如果主线程是一辆货运车，那么event-loop就是分拣工，而任务就是货物，微任务是奢侈品，宏任务是日常消费品；
> 
> 每种日常消费品都有自己专有的流水线(宏任务中多个任务源对应多个task queue)运输,所以日常消费品有多个流水线，而所有的奢饰品不分种类全部由一个流水线运输。
> 
> 分拣工的工作规则是先去清空日常消费品的多个流水线，根据产品的价格由高到低在离他最近的日消品中依次将货物扔到货运车上。
> 
> 每次分拣工都只拿一个日消品扔车上（毕竟不值钱，看不上），没有就不拿，然后立马再去奢侈品流水线看有没有待处理的奢侈品，有的话就把所有的奢侈品依次都轻轻的放在货运车上，没有了的话就再跑去多条日消品流水线那边检查并根据之前说的规则把货物扔车里。
> 
> 分拣工就这样一直来回检查来回跑...

总结并简化一下event-loop的工作方式就是：

	-> 执行整个script(作为宏任务) 
	-> 检查微任务队列并执行直到微任务队列为空
	-> 根据macroTask queue优先级执行宏任务
	-> 更新视图
	-> 检查微任务队列并执行直到微任务队列为空
	-> ...

#### 参考资料与图片来源
1. [Jake`s blog](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
2. [aooy`s github](https://github.com/aooy/blog/issues/5)
3. [朴灵批注：再看Event-loop-阮一峰](http://blog.csdn.net/lin_credible/article/details/40143961)
4. [Standard ECMA-262](http://www.ecma-international.org/ecma-262/6.0/#sec-jobs-and-job-queues)
5. [HTML Standard](https://html.spec.whatwg.org/#event-loops)
6. [Concurrency model and Event Loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop)


	


	
	






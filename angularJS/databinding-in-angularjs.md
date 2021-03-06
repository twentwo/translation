#AngularJS中的数据绑定

这是一篇来自Stack Overflow 的问答，答案最新修改时间是2014-11-21。译者对其中的字句进行了调整，力求解释清楚答案的意思。不过由于我的水平有限，文章中仍会有很多纰漏，欢迎指正。 

译者感觉答主并没有正面回答提问者的问题，但由于答主解释了脏检查的机制，也不失是一篇很好的答案。

[原文连接](http://stackoverflow.com/questions/9682092/databinding-in-angularjs) 翻译[RChee](https://github.com/rchee)

##Q：AngularJS框架里面的数据绑定到底是怎么工作的呢？

我在他们的官网找不到具体的技术细节。
多多少少还是可以猜到他是怎么把数据从view传递到model的。
（译注：指的是提问者可以猜到，angular建立了一系列的事件监听器，当视图中触发了监听器时同步更新模型）
不过，AngularJS到底是怎么发现model上的属性发生了变化呢，它并没有设置setters和getters呀？
我觉得如果用了JavaScript watchers应该可以做到这一点。
但在IE6和IE7中并不支持这个特性。
现在问题来了，举个例子，对于下面这种情况，AngularJS是怎么知道我改变了属性，并且更新view的呢？

``myobject.myproperty="new value";``

##A：

AngularJS 会记录值并对拿他和旧的值做比较。
这就是**脏检查**机制。
如果有某个值发生了变化，他会触发相应的change事件。

当你从一个没有AngularJS的世界进入AngularJS的世界时，你会调用`$apply()`函数。而`$apply()`函数又会去调用` $digest()`。
这里的digest做的就是基本脏检查工作。
它可以在所有的浏览器里面按我们预想的那样工作。

对比一下脏检查机制(AngularJS)和change侦听器机制(KnockoutJS and Backbone.js)吧：<br/>
虽然脏检查看起来简单，甚至有时候效率很低（稍后会提到），他却总是可以保证代码的可读性（semantically correct）。change侦听器需要对一些奇奇怪怪的情况做特殊的处理，需要用类似依赖跟踪（dependency tracking）的方式保证代码的可读性。
KnockoutJS中的依赖跟踪虽然是个很有趣的特性，但在AngularJS的世界中压根就不需要他。


###Change侦听机制的问题：

* 如果浏览器不支持的话，语法会变得很糟糕。
是的，你会提到代理，不过并不是所有情况下代理都能保证代码的可读性，更何况老式浏览器里面压根没有代理。
最起码，脏检查机制允许你使用像 [POJO](http://en.wikipedia.org/wiki/Plain_Old_Java_Object)这样的简单对象，而 KnockoutJS 和  Backbone.js 强制你去继承他们的类，还必须要用setter和getter这样的访问器（accessors）去读写数据。

* 不能将多次修改合并到一起。
例如你有一些数据要插进数组，当你使用循环语句去插入的时候，每当你添加一个元素都会触发change事件并导致重绘UI。
这会导致性能变得很差。
而实际上我们只需要在最后更新一次UI就够了。
change事件机制会导致修改过于频繁（too fine grained）。

* 由于setter会立即触发一个事件，change监听机制会带来一个问题。那就是在change侦听器里可以进一步的修改数据，最终触发更多的事件。
如果你的栈里同时触发了很多个change事件，这会变得很糟。
比如，因为某些原因，你要让两个数组保持同步的状态。
你可以只在其中一个数组里面添加新元素，不过每次你的添加操作都会触发一个change事件，全世界对这个问题看法不一。
这个问题和线程锁很像。JavaScript通过每个回调完全执行并完成避免了这个问题。
因为setters会带来如此深远但又难以察觉的影响，change事件会打断这个过程。这又一次引入了线程同步问题。
由此我们可以得出结论，其实我们只希望事件侦听器延迟执行，并且保证同一时间只有一个侦听器在执行。因此任何的代码都可以轻松的改变数据，并且知道这时候只有这段代码在运行。

###那性能方面呢？

因为脏检查机制效率比较低，你也许会觉得它运行的太慢了。
现在我们来看看真实的运行数据是什么样的吧，首先我们做个说明。

我们人类是这样的：

* 反应慢 — 我们察觉不到 50 ms 内发生的事情，对我们来说就是一瞬间。

* 精力有限 — 你不可能在一个页面里面同时展示超过 2000 条信息。只要是超过这个的肯定是个糟糕的UI设计，用户也不可能同时看这么多。

所以真正的问题是：50 ms 内到底能做多少次比较操作呢？
因为有太多的因素需要考虑了，这个问题有点难回答。不过这里有个测试样例，让你看看创建了一万个watchers时的情况：[http://jsperf.com/angularjs-digest/6] 
在现代浏览器里面只用了不到 6 ms。
在IE8里面也不过是 40 ms 的事情。
所以到了今天这个时候，即使在比较慢的浏览器里面，比较带来的性能开销也不成问题了。
不过注意了，这些比较要尽量简单才不会超时。
可惜在 AngularJS 中一不小心就会插入了一些耗时较长的比较，所以如果乱来的话，也很容易设计出一些性能低的应用。
希望能有一些性能分析工具来查看到底是哪些比较操作导致了性能问题。这样我们就能更好的找到答案了。
实际上，在游戏和GPU中也采用了脏检查机制。
在游戏里，每次刷新屏幕的速度越慢（一般是50-60 hz, 或者说每 16.6-20 ms 刷新一次），性能就会显得越糟糕。所以对于游戏开发者会通过减小重绘的数量来加快FPS。


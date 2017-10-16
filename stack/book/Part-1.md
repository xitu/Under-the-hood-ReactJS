## 第 1 部分

[![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/part-1.svg)](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/part-1.svg)

<em>1.0 第 1 部分(点击查看大图)</em>

### 事务

那么，组件实例应该以某种方式与React的生态系统**相关联**，当然会对其**产生一些影响**。有一个专门的模块 `ReactUpdates` 有助于此。 正如大家所知, **React 以块形式执行更新**，这意味着它会收集操作然后**一起**执行。
这样做反而更好，因为这样允许为整个项目列表（块）只应用一次某些**前置条件**和**后置条件**，而不是为每个项目都应用。


什么真正有助于解决这个前/后处理？对， **事务**！对某些人来说，**事务**可能是一个新术语，至少对UI方面的需求来说。所以我们再来谈一下，接下来我们从一个简单的例子开始。

想象一下 `通讯通道`。你需要开启连接，发送消息，然后关闭连接。 如果你逐个发送消息，大多数情况下就这么做。相对的，你也可以只开启一次连接，发送所有挂起的消息然后关闭连接。


[![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/communication-channel.svg)](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/communication-channel.svg)

<em>1.1 非常真实的事务示例 (查看大图)</em>

好的，让我们再想想更多抽象的东西。想象一下，在执行操作期间，“发送消息”是您要执行的任何操作，“打开/关闭连接”是预处理/后处理。 然后，再想想一下，你可以分别定义任何 open/close 对，并使用任何方法（我们可以将它们命名为 `wrapper` ,因为事实上每一对都包装动作方法）。听起来很酷，不是吗？

我们回到 React。 事务是 React 中广泛使用的模式。除了包装行为外，事务允许应用程序重置事务流，如果事务已在进行中则阻止同时执行，等等。有很多不同的事务类，它们每个都描述具体的行为，它们都继承自`Transaction` 模块。事务类之间的主要区别是具体的事务包装器的列表的不同。包装器只是一个包含初始化和关闭方法的对象。

所以，**我的想法是**：
* 调用每个 wrapper.initialize 并缓存起来（可以进一步使用）
* 调用事务方法本身
* 调用每个 wrapper.close

[![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/transaction.svg)](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/transaction.svg)

<em>1.2 事务实现 (点击查看大图)</em>

我们来看看 React 中的一些**其他事务用例**：
* 即使在发生意外错误的情况下，也可以在 Reconciliation 恢复选择之前／之后保留 input 选择范围。
* 在重排DOM时，停用事件，防止模糊/焦点，同时保证事件系统重新启动。
* Flushing a queue of collected DOM mutations to the main UI thread after a reconciliation takes place in a worker thread.
在 worker 线程中发生 Reconciliation 后，将收集到的DOM突变的队列刷新到主UI线程。
* 在渲染新内容后调用任何收集的 `componentDidUpdate` 回调。

好吧，让我们回到具体案例。

正如我们看到的， React 使用  `ReactDefaultBatchingStrategyTransaction` (1)。我们前文提到过，事务最重要的是它的包装器。所以，我们可以看看包装器，并弄清楚事务是什么。好，这里有两个包装器：`FLUSH_BATCHED_UPDATES`，`RESET_BATCHED_UPDATES`。我们来看下面的代码：

```javascript
//\src\renderers\shared\stack\reconciler\ReactDefaultBatchingStrategy.js#19
var RESET_BATCHED_UPDATES = {
	  initialize: emptyFunction,
	  close: function() {
		ReactDefaultBatchingStrategy.isBatchingUpdates = false;
	  },
};

var FLUSH_BATCHED_UPDATES = {
	 initialize: emptyFunction,
	 close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates),
}

var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];
```

所以，你可以看看事务的写法。此代码中事务没有前置条件。 `initialize` 方法是空的,但其中一个 `close` 方法很有趣。它调用了`ReactUpdates.flushBatchedUpdates`。 这意味着什么? 它实际上对对脏组件的验证进一步重新渲染。所以，你理解了，对吗？我们调用 mount 方法并包装在具体的事务中，因为在 mount 执行后，React 检查已加载的组件有什么影响并更新它们。

好，我们来看看包装在事务中的方法。 那么事实上，它引发了另外一个事务...


### 好，**第 1 部分**我们讲完了

我们来回顾一下我们学到的。我们再看一下这种模式，然后去掉冗余的部分：

[![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/part-1-A.svg)](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/part-1-A.svg)

<em>1.3 第 1 部分简化版 (点击查看大图)</em>

然后我们适当再调整一下：

[![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/part-1-B.svg)](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/part-1-B.svg)

<em>1.4 第 1 部分简化和重构 (点击查看大图)</em>

Nice. In fact, that’s all that happens here. So, we can take the essential value from *Part 1* and use it for the final `mounting` scheme:很好，实际上，下面的示意图就是我们所讲的。因此，我们可以理解**第 1 部分**的本质，并将其用于最终的 `mount` 方案：

[![](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/part-1-C.svg)](https://rawgit.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/master/stack/images/1/part-1-C.svg)

<em>1.5 第 1 部分本质(点击查看大图)</em>

完成！

[下一节: 第 2 部分 >>](./Part-2.md)

[<< 上一节 第 0 部分](./Part-0.md)


[主页](../../README.md)

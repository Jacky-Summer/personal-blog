# 浅析 requestIdleCallback

## 前言

最近在研究 React Fiber 相关的知识，上一篇文章 [浅谈对 React Fiber 的理解](https://juejin.cn/post/6926432527980691470) 简单提到了 requestIdleCallback， React 源码中 polyfill 了这个方法，了解它对 Fiber 也能有进一步理解。本篇会深入介绍下这个方法。

## requestIdleCallback 是什么

requestIdleCallback 是 window 属性上的方法，它的作用是在浏览器一帧的剩余空闲时间内执行优先度相对较低的任务。

## 为什么需要 requestIdleCallback

在网页运行中，有很多耗时但又不是那么重要的任务。这些任务和重要的任务如对用户的输入作出及时响应的之类的任务，它们共享事件队列。如果两者发生冲突，用户体验会很糟糕。

requestIdleCallback 就解决了这个痛点，requestIdleCallback 会在每一帧结束时并且有空闲时间执行回调。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ea116d4702c45edaf176599952cb0af~tplv-k3u1fbpfcp-watermark.image)

假设需要大量涉及到 DOM 的操作的计算，在运算时，浏览器可能就会出现明显的卡顿行为，甚至不能进行任何操作，因为是 JS 单线程，就算用在输入处理，给定帧渲染和合成之后，用户的主线程就会变得空闲，直到下一帧的开始。

而这些空闲时间可以拿来处理低优先级的任务，React16 的调度策略异步可中断，其中关键就靠的这个（polyfill）方法功能；React 把任务细分（时间切片），在浏览器空闲的时间去执行，从而尽可能地提高渲染性能。

时间切片的本质是模拟实现 requestIdleCallback

讲到这里，从 React15 到 React16 Fiber，对整体性能来说是大优化了；但要知道的是，React16 相对 15 做出的优化，并不是大大减少了任务量，你写的代码的任务总量并没有变化，只是把空闲时间利用起来了，不停的干活，就能更快的把活干完；这只是其中一个角度，React 还做了区分优先级执行等等。

## 屏幕刷新率和 FPS 的关系

当前大多数的屏幕刷新率都是 60HZ，一秒 60 帧（FPS 为 60），也就是每秒屏幕刷新 60 次，此时一帧的时间为 16.7ms（1000ms/60）低于 60HZ 人眼就会感知卡顿掉帧等情况。

> 浏览器的一帧说的就是一次完整的重绘。

前端浏览器所说的渲染频率 FPS(Frames Per Second)是每秒传输帧数，即是每秒刷新的次数，理论上 FPS 越高人眼觉得界面越流畅。

## window.requestIdleCallback() 用法

window.requestIdleCallback()方法将在浏览器的空闲时段内调用的函数排队。这使开发者能够在主事件循环上执行后台和低优先级工作，而不会影响延迟关键事件，如动画和输入响应。函数一般会按先进先调用的顺序执行，然而，如果回调函数指定了执行超时时间 timeout，则有可能为了在超时前执行函数而打乱执行顺序。

你可以在空闲回调函数中调用 requestIdleCallback()，以便在下一次通过事件循环之前调度另一个回调。

```javascript
var handle = window.requestIdleCallback(callback[, options])
```

返回一个 ID 标识符。

```
callback：
一个在事件循环空闲时即将被调用的函数的引用。函数会接收到一个名为 deadline 的参数，这个参数可以获取当前空闲时间以及回调是否在超时时间前已经执行的状态；该对象上有两个属性：
  - timeRemaining：timeRemaining属性是一个函数，函数的返回值表示返回当前空闲时间的剩余时间
  - didTimeout：didTimeout属性是一个布尔值，如果didTimeout是true，那么表示本次callback的执行是因为超时的原因

options 可选
包括可选的配置参数。具有如下属性：
timeout：如果指定了timeout并具有一个正值，并且尚未通过超时毫秒数调用回调，那么回调会在下一次空闲时期被强制执行，尽管这样很可能会对性能造成负面影响。
```

```javascript
function myNonEssentialWork(deadline) {
  while ((deadline.timeRemaining() > 0 || deadline.didTimeout) && tasks.length > 0) {
    doWorkIfNeeded()
  }

  if (tasks.length > 0) {
    requestIdleCallback(myNonEssentialWork)
  }
}

requestIdleCallback(myNonEssentialWork, 5000)
```

来看一个实际的例子：

```javascript
requestIdleCallback(myWork)

// 一个任务队列
let tasks = [
  function t1() {
    console.log('执行任务1')
  },
  function t2() {
    console.log('执行任务2')
  },
  function t3() {
    console.log('执行任务3')
  },
]

// deadline是requestIdleCallback返回的一个对象
function myWork(deadline) {
  console.log(`当前帧剩余时间: ${deadline.timeRemaining()}`)
  // 查看当前帧的剩余时间是否大于0 && 是否还有剩余任务
  if (deadline.timeRemaining() > 0 && tasks.length) {
    // 在这里做一些事情
    const task = tasks.shift()
    task()
  }
  // 如果还有任务没有被执行，那就放到下一帧调度中去继续执行，类似递归
  if (tasks.length) {
    requestIdleCallback(myWork)
  }
}
```

我的运行结果如下（每次运行，不同机器运行都不一样）：

```
当前帧剩余时间: 15.120000000000001
执行任务1
当前帧剩余时间: 15.445000000000002
执行任务2
当前帧剩余时间: 15.21
执行任务3
```

如果是因为 timeout 回调才得以执行的话，其实用户就有可能会感觉到卡顿了，因为一帧的执行时间必然已经超过 16ms 了

## requestIdleCallback 方法的缺陷

这个方法理论上可行，但为什么 React 团队又 polyfill 这个方法呢？

1. 浏览器兼容不好的问题

2. requestIdleCallback 的 FPS 只有 20，也就是 50ms 刷新一次，远远低于页面流畅度的要求，所以 React 团队需要自己实现。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc589c5d1b0f4c30997ed0fda92f68fe~tplv-k3u1fbpfcp-watermark.image)

> 注意：timeRemaining 最大为 50ms，是有根据研究得出的，即是说人对用户输入的 100 毫秒以内的响应通常被认为是瞬时的，不会被人察觉到。将空闲时间限制在 50ms 内意味着即使在闲置任务开始后立即发生用户操作，用户代理仍然有剩余的 50ms 可以在其中响应用户输入而不会产生用户可察觉的滞后。

## requestIdleCallback 和 requestAnimationFrame 区别

- requestAnimationFrame 的回调会在每一帧确定执行，属于高优先级任务
- requestIdleCallback 的回调则不一定，有空闲时间才执行，属于低优先级任务。

> window.requestAnimationFrame() 告诉浏览器——你希望执行一个动画，并且要求浏览器在下次重绘之前调用指定的回调函数更新动画。该方法需要传入一个回调函数作为参数，该回调函数会在浏览器下一次重绘之前执行

requestAnimationFrame 比起 setTimeout、setInterval 的优势主要有两点：

- requestAnimationFrame 会把每一帧中的所有 DOM 操作集中起来，在一次重绘或回流中就完成，并且重绘或回流的时间间隔紧紧跟随浏览器的刷新频率，一般来说，这个频率为每秒 60 帧。

- 在隐藏或不可见的元素中，requestAnimationFrame 将不会进行重绘或回流，这当然就意味着更少的的 cpu，gpu 和内存使用量。

## 模拟实现 requestIdleCallback

### setTimeout 模拟

当然一般不会用 setTimeout 的，因为误差很大

```javascript
window.requestIdleCallback = function (handler) {
  let startTime = Date.now()

  return setTimeout(function () {
    handler({
      didTimeout: false,
      timeRemaining: function () {
        return Math.max(0, 50.0 - (Date.now() - startTime))
      },
    })
  }, 1)
}
```

React 团队则是用 requestAnimationFrame 和 postMessage 模拟实现的，本篇就不讲了，有兴趣的可以去了解看看。

## 参考

- [熟悉 requestidlecallback 到了解 react ric polyfill 实现](https://juejin.cn/post/6844904196345430023)
- [react 中 requestIdleCallback 的实现原理](https://juejin.cn/post/6861590253434585096)

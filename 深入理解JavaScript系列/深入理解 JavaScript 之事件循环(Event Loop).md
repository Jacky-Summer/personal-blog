# 深入理解 JavaScript 之事件循环(Event Loop)

## 前言

深入理解 JavaScript 系列已经很久没更新了，之前的系列文章如下：

- [深入理解 JavaScript 之变量提升](https://juejin.im/post/6844903976765227016)
- [深入理解 JavaScript 之由一道题来思考闭包](https://juejin.im/post/6844903977511829517)
- [深入理解 JavaScript 之执行上下文和执行栈](https://juejin.im/post/6844904006179913741)
- [深入理解 JavaScript 之执行上下文和变量对象](https://juejin.im/post/6844904006901301255)
- [深入理解 JavaScript 之作用域链与闭包](https://juejin.im/post/6844904079051718664)
- [深入理解 JavaScript 之原型与原型链](https://juejin.im/post/6844904065785315341)
- [深入理解 JavaScript 之 new 原理及模拟实现](https://juejin.im/post/6844904070113656839)
- [深入理解 JavaScript 之获取数组中的最大值方法（this,apply）](https://juejin.im/post/6844903978438754318)
- [深入理解 JavaScript 之实现继承的 7 种方式](https://juejin.im/post/6844904003277455368)

更多前端内容可以看我 [个人博客](https://github.com/Jacky-Summer/personal-blog)

这次继续一步步回顾 JS 基础知识点，今天讲的是 JS 中的事件循环。

## JavaScript 是单线程的

JS 是一门单线程的非阻塞的脚本语言，这表示在同一时刻最多也只有一个代码段执行。

## 为什么 JavaScript 是单线程的

如果 JS 是多线程的，因为 JS 有 DOM API 可以操作 DOM，如果同时开了两个线程同时操作 DOM 的话，一个线程删除了当前的 DOM 节点，另一个线程要操作当前的 DOM，那么就会有矛盾到底以哪个线程为主。为了避免这种情况出现，JS 就被设计为单线程，而且单线程执行效率高。现在虽然也有 web worker 标准的出现，但它也有很多限制，受主线程控制，是主线程的子线程。

## JS 如何处理异步任务

JS 是单线程，那么非阻塞怎么体现呢？如果 JS 是阻塞的，那么 JS 发起一个异步 IO 请求，在等待结果返回的这个时间段，后面的代码就无法执行了，而 JS 主线程和渲染进程是互斥的，因此可能造成浏览器假死的状态。事实 JS 是非阻塞的，那它要怎么实现异步任务呢，靠的就是事件循环。

## 事件循环

事件循环就是通过异步执行任务的方法来解决单线程的弊端的。

1. 一开始整个脚本作为一个宏任务执行
2. 执行过程中同步代码直接执行，宏任务进入宏任务队列，微任务进入微任务队列
3. 当前宏任务执行完出队，读取微任务列表，有则依次执行，直到全部执行完
4. 执行浏览器 UI 线程的渲染工作
5. 检查是否有 Web Worker 任务，有则执行
6. 执行完本轮的宏任务，回到第 2 步，继续依此循环，直到宏任务和微任务队列都为空

## 宏任务与微任务

JS 引擎把所有任务分成两类，一类叫宏任务(macroTask)，一类叫微任务(microTask)

### 宏任务

- script(整体代码)
- setTimeout/setInterval
- I/O
- UI 渲染
- postMessage
- MessageChannel
- requestAnimationFrame
- setImmediate(Node.js 环境)

### 微任务

- new Promise().then()
- MutaionObserver
- process.nextTick(Node.js 环境）

## 经典题目 1

关于更细节的描述我就不写了，因为有更好的文章可以参考学习：

- [这一次，彻底弄懂 JavaScript 执行机制](https://juejin.im/post/6844903512845860872)

简单介绍后，接下来就来看几道经典题目：

```javascript
console.log('script start')

setTimeout(function () {
  console.log('setTimeout')
}, 0)

Promise.resolve()
  .then(function () {
    console.log('promise1')
  })
  .then(function () {
    console.log('promise2')
  })

console.log('script end')
```

1. 整体 script 作为第一个宏任务进入主线程，输出`script start`
2. 遇到 setTimeout，setTimeout 为宏任务，加入宏任务队列
3. 遇到 Promise，其 then 回调函数加入到微任务队列；第二个 then 回调函数也加入到微任务队列
4. 继续往下执行，输出`script end`
5. 检测微任务队列，输出`promise1`、`promise2`
6. 进入下一轮循环，执行 setTimeout 中的代码，输出`setTimeout`

最后执行结果为：

```
script start
script end
promise1
promise2
setTimeout
```

## 经典题目 2

来看一道面试的经典题目

```javascript
async function async1() {
  console.log('async1 start')
  await async2()
  console.log('async1 end')
}
async function async2() {
  console.log('async2')
}
console.log('script start')
setTimeout(function () {
  console.log('setTimeout')
}, 0)
async1()
new Promise(function (resolve) {
  console.log('promise1')
  resolve()
}).then(function () {
  console.log('promise2')
})
console.log('script end')
```

1. 整体 script 作为第一个宏任务进入主线程，代码自上而下执行，执行同步代码，输出 `script start`
2. 遇到 setTimeout，加入到宏任务队列
3. 执行 async1()，输出`async1 start`；然后遇到`await async2()`,await 实际上是让出线程的标志，首先执行 async2()，输出`async2`；把 async2() 后面的代码`console.log('async1 end')`加入微任务队列中，跳出整个 async 函数。（async 和 await 本身就是 promise+generator 的语法糖。所以 await 后面的代码是微任务。）
4. 继续执行，遇到 new Promise，输出`promise1`，把`.then()`之后的代码加入到微任务队列中
5. 继续往下执行，输出`script end`。接着读取微任务队列，输出`async1 end`，`promise2`，执行完本轮的宏任务。继续执行下一轮宏任务的代码，输出`setTimeout`

最后执行结果为：

```
script start
async1 start
async2
promise1
script end
async1 end
promise2
setTimeout
```

## 经典题目 3

我们来看下面一段代码：

```javascript
setTimeout(function () {
  console.log('timer1')
}, 0)

requestAnimationFrame(function () {
  console.log('requestAnimationFrame')
})

setTimeout(function () {
  console.log('timer2')
}, 0)

new Promise(function executor(resolve) {
  console.log('promise 1')
  resolve()
  console.log('promise 2')
}).then(function () {
  console.log('promise then')
})

console.log('end')
```

- 整体 script 代码执行，开局新增三个宏任务，两个 setTimeout 和一个 requestAnimationFrame
- 遇到 Promise，先输出`promise1`, `promise2`，加把 then 回调加入微任务队列。
- 继续往下执行，输出`end`
- 执行 promise 的 then 回调，输出`promise then`
- 接下来剩三个宏任务，我们可以知道的是`timer1`会比`timer2`先执行，那么`requestAnimationFrame`呢？

当每一轮事件循环的微任务队列被清空后，有可能发生 UI 渲染，也就是说执行任务的耗时会影响视图渲染的时机。

通常浏览器以每秒 60 帧（60fps）的速率刷新页面，这个帧率最适合人眼交互，大概 `1000ms/60` 约等于 16.7ms 渲染一帧，如果要让用户看得顺畅，单个宏任务及它相应的微任务最好能在 16.7ms 内完成。

requestAnimationFrame 是什么？

> window.requestAnimationFrame() 告诉浏览器——你希望执行一个动画，并且要求浏览器在下次重绘之前调用指定的回调函数更新动画。该方法需要传入一个回调函数作为参数，该回调函数会在浏览器下一次重绘之前执行。
> requestAnimationFrame 的基本思想是 让页面重绘的频率和刷新频率保持同步，相比 setTimeout,requestAnimationFrame 最大的优势是由系统来决定回调函数的执行时机。

但这个也不是每轮事件循环都会执行 UI 渲染，不同浏览器有自己的优化策略，比如把几次的视图更新累积到一起重绘，重绘之前会通知 requestAnimationFrame 执行回调函数，也就是说 requestAnimationFrame 回调的执行时机是在一次或多次事件循环的 UI render 阶段。

在我的谷歌浏览器执行结果：

```
promise 1
promise 2
end
promise then
requestAnimationFrame
timer1
timer2
```

在我的火狐浏览器执行结果：

```
promise 1
promise 2
end
promise then
timer1
timer2
requestAnimationFrame
```

谷歌浏览器中的结果 requestAnimationFrame()是在一次事件循环后执行，火狐浏览器中的结果是在三次事件循环结束后执行。

可以知道，浏览器只保证 requestAnimationFrame 的回调在重绘之前执行，但没有确定的时间，何时重绘由浏览器决定。

## 参考文章

- [JavaScript 中的 Event Loop（事件循环）机制](https://segmentfault.com/a/1190000022805523)

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~

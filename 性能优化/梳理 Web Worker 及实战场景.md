# 梳理 Web Worker 及实战场景

## 前言

有一些前端技术点，即使以前用过，但没有自己动手归纳总结过，许久还是要回过头来还是需要重新梳理。于是，本文就来梳理一下 Web Worker。

## 为什么需要 Web Worker

由于 JavaScript 语言采用的是单线程，同一时刻只能做一件事，如果有多个同步计算任务执行，则在这段同步计算逻辑执行完之前，它下方的代码不会执行，从而造成了阻塞，用户的交互也可能无响应。

但如果把这段同步计算逻辑放到 Web Worker 执行，在这段逻辑计算运行期间依然可以执行它下方的代码，用户的操作也可以响应了。

## Web Worker 是什么

HTML5 提供并规范了 Web Worker 这样一套 API，它允许一段 JavaScript 程序运行在主线程之外的另外一个线程（Worker 线程）中。

**Web Worker 的作用，就是为 JavaScript 创造多线程环境，允许主线程创建 Worker 线程，将一些任务分配给后者运行**。这样的好处是，一些计算密集型或高延迟的任务，被 Worker 线程负担了，主线程就会很流畅，不会被阻塞或拖慢。

## Web Worker 的分类

Web Worker 根据工作环境的不同，可分为专用线程 Dedicated Worker 和共享线程 Shared Worker。

Dedicated Worker 的 Worker 只能从创建该 Woker 的脚本中访问，而 SharedWorker 则可以被多个脚本所访问。

在开发中如果使用到 Web Worker，目前大部分主要还是使用 Dedicated Worker 的场景多，它只能为一个页面所使用，本文讲的也是这一类；而 Shared Worker 可以被多个页面共享，为跨浏览器 tab 共享数据提供了一种解决方案。

## Web Worker 的使用限制

### 同源限制

分配给 Worker 线程运行的脚本文件，必须与主线程的脚本文件同源。

### 文件限制

Worker 线程无法读取本地文件（`file://`），会拒绝使用 file 协议来创建 Worker 实例，它所加载的脚本，必须来自网络。

### DOM 操作限制

Worker 线程所在的全局对象，与主线程不一样，区别是：

- 无法读取主线程所在网页的 DOM 对象
- 无法使用`document`、`window`、`parent`这些对象

### 通信限制

Worker 线程和主线程不在同一个上下文环境，它们不能直接通信，必须通过消息完成，交互方法是`postMessage`和`onMessage`，并且在数据传递的时候， Worker 是使用拷贝的方式。

### 脚本限制

Worker 线程不能执行`alert()`方法和`confirm()`方法，但可以使用 XMLHttpRequest 对象发出 AJAX 请求，也可以使用`setTimeout/setInterval`等 API

## 基本 API

```js
const worker = new Worker(aURL, options)
```

- `worker.postMessage`: 向 worker 的内部作用域发送一个消息，消息可由任何  JavaScript 对象组成
- `worker.terminate`: 立即终止 worker。该方法并不会等待 worker 去完成它剩余的操作；worker 将会被立刻停止
- `worker.onmessage`:当 worker 的父级接收到来自其 worker 的消息时，会在 Worker 对象上触发 message 事件
- `worker.onerror`: 当 worker 出现运行中错误时，它的 onerror 事件处理函数会被调用。它会收到一个扩展了 ErrorEvent 接口的名为 error 的事件

```js
worker.addEventListener('error', function (e) {
  console.log(e.message) // 可读性良好的错误消息
  console.log(e.filename) // 发生错误的脚本文件名
  console.log(e.lineno) // 发生错误时所在脚本文件的行号
})
```

## 常见的使用方式

### 1. 直接指定脚本文件

```js
const myWorker = new Worker(aURL, options)
```

`aURL`表示 worker 将执行的脚本的 URL（脚本文件）， 即 Web Worker 所要执行的任务。

案例如下：

```js
// 主线程下创建worker线程
const worker = new Worker('./worker.js')

// 监听接收worker线程发的消息
worker.onmessage = function (e) {
  console.log('主线程收到worker线程消息：', e.data)
}

// 向worker线程发送消息
worker.postMessage('主线程发送hello world')
```

`worker.js`：

```js
// self 代表子线程自身，即子线程的全局对象
self.addEventListener('message', function (e) {
  // e.data表示主线程发送过来的数据
  self.postMessage('worker线程收到的：' + e.data) // 向主线程发送消息
})
```

> Web Worker 的执行上下文名称是 self，无法调用主线程的 window 对象的。上述写法等同于以下写法：

```
this.addEventListener("message", function (e) {
  // e.data表示主线程发送过来的数据
  this.postMessage("worker线程收到的：" + e.data); // 向主线程发送消息
});
```

将 JS 文件引入 html 挂在本地开发环境运行，运行结果如下：

```
主线程收到worker线程消息： worker线程收到的：主线程发送hello world
```

### 2. 使用 Blob URL 创建

除了这种通过引入 js 文件的方式，也可以通过`URL.createObjectURL()`创建 URL 对象，创建内嵌的 worker

```js
/**
 * const blob = new Blob(array, options);
 * Blob() 构造函数返回一个新的 Blob 对象。blob 的内容由参数数组中给出的值的串联组成。
 * @params array 是一个由ArrayBuffer, ArrayBufferView, Blob, DOMString 等对象构成的 Array
 * @options type，默认值为 ""，它代表了将会被放入到 blob 中的数组内容的 MIME 类型。还有两个这里忽略不列举了
 */

/**
 * URL.createObjectURL()：静态方法会创建一个 DOMString，其中包含一个表示参数中给出的对象的 URL。这个 URL 的生命周期和创建它的窗口中的 document 绑定。这个新的 URL 对象表示指定的 File 对象或 Blob 对象
 */
const worker = new Worker(URL.createObjectURL(blob))
```

> - Blob 对象表示一个不可变、原始数据的类文件对象，它的数据可以按文本或二进制的格式进行读取。File 接口基于 Blob，继承了 blob 的功能并将其扩展以支持用户系统上的文件。
> - Blob URL/Object URL 是一种伪协议，允许 Blob 和 File 对象用作图像，下载二进制数据链接等的 URL 源。在浏览器中，我们使用 URL.createObjectURL 方法来创建 Blob URL，该方法接收一个 Blob 对象，并为其创建一个唯一的 URL，其形式为 `blob:<origin>/<uuid>`
> - 浏览器内部为每个通过 URL.createObjectURL 生成的 URL 存储了一个 URL 到 Blob 映射。因此，此类 URL 较短，但可以访问 Blob。生成的 URL 仅在当前文档打开的状态下才有效，它保存在内存中的。它允许引用 `<img>`、`<a>` 中的 Blob，但如果你访问的 Blob URL 不再存在，则会从浏览器中收到 404 错误

```js
function func() {
  console.log('hello')
}

function createWorker(fn) {
  // const blob = new Blob([fn.toString() + ' fn()'], { type: 'text/javascript' })
  const blob = new Blob([`(${fn.toString()})()`], { type: 'text/javascript' })
  return URL.createObjectURL(blob)
}

createWorker(func)
```

## Worker 线程中引入脚本

Worker 线程内部要加载其他脚本，可以使用 `importScripts()`

```js
// worker.js
importScripts('constants.js')

// self 代表子线程自身，即子线程的全局对象
self.addEventListener('message', function (e) {
  self.postMessage(foo) // 可拿到 `foo`、`getAge()`、`getName`的结果值
})

// constants.js
const foo = '变量'

function getAge() {
  return 25
}

const getName = () => {
  return 'jacky'
}
```

还可以同时加载多个脚本

```js
importScripts('script1.js', 'script2.js')
```

## 实战应用场景

### 处理大量 CPU 耗时计算操作

大家最关心的还是 Web Worker 实战场景，开头我们说到，当有大量复杂计算场景时，可使用 Web Worker

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>worker计算</title>
  </head>
  <body>
    <div>计算从 1 到给定数值的总和</div>
    <input type="text" placeholder="请输入数字" id="num" />
    <button onclick="calc()">开始计算</button>
    <span>计算结果为：<span id="result">-</span></span>

    <div>在计算期间你可以填XX表单</div>
    <input type="text" placeholder="请输入姓名" />
    <input type="text" placeholder="请输入年龄" />

    <script>
      function calc() {
        const num = parseInt(document.getElementById('num').value)
        let result = 0
        let startTime = performance.now()
        // 计算求和（模拟复杂计算）
        for (let i = 0; i <= num; i++) {
          result += i
        }
        // 由于是同步计算，在没计算完成之前下面的代码都无法执行
        const time = performance.now() - startTime
        console.log('总计算花费时间:', time)
        document.getElementById('result').innerHTML = result
      }
    </script>
  </body>
</html>
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/853563262feb42ec81fd8cd364896287~tplv-k3u1fbpfcp-watermark.image?)

如上，第一个输入框与按钮是负责模拟复杂计算的，比如输入 10000000000，点击开始计算，这时主线程处理一直在处理同步计算逻辑，在完成计算之前，会发现页面处于卡顿的状态，下方的两个输入框也无法点击交互，在我的电脑这部分计算是花了 14s 左右，这个卡顿时间给用户的体验就很差了。

打开控制台调用也可以看到这里 CPU 使用率是 100%

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9a4c201e32914052b116e888be648147~tplv-k3u1fbpfcp-watermark.image?)

如果把这部分计算交给 Web Worker 来处理，修改代码：

```html
<script>
  const worker = new Worker('./worker.js')

  function calc() {
    const num = parseInt(document.getElementById('num').value)
    worker.postMessage(num)
  }

  worker.onmessage = function (e) {
    document.getElementById('result').innerHTML = e.data
  }
</script>
```

`./worker.js`

```js
function calc(num) {
  let result = 0
  let startTime = performance.now()
  // 计算求和（模拟复杂计算）
  for (let i = 0; i <= num; i++) {
    result += i
  }
  // 由于是同步计算，在没计算完成之前下面的代码都无法执行
  const time = performance.now() - startTime
  console.log('总计算花费时间:', time)
  self.postMessage(result)
}

self.onmessage = function (e) {
  calc(e.data)
}
```

然后重复上述一样的操作，输入 10000000000 计算，会发现下方两个输入框可正常流畅输入，整个页面也不卡顿。

Worker 运行独立于主线程的后台线程中，分担执行了大量占用 CPU 密集型的操作（但运行时间并不会变短），解放了主线程，主线程就能及时响应用户操作而不会造成卡顿的现象。使用 Web Worker 后，控制台工具可看到 CPU 使用率处于较低正常水平，计算过程跟没计算之前的水平一样。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02132233e62741a1a089e9aa9216bd27~tplv-k3u1fbpfcp-watermark.image?)

### 音视频 canvas 绘制录屏

这个是我工作中遇到的场景，通过 绘制 WebRTC 视频流录制视频，最后生成视频。

以前写过一篇文章[如何实现前端录屏](https://juejin.cn/post/6990911521446445063)，这篇基本就没怎么认真写，就是纯属记录，而且现在的代码跟这篇文章的示例代码差别很大（就是说示例代码改进空间很大），建议总体看下思路，实现具体细节就不细究了，毕竟思路相通。

就以上面这篇文章的代码作为示例（现最新代码跟公司代码业务结合就不放了），有个比较耗 CPU 的操作

```js
// 16ms一次的定时器
refreshTimer.current = window.setInterval(onRefreshTimer, 16)

// onRefreshTimer 函数里面做的实际就是高频执行 recorderDrawFrame() 方法
// 录屏绘制操作
const recorderDrawFrame = () => {
    const $recorderCanvas = recorderCanvas.current!
    const $player = videoRef.current!
    const ctx = recorderContext.current!
    const { width, height } = getResolution()
    $recorderCanvas.width = width
    $recorderCanvas.height = height

    // 其中这个绘制函数对CPU占用率会比较高（在低配置的电脑浏览器上）
    ctx.drawImage(
      $player,
      0,
      0,
      $player.videoWidth,
      $player.videoHeight,
      0,
      0,
      $recorderCanvas.width,
      $recorderCanvas.height,
    )
    drawWatermark(ctx, width)
}
```

那么怎么做优化呢？就是把整个 `onRefreshTimer` 这个定时器函数交给 Web Worker 执行。

上面说到，Web Worker 虽有 DOM 操作的限制，但可以使用 `setTimeout/setInterval`等 API，所以具体实现就是把 Worker 封装为类，在类内部处理好逻辑，然后暴露 setInterval 等方法给外部实例调用。

```js
if (worker) {
  refreshTimer.current = worker.setInterval(onRefreshTimer, 16)
}
```

### 其他场景

在网上看过的文章，其他场景比如有：

- 前端导出生成 excel
- 图片批量压缩等

可参考阅读：[我不允许你们学会了 worker 却还没有应用场景](https://juejin.cn/post/7148239142806093838)

## 结语

Web Worker 日常业务开发整体用到的次数估计不多，但如果有遇到上述提到的那些类似的场景，我们优化的思考方向又可以多一个选择了。

## 参考

- [Web Worker 使用教程](http://www.ruanyifeng.com/blog/2018/07/web-worker.html)
- [Web Worker 在项目中的妙用](https://juejin.cn/post/6844903510673211400)
- [浅析 Web Workers 及应用](https://segmentfault.com/a/1190000039861897)

---

**_本文正在参加[「金石计划 . 瓜分 6 万现金大奖」](https://juejin.cn/post/7162096952883019783 'https://juejin.cn/post/7162096952883019783')_**

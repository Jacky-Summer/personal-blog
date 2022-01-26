# Web Worker 学习

## 前言

JavaScript 语言采用的是单线程，所有任务只能在一个线程上完成，一次只能做一件事。比如代码运行到一段计算复杂量大的逻辑，在这段计算逻辑执行完之前，它下方的代码不会执行。但如果把这段计算逻辑放到 Web Worker 执行，则两者互不干扰，在这段逻辑计算运行期间则依然还可以执行下方的代码，对应的是可以响应用户的其它操作。

## Web Worker 是什么

HTML5 提供并规范了 Web Worker 这样一套 API，它允许一段 JavaScript 程序运行在主线程之外的另外一个线程（Worker 线程）中。它的作用是为 JavaScript 创造多线程环境，允许主线程创建 Worker 线程，将一些任务分配给后者运行，主线程与 worker 线程的运行互不干扰，这样的好处是主线程不会被阻塞或拖慢。

## Web Worker 的限制

### 同源限制

分配给 Worker 线程运行的脚本文件，必须与主线程的脚本文件同源。

### DOM 限制

Worker 线程所在的全局对象，与主线程不一样，无法读取主线程所在网页的 DOM 对象，也无法使用`document`、`window`、`parent`这些对象。但是，Worker 线程可以`navigator`对象和`location`对象。

### 通信联系

Worker 线程和主线程不在同一个上下文环境，它们不能直接通信，必须通过消息完成。

### 脚本限制

Worker 线程不能执行`alert()`方法和`confirm()`方法，但可以使用 XMLHttpRequest 对象发出 AJAX 请求。

### 文件限制

Worker 线程无法读取本地文件，即不能打开本机的文件系统（`file://`），它所加载的脚本，必须来自网络。

## API

- `worker.postMessage`: 发送一条消息到最近的外层对象，消息可由任何  JavaScript 对象组成
- `worker.terminate`: 立即终止 worker。该方法不会给 worker 留下任何完成操作的机会；就是简单的立即停止
- `worker.onmessage`: 当 MessageEvent 类型的事件冒泡到 worker  时，事件监听函数  EventListener 被调用
- `worker.onerror`: 当 ErrorEvent 类型的事件冒泡到 worker 时，事件监听函数 EventListener 被调用

## 基本使用

```js
const myWorker = new Worker(aURL, options)
```

`aURL`表示 worker 执行的脚本的 URL（脚本文件），该文件就是 Worker 线程所要执行的任务。

比如，主线程的 js 代码如下：

```js
const worker = new Worker('worker.js')

// 主线程监听函数，接收子线程发回来的消息
worker.onmessage = function (e) {
  console.log('主线程收到worker线程消息：', e.data)
}

// 主线程调用`worker.postMessage()`方法，向 Worker 发消息
worker.postMessage('主线程发送world')
```

`worker.js`代码如下：

```js
// `self`代表子线程自身，即子线程的全局对象
self.addEventListener('message', function (e) {
  // `e.data`属性包含主线程发来的数据
  console.log('worker线程收到主线程消息：', e.data)
})

self.postMessage('worker线程发送hello')
```

新建 html 文件，引入主线程 js 代码，运行结果如下：

```
主线程收到worker线程消息： worker线程发送hello
worker线程收到主线程消息： 主线程发送world
```

除了这种通过引入 js 文件的方式，也可以通过 URL.createObjectURL()创建 URL 对象，创建内嵌的 worker

```js
const calc = () => {
  let result = 0
  for (let i = 0; i < 100000; i++) {
    // 复杂计算...
    result += i
  }
  console.log('result', result)
}

const createBlobURL = (func) => {
  const blob = new Blob([`(${func.toString()})()`])
  return URL.createObjectURL(blob)
}

const myWorker = new Worker(createBlobURL(calc))
```

### 终止 worker

Worker 接口中的`terminate()`方法用于立即终止 Worker 的行为. 本方法并不会等待 worker 去完成它剩余的操作；worker 将会被立刻停止

```js
// 主页面调用
const myWorker = new Worker('worker.js')
myWorker.terminate()

// Worker 线程调用
self.close()
```

参考：

- [Web Worker 使用教程](http://www.ruanyifeng.com/blog/2018/07/web-worker.html)

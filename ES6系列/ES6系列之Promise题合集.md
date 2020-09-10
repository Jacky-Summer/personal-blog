# ES6 系列之 Promise 题合集

## 前言

关于 Promise 的讲解文章实在太多了，在此我就不写讲解了，直接实战检测自己对 Promise 的理解，下面是我做过的众多 Promise 题里面挑出来的 13 道题，我觉得容易错或者值得考究知识点的题，如果你想考察自己对 Promise 的掌握程度，可以做做看。

这些题是我从下面两个链接的题目又选出来的，仅作为自己的错题集/笔记题来记录，方便以后我自己检验回顾。如果你想更全面学，建议直接去做下面两篇的题。

## 特别鸣谢

- [【建议星星】要就来 45 道 Promise 面试题一次爽到底(1.1w 字用心整理)](https://juejin.im/post/6844904077537574919)
- [八段代码彻底掌握 Promise](https://juejin.im/post/6844903488695042062)

### 题目 1

```javascript
const promise1 = new Promise((resolve, reject) => {
  console.log('promise1')
  resolve('resolve1')
})
const promise2 = promise1.then(res => {
  console.log(res)
})
console.log('1', promise1)
console.log('2', promise2)
```

分析：

- 先遇到`new Promise`，执行该构造函数中的代码输出 `promise1`
- 遇到`resolve`函数, 将`promise1`的状态改变为`fulfilled`, 并将结果保存下来
- 碰到`promise1.then`这个微任务，返回的是`Promise`的`pending`状态，将它放入微任务队列
- 故`promise2`是一个新的状态为`pending`的`Promise`
- 继续执行同步代码输出`promise1`的状态是`fulfilled`，输出`promise2`的状态是`pending`
- 宏任务执行完毕，查找微任务队列，发现`promise1.then`这个微任务且状态为`fulfilled`，执行它。

输出结果：

```
promise1
1 Promise{<fulfilled>: 'resolve1'}
2 Promise{<pending>}
resolve1
```

### 题目 2

```javascript
const promise = new Promise((resolve, reject) => {
  console.log(1)
  setTimeout(() => {
    console.log('timerStart')
    resolve('success')
    console.log('timerEnd')
  }, 0)
  console.log(2)
})
promise.then(res => {
  console.log(res)
})
console.log(4)
```

分析：

- 执行`new Promsise`，输出`1`，`setTimeout`宏任务加到宏任务队列，继续执行同步代码输出`2`
- 遇到`promise.then`，但其状态还是 pending，这里理解为先不执行；然后输出同步代码`4`
- 一轮循环过后，进入第二次宏任务，发现延迟队列中有`setTimeout`定时器，执行它
- 输出`timerStart`，遇到`resolve`，将`promise`的状态改为`fulfilled`且保存结果并将之前的`promise.then`推入微任务队列
- 继续执行同步代码`timerEnd`
- 宏任务全部执行完毕，查找微任务队列，发现`promise.then`这个微任务，执行它。

输出结果：

```
1
2
4
timerStart
timerEnd
success
```

### 题目 3

```javascript
const promise1 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('success')
  }, 1000)
})
const promise2 = promise1.then(() => {
  throw new Error('error!!!')
})
console.log('promise1', promise1)
console.log('promise2', promise2)
setTimeout(() => {
  console.log('promise1', promise1)
  console.log('promise2', promise2)
}, 2000)
```

分析：

- 从上至下，先执行第一个`new Promise`中的函数，碰到`setTimeout`将它加入下一个宏任务列表
- 跳出`new Promise`，碰到`promise1.then`这个微任务，但其状态还是为`pending`，这里理解为先不执行；故`promise2`是一个新的状态为`pending`的`Promise`
- 执行同步代码`console.log('promise1')`，输出`promise1`的状态为`pending`
- 执行同步代`console.log('promise2')`，输出`promise2`的状态为`pending`
- 碰到第二个`setTimeout`，将其放入下一个宏任务列表
- 第一轮宏任务执行结束，并且没有微任务需要执行，因此执行第二轮宏任务
- 先执行第一个定时器里的内容，将`promise1`的状态改为`fulfilled`且保存结果并将之前的`promise1.then`推入微任务队列
- 该定时器中没有其它的同步代码可执行，因此执行本轮的微任务队列，也就是`promise1.then`，它抛出了一个错误，且将`promise2`的状态设置为了`rejected`
- 第一个定时器执行完毕，开始执行第二个定时器中的内容
- 打印出`promise1`，且此时`promise1`的状态为`fulfilled`
- 打印出`promise2`，且此时`promise2`的状态为`rejected`

```javascript
promise1 Promise {<pending>}
promise2 Promise {<pending>}
Uncaught (in promise) Error: error!!! at <anonymous>:7:9
promise1 Promise {<fulfilled>: "success"}
promise2 Promise {<rejected>: Error: error!!! at <anonymous>:7:9}
```

### 题目 4

```javascript
const promise = new Promise((resolve, reject) => {
  reject('error')
  resolve('success2')
})
promise
  .then(res => {
    console.log('then1: ', res)
  })
  .then(res => {
    console.log('then2: ', res)
  })
  .catch(err => {
    console.log('catch: ', err)
  })
  .then(res => {
    console.log('then3: ', res)
  })
```

分析：

- `catch`不管被连接到哪里，都能捕获上层未捕捉过的错误。
- 由于 `catch()`也会返回一个`Promise`，且由于这个`Promise`没有返回值，所以打印出来的是`undefined`。

输出结果：

```
catch:  error
then3:  undefined
```

### 题目 5

```javascript
Promise.resolve()
  .then(() => {
    return new Error('error!!!')
  })
  .then(res => {
    console.log('then: ', res)
  })
  .catch(err => {
    console.log('catch: ', err)
  })
```

分析：

- 在`Promise`中，返回任意一个非`promise`的值都会被包裹成`promise`对象，例如`return new Error('error!!!')`会被包装为`return Promise.resolve(new Error('error!!!'))`
- `.then`或者 `.catch` 中 `return` 一个 `error` 对象并不会抛出错误，所以不会被后续的 `.catch` 捕获。

当然如果你抛出一个错误的话，可以用下面任意一种：

```javascript
return Promise.reject(new Error('error!!!'))
// or
throw new Error('error!!!')
```

输出结果：

```
then:  Error: error!!!
```

### 题目 6

```javascript
const promise = Promise.resolve().then(() => {
  return promise
})
promise.catch(console.err)
```

分析：

- `.then` 或 `.catch` 返回的值不能是 `promise` 本身，否则会造成死循环。

输出结果：

```
Promise {<rejected>: TypeError: Chaining cycle detected for promise #<Promise>}
```

### 题目 7

```javascript
Promise.resolve(1).then(2).then(Promise.resolve(3)).then(console.log)
```

分析：

- `.then` 或者 `.catch` 的参数期望是函数，传入非函数则会发生值透传。
- 第一个`then`和第二个`then`中传入的都不是函数，一个是数字类型，一个是对象类型，因此发生了透传，将`resolve(1)` 的值直接传到最后一个`then`里。

输出结果：

```
1
```

### 题目 8

```javascript
Promise.reject('err!!!')
  .then(
    res => {
      console.log('success', res)
    },
    err => {
      console.log('error', err)
    }
  )
  .catch(err => {
    console.log('catch', err)
  })
```

分析：

- `.then`函数的第一个参数是用来处理`Promise`成功的函数，第二个则是处理失败的函数。
- `Promise.resolve('err!!!')`的值会进入成功的函数，`Promise.reject('err!!!')`的值会进入失败的函数。
- 如果去掉第二个参数，就会进入`catch()`中

输出结果：

```
'error' 'error!!!'
```

### 题目 9

```javascript
Promise.resolve('1')
  .then(res => {
    console.log(res)
  })
  .finally(() => {
    console.log('finally')
  })
Promise.resolve('2')
  .finally(() => {
    console.log('finally2')
    return '我是finally2返回的值'
  })
  .then(res => {
    console.log('finally2后面的then函数', res)
  })
```

分析：

- `.finally()`方法不管`Promise`对象最后的状态如何都会执行
- 它最终返回的默认会是一个上一次的`Promise`对象值，不过如果抛出的是一个异常则返回异常的`Promise`对象。
- 第一行代码遇到`Promise.resolve('1')`,再把`.then()`加入微任务，这时候要注意，代码并不会接着往链式调用的下面走，也就是不会先将`.finally`加入微任务列表，那是因为`.then`本身就是一个微任务，它链式后面的内容必须得等当前这个微任务执行完才会执行，因此这里我们先不管`.finally()`
- 执行`Promise.resolve('2')`，把`.finally`加入微任务队列，且链式调用后面的内容得等该任务执行完后才执行
- 本轮宏任务执行完，执行微任务列表第一个微任务输出`1`，遇到`.finally()`，将它加入微任务列表（第三个）待执行；再执行第二个微任务输出`finally2`，遇到`.then`，把它加入到微任务（第四个）列表待执行
- 本轮执行完两个微任务后，检索微任务列表发现还有两个微任务，故执行，输出`finally`和`finally2后面的then函数 2`

输出结果：

```
1
finally2
finally
finally2后面的then函数 2
```

### 题目 10

```javascript
async function async1() {
  console.log('async1 start')
  await new Promise(resolve => {
    console.log('promise1')
  })
  console.log('async1 success')
  return 'async1 end'
}
console.log('srcipt start')
async1().then(res => console.log(res))
console.log('srcipt end')
```

分析：

- 在`async1`中`await`后面的`Promise`是没有返回值的，也就是它的初始状态是`pending`状态，因此相当于一直在`await`
- 所以在`await`之后的内容是不会执行的，也包括`async1`后面的`.then`。

输出结果：

```
srcipt start
async1 start
promise1
srcipt end
```

### 题目 11

```javascript
async function async1() {
  await async2()
  console.log('async1')
  return 'async1 success'
}
async function async2() {
  return new Promise((resolve, reject) => {
    console.log('async2')
    reject('error')
  })
}
async1().then(res => console.log(res))
```

分析：

- `await`后面跟着的是一个状态为`rejected`的`promise`。
- 如果在`async`函数中抛出了错误，则终止错误结果，不会继续向下执行。

输出结果：

```
async2
Uncaught (in promise) error
```

### 题目 12

```javascript
var p1 = new Promise(function (resolve, reject) {
  foo.bar()
  resolve(1)
})

p1.then(
  function (value) {
    console.log('p1 then value: ' + value)
  },
  function (err) {
    console.log('p1 then err: ' + err)
  }
).then(
  function (value) {
    console.log('p1 then then value: ' + value)
  },
  function (err) {
    console.log('p1 then then err: ' + err)
  }
)

var p2 = new Promise(function (resolve, reject) {
  resolve(2)
})

p2.then(
  function (value) {
    console.log('p2 then value: ' + value)
    foo.bar()
  },
  function (err) {
    console.log('p2 then err: ' + err)
  }
)
  .then(
    function (value) {
      console.log('p2 then then value: ' + value)
    },
    function (err) {
      console.log('p2 then then err: ' + err)
      return 1
    }
  )
  .then(
    function (value) {
      console.log('p2 then then then value: ' + value)
    },
    function (err) {
      console.log('p2 then then then err: ' + err)
    }
  )
```

分析：

Promise 中的异常由 then 参数中第二个回调函数（Promise 执行失败的回调）处理，异常信息将作为 Promise 的值。异常一旦得到处理，then 返回的后续 Promise 对象将恢复正常，并会被 Promise 执行成功的回调函数处理。另外，需要注意 p1、p2 多级 then 的回调函数是交替执行的 ，这正是由 Promise then 回调的异步性决定的。

输出结果：

```
p1 then err: ReferenceError: foo is not defined
p2 then value: 2
p1 then then value: undefined
p2 then then err: ReferenceError: foo is not defined
p2 then then then value: 1
```

### 题目 13

```javascript
var p1 = new Promise(function (resolve, reject) {
  resolve(Promise.resolve('resolve'))
})

var p2 = new Promise(function (resolve, reject) {
  resolve(Promise.reject('reject'))
})

var p3 = new Promise(function (resolve, reject) {
  reject(Promise.resolve('resolve'))
})

p1.then(
  function fulfilled(value) {
    console.log('fulfilled: ' + value)
  },
  function rejected(err) {
    console.log('rejected: ' + err)
  }
)

p2.then(
  function fulfilled(value) {
    console.log('fulfilled: ' + value)
  },
  function rejected(err) {
    console.log('rejected: ' + err)
  }
)

p3.then(
  function fulfilled(value) {
    console.log('fulfilled: ' + value)
  },
  function rejected(err) {
    console.log('rejected: ' + err)
  }
)
```

分析：

Promise 回调函数中的第一个参数 resolve，会对 Promise 执行"拆箱"动作。即当 resolve 的参数是一个 Promise 对象时，resolve 会"拆箱"获取这个 Promise 对象的状态和值，但这个过程是异步的。p1"拆箱"后，获取到 Promise 对象的状态是 resolved，因此 fulfilled 回调被执行；p2"拆箱"后，获取到 Promise 对象的状态是 rejected，因此 rejected 回调被执行。但 Promise 回调函数中的第二个参数 reject 不具备”拆箱“的能力，reject 的参数会直接传递给 then 方法中的 rejected 回调。因此，即使 p3 reject 接收了一个 resolved 状态的 Promise，then 方法中被调用的依然是 rejected，并且参数就是 reject 接收到的 Promise 对象。

输出结果：

```
p3 rejected: [object Promise]
p1 fulfilled: resolve
p2 rejected: reject
```

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~

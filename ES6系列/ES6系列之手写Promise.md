# ES6 系列之手写 Promise

## 前言

Promise 在开发中我们经常用到，它解决了回调地狱问题，对错误的处理也非常方便。本文我将通过一步步完善 Promise，从简单到复杂的过程来说。

本文适合熟练运用 Promise 的人阅读。

## 极简版

首先`Promise`是一个类，类中的构造函数需要接收一个执行函数`executor`默认就会执行，它有两个参数：`resolve`和`reject`，这两个参数是`Promise`内部定义的两个函数，用来改变状态并执行对应回调函数。

默认创建一个`Promise`状态就是`pending`，`promise`只有三种状态：`pending`,`fulfilled`,`rejected`，调用成功`resolve`和失败`reject`方法时，需要传递一个成功的原因/值`value`和失败的原因`reason`。每一个`promise`实例都有一个 then 方法。

`Promise`状态一经改变就不能再改变，故我们限制只能在状态为`pending`才改变，这样就保证状态只能改变一次。

如果抛出异常按照失败来处理。

按照以上`Promise`的基本要求，就有个基本结构：

```
const STATUS = {
  PENDING: 'PENDING',
  FULFILLED: 'FULFILLED',
  REJECTED: 'REJECTED',
}

class Promise {
  constructor(executor) {
    this.status = STATUS.PENDING
    this.value = undefined // 成功的值
    this.reason = undefined // 失败原因
    this.onResolvedCallbacks = [] // 存放成功的回调
    this.onRejectedCallbacks = [] // 存放失败的回调

    const resolve = val => {
      if (this.status === STATUS.PENDING) {
        this.status = STATUS.FULFILLED
        this.value = val
        this.onResolvedCallbacks.forEach(fn => fn())
      }
    }

    const reject = reason => {
      if (this.status === STATUS.PENDING) {
        this.status = STATUS.REJECTED
        this.reason = reason
        this.onRejectedCallbacks.forEach(fn => fn())
      }
    }

    try {
      executor(resolve, reject)
    } catch (e) {
      console.log(e)
    }
  }

  then(onFulfilled, onRejected) {
    if (this.status === STATUS.FULFILLED) {
      onFulfilled(this.value)
    }
    if (this.status === STATUS.REJECTED) {
      onRejected(this.reason)
    }
    if (this.status === STATUS.PENDING) {
      this.onResolvedCallbacks.push(() => {
        onFulfilled(this.value)
      })
      this.onRejectedCallbacks.push(() => {
        onRejected(this.reason)
      })
    }
  }
}
```

上面代码中，为什么不直接执行回调还要存储呢？因为如果当`resolve(3)`被延迟执行时，此时常理写代码来说 then 是会被执行的，但此时没有`resolve`，故`p`的状态应为`pending`，不应立即执行成功调用的函数，需要把它存起来，直到执行`resolve`再执行成功调用的函数。

```javascript
let p = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve(3)
  }, 1000)
})
p.then(res => {
  console.log('then1', res)
  return 2
})
```

## 链式调用

链式调用想必大家多少有点了解，在 jQuery 里面的链式调用则是返回 this，而 Promise 里面的链接调用则是返回一个新的 Promise 对象。

```javascript
let p = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve(3)
  }, 1000)
})
p.then(res => {
  console.log('then1', res) // then1 3
  return 2
}).then(res => {
  // 链式调用
  console.log('then2', res) // then2 2
})
```

修改`then`方法和增加`catch`方法

```javascript
then(onFulfilled, onRejected) {
  let nextPromise = new Promise((resolve, reject) => {
    if (this.status === STATUS.FULFILLED) {
      setTimeout(() => {
        try {
          let res = onFulfilled(this.value)
          resolve(res)
        } catch (e) {
          reject(e)
        }
      })
    }
    if (this.status === STATUS.REJECTED) {
      setTimeout(() => {
        try {
          let res = onRejected(this.reason)
          resolve(res)
        } catch (e) {
          reject(e)
        }
      })
    }
    if (this.status === STATUS.PENDING) {
      this.onResolvedCallbacks.push(() => {
        setTimeout(() => {
          try {
            let res = onFulfilled(this.value)
            resolve(res)
          } catch (e) {
            reject(e)
          }
        })
      })
      this.onRejectedCallbacks.push(() => {
        setTimeout(() => {
          try {
            let res = onRejected(this.reason)
            resolve(res)
          } catch (e) {
            reject(e)
          }
        })
      })
    }
  })
  return nextPromise
}

catch(err) {
  // 默认没有成功，只有失败
  return this.then(undefined, err)
}
```

此次修改我们是在最外层包了新的`Promise`，然后加了个 setTimeout 模拟微任务（因为这里用的 setTimeout 模拟微任务，所以 JS 事件循环执行顺序上和原生 Promise 有区别），把回调放入，等待确保异步执行。

## 链式调用进阶版

上面的`Promise`依旧没有过关，因为如果链式调用中`Promise`返回的是普通值，就应该把值包装成新的`Promise`对象

- 每个 then 方法都返回一个新的 Promise 对象（重点）
- 如果 then 方法返回了一个 Promise 对象，则需要查看它的状态，如果状态是成功，则调用`resolve`方法，把成功的状态传递给它；如果是失败的，则把失败的状态传递给下一个`Promise`对象。
- 如果 then 方法中返回的是一个原始数据类型值（如 Number、String 等）就使用此值包装成一个新的 Promise 对象返回。
- 如果 then 方法中没有 return 语句，则返回一个用 undefined 包装的 Promise 对象
- 如果 then 方法没有传入任何回调，则继续向下传递（值的传递特性）。
- 如果是循环引用则需要抛出错误

修改`then`方法

```javascript
then(onFulfilled, onRejected) {
  onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : x => x
  onRejected =
    typeof onRejected === 'function'
      ? onRejected
      : err => {
          throw err
        }
  let nextPromise = new Promise((resolve, reject) => {
    if (this.status === STATUS.FULFILLED) {
      setTimeout(() => {
        try {
          let res = onFulfilled(this.value)
          resolvePromise(res, nextPromise, resolve, reject)
        } catch (e) {
          reject(e)
        }
      })
    }

    if (this.status === STATUS.REJECTED) {
      setTimeout(() => {
        try {
          let res = onRejected(this.reason)
          resolvePromise(res, nextPromise, resolve, reject)
        } catch (e) {
          reject(e)
        }
      })
    }
    if (this.status === STATUS.PENDING) {
      this.onResolvedCallbacks.push(() => {
        setTimeout(() => {
          try {
            let res = onFulfilled(this.value)
            resolvePromise(res, nextPromise, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })
      })
      this.onRejectedCallbacks.push(() => {
        setTimeout(() => {
          try {
            let res = onRejected(this.reason)
            resolve(res)
          } catch (e) {
            reject(e)
          }
        })
      })
    }
  })
  return nextPromise
}
```

里面增加了一个`resolvePromise`函数，处理调用`then`方法后不同返回值的情况 ，实现如下：

```javascript
function resolvePromise(x, nextPromise, resolve, reject) {
  if (x === nextPromise) {
    // x 和 nextPromise 指向同一对象，循环引用抛出错误
    return reject(new TypeError('循环引用'))
  } else if (x && (typeof x === 'object' || typeof x === 'function')) {
    // x 是对象或者函数

    let called = false // 避免多次调用
    try {
      let then = x.then // 判断对象是否有 then 方法
      if (typeof then === 'function') {
        // then 是函数，就断定 x 是一个 Promise（根据Promise A+规范）
        then.call(
          x,
          function (y) {
            // 调用返回的promise，用它的结果作为下一次then的结果
            if (called) return
            called = true
            resolvePromise(y, nextPromise, resolve, reject) // 递归解析成功后的值，直到它是一个普通值为止
          },
          function (r) {
            if (called) return
            called = true
            reject(r) // 取then时发生错误了
          }
        )
      } else {
        resolve(x) // 此时 x 就是一个普通对象
      }
    } catch (e) {
      reject(e)
    }
  } else {
    // x 是原始数据类型 / 没有返回值，这里即是undefined
    resolve(x)
  }
}
```

## Promise.resolve() 实现

```javascript
class Promise {
  // ...
  static resolve(val) {
    return new Promise((resolve, reject) => {
      resolve(val)
    })
  }
}
```

## Promise.reject() 实现

```javascript
class Promise {
  // ...
  static reject(reason) {
    return new Promise((resolve, reject) => {
      reject(reason)
    })
  }
}
```

## Promise.prototype.finally 实现

该方法主要有两大重点

- 无论当前这个 Promise 对象最终的状态是成功还是失败 ,finally 方法里的回调函数都会执行一次
- 在 finally 方法后面可以继续链式调用 then 方法，拿到当前这个 Promise 对象最终返回的结果

```javascript
Promise.prototype.finally = function (callback) {
  return this.then(
    data => {
      // 让函数执行，内部会调用方法，如果方法是promise需要等待它完成
      return Promise.resolve(callback()).then(() => data)
    },
    err => {
      return Promise.resolve(callback()).then(() => {
        throw err
      })
    }
  )
}
```

## Promise.all() 实现

Promise.all 可以将多个 Promise 实例包装成一个新的 Promise 实例。同时，成功和失败的返回值是不同的，成功的时候返回的是一个结果数组，而失败的时候则返回最先被 reject 失败状态的值。

```javascript
Promise.all = function (promises) {
  if (!Array.isArray(promises)) {
    throw new Error('Not a array')
  }
  return new Promise((resolve, reject) => {
    let result = []
    let times = 0 // 计数器
    function processData(index, val) {
      result[index] = val
      if (++times === promises.length) {
        resolve(result)
      }
    }
    for (let i = 0; i < promises.length; i++) {
      let p = promises[i]
      if (isPromise(p)) {
        // Promise对象
        p.then(data => {
          processData(i, data)
        }, reject)
      } else {
        processData(i, p) // 普通值
      }
    }
  })
}
```

## Promise.race() 实现

在执行多个异步操作中，只保留取第一个执行完成的异步操作的结果，即是哪个结果获得的快，就返回那个结果，不管结果本身是成功状态还是失败状态。其他的方法仍在执行，不过执行结果会被抛弃。

```javascript
Promise.race = function (promises) {
  if (!Array.isArray(promises)) {
    throw new Error('Not a array')
  }
  return new Promise((resolve, reject) => {
    if (promises.length === 0) {
      return
    } else {
      for (let p of promises) {
        resolve(p).then(
          value => {
            resolve(value)
          },
          reason => {
            reject(reason)
          }
        )
      }
    }
  })
}
```

到此为止，Promise 就实现完成了。

[完整源码地址](https://github.com/Jacky-Summer/handwriting-source-code/blob/master/src/Promise/mini-promise.js)

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，鼓励我继续写作吧~

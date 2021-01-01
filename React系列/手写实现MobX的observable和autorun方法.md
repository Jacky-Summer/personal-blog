# 手写实现 MobX 的 observable 和 autorun 方法

## 前言

学了 Mobx，了解了简单原理，故简单粗略的方式实现下

Demo

```javascript
import { observable, autorun } from './mobx'

let obj = observable({
  name: 'jacky',
  age: 22,
})
// autorun方法这个回调函数会在初始化的时候被执行一次，之后每次内部相关的observable中的依赖发生变动时被再次调用
autorun(() => {
  console.log('autorun', obj.name, obj.age)
})

obj.name = 'xxx'
```

实现 observable 方法

`./mobx/observable.js`

```javascript
import Reaction from './reaction'

// 深层Proxy代理返回
function deepProxy(val, handler) {
  if (typeof val !== 'object') return val
  for (let key in val) {
    // 从后往前依次实现代理的功能，相当于是后序遍历
    val[key] = deepProxy(val[key], handler)
  }

  return new Proxy(val, handler())
}

function createObservable(val) {
  // 声明一个专门用来代理的对象
  let handler = () => {
    let reaction = new Reaction()

    return {
      get(target, key) {
        reaction.collect()
        return Reflect.get(target, key)
      },
      set(target, key, value) {
        // 对于数组的值设置处理: 当对数组进行观察监听时，由于对数组的操作会有两步执行:
        // 更新数组元素值
        // 更改数组的length属性，所以需要将更改length属性的操作给拦截，避免一次操作数组，多次触发handler
        if (key === 'length') return true
        let r = Reflect.set(target, key, value)
        reaction.run()
        return r
      },
    }
  }
  return deepProxy(val, handler)
}

function observable(target, key, descriptor) {
  if (typeof key === 'string') {
    // 装饰器写法：先把装饰的对象进行深度代理
    let v = descriptor.initializer()
    v = createObservable(v)
    let reaction = new Reaction()
    // 返回描述器
    return {
      enumerable: true,
      configurable: true,
      get() {
        reaction.collect()
        return v
      },
      set(value) {
        v = value
        reaction.run()
      },
    }
  }
  return createObservable(target) // 不是装饰器写法：将目标对象进行代理操作，创建成可操作对象
}

export default observable
```

autorun 方法

`./mobx/autorun.js`

```javascript
import Reaction from './reaction'

function autorun(handler) {
  Reaction.start(handler) // 先保存函数，搜集依赖，设置Reaction中的nowFn
  handler() // 调用该方法会触发 get 属性
  Reaction.end() // 清除nowFn
}

export default autorun
```

Reaction 类

`./mobx/Reaction.js`

```javascript
let nowFn = null // 表示当前的 autorun 中的 handler 方法
let counter = 0 // 记录一个计数器值作为每个 observable 属性的 id 值进行和 nowFn 进行绑定

class Reaction {
  constructor() {
    this.id = ++counter // 每次对 observable 属性进行 Proxy 的时候，对 Proxy 进行标记
    this.store = {} // 存储当前可观察对象的nowFn, { id: [nowFn] }
  }

  // 进行依赖搜集
  collect() {
    // 当前有需要绑定的函数才进行绑定
    if (nowFn) {
      this.store[this.id] = this.store[this.id] || []
      this.store[this.id].push(nowFn)
    }
  }

  // 运行依赖函数
  run() {
    if (this.store[this.id]) {
      this.store[this.id].forEach(w => {
        w()
      })
    }
  }

  // 用于在调用 autorun 方法时候对 nowFn 进行设置和消除
  static start(handler) {
    nowFn = handler
  }

  // 在注册绑定这个就要清空当前的 nowFn，用于之后进行搜集绑定
  static end() {
    nowFn = null
  }
}

export default Reaction
```

`./mobx/index.js`

```javascript
import autorun from './autorun'
import observable from './observable'

export { autorun, observable }
```

源码地址：[mini-mobx](https://github.com/Jacky-Summer/mini-mobx/tree/master)

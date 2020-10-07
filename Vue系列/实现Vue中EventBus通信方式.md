# 实现 Vue 中 EventBus 通信方式

## 前言

Vue 非父子组件可以用 Event Bus 通信，用起来很简单但要不只是会使用 api，还要懂大致原理，于是自己实现了一下。

## 代码实现

```javascript
class EventBus {
  constructor() {
    this._events = new Map()
  }

  // 注册事件
  on(name, fn) {
    const handler = this._events.get(name)
    if (!handler) {
      this._events.set(name, [])
    }
    this._events.get(name).push(fn)
  }

  /**
   * 触发事件
   * 可以用apply与call两种方法,在少数参数时call的性能更好,多个参数时apply性能更好
   * Node的Event模块就在三个参数以下用call否则用apply.
   */
  emit(name, ...args) {
    const handler = this._events.get(name)
    if (!handler) return
    handler.forEach(cb => {
      if (args.length >= 3) {
        cb.apply(this, args)
      } else {
        cb.call(this, ...args)
      }
    })
  }

  // 只被触发一次的事件
  once(name, fn) {
    const cb = (...args) => {
      if (args.length >= 3) {
        fn.apply(this, args)
      } else {
        fn.call(this, ...args)
      }
      this.off(name, cb)
    }
    this.on(name, cb)
  }

  // 取消事件
  off(name, fn) {
    if (typeof fn === 'undefined') {
      // 没有传指定函数就移除所有同名事件
      this._events.delete(name)
      return
    }
    const handler = this._events.get(name)
    if (!handler) return
    let fnIndex = handler.findIndex(cb => cb === fn)
    handler.splice(fnIndex, 1)
    if (!this._events.get(name).length) {
      this._events.delete(name)
    }
  }
}
```

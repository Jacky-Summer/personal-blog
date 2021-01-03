# ES6 系列之 Proxy

## 前言

前几天模拟实现了 MobX 的两个函数 —— [手写实现 MobX 的 observable 和 autorun 方法](https://github.com/Jacky-Summer/personal-blog/blob/master/React%E7%B3%BB%E5%88%97/%E6%89%8B%E5%86%99%E5%AE%9E%E7%8E%B0MobX%E7%9A%84observable%E5%92%8Cautorun%E6%96%B9%E6%B3%95.md)，其中用到了 Proxy，所以打算再对 Proxy 深入了解一下，做个笔记。

## Proxy 是什么

**Proxy** 对象用于创建一个对象的代理，从而实现基本操作的拦截和自定义（如属性查找、赋值、枚举、函数调用等）。可以理解成，在目标对象之前架设一层“拦截”，外界对该对象的访问，都必须先通过这层拦截，因此提供了一种机制，可以对外界的访问进行过滤和改写。

> const p = new Proxy(target, handler)

- target: 使用 Proxy 包装的目标对象（可以是任何类型的 JavaScript 对象，包括原生数组，函数，甚至另一个代理）。
- handler: 一个通常以函数作为属性的对象，用来定制拦截行为。

在支持 Proxy 的浏览器环境中，Proxy 是一个全局对象，可以直接使用。`Proxy(target, handler)`是一个构造函数，`target`是被代理的对象，最终返回一个代理对象。

## 为什么需要 Proxy

学习一样东西之前我们先要想想为什么需要它，在我看来，一般几种情况。

1. 被代理的对象不想直接被访问
2. 控制和修改被代理对象的行为（调用属性、属性赋值、方法调用等等），使之可以进行访问控制和增加功能。

## API

API 概览如下：

- **get(target, propKey, receiver)**：拦截对象属性的读取，比如 proxy.foo 和 proxy['foo'] 。
- **set(target, propKey, value, receiver)**：拦截对象属性的设置，比如 proxy.foo = v 或 proxy['foo'] = v ，返回一个布尔值。
- **has(target, propKey)**：拦截 propKey in proxy 的操作，返回一个布尔值。
- **deleteProperty(target, propKey)**：拦截 delete proxy[propKey]的操作，返回一个布尔值。
- **ownKeys(target)**：拦截 Object.getOwnPropertyNames(proxy) 、 Object.getOwnPropertySymbols(proxy) 、 Object.keys(proxy) 、 for...in 循环，返回一个数组。该方法返回目标对象所有自身的属性的属性名，而 Object.keys() 的返回结果仅包括目标对象自身的可遍历属性。
- **getOwnPropertyDescriptor(target, propKey)**：拦截 Object.getOwnPropertyDescriptor(proxy, propKey) ，返回属性的描述对象。
- **defineProperty(target, propKey, propDesc)**：拦截 Object.defineProperty(proxy, propKey, propDesc） 、
- **Object.defineProperties(proxy, propDescs)**，返回一个布尔值。
- **preventExtensions(target)**：拦截 Object.preventExtensions(proxy) ，返回一个布尔值。
- **getPrototypeOf(target)**：拦截 Object.getPrototypeOf(proxy)，返回一个对象 。
- **isExtensible(target)**：拦截 Object.isExtensible(proxy) ，返回一个布尔值。
- **setPrototypeOf(target, proto)**：拦截 Object.setPrototypeOf(proxy, proto) ，返回一个布尔值。如果目标对象是函数，那么还有两种额外操作可以拦截。
- **apply(target, object, args)**：拦截 Proxy 实例作为函数调用的操作，比如 proxy(...args)、proxy.call(object, ...args)、proxy.apply(...)`。
- **construct(target, args)**：拦截 Proxy 实例作为构造函数调用的操作，比如 new proxy(...args) 。

最常用的方法就是`get`和`set`

## 例子

### get

```javascript
const target = {
  name: 'jacky',
  sex: 'man',
}
const handler = {
  get(target, key) {
    console.log('获取名字/性别')
    return Reflect.get(target, key)
  },
}
const proxy = new Proxy(target, handler)
console.log(proxy.name)
```

运行，打印台输出:

```
获取名字/性别
jacky
```

在获取`name`属性是先进入`get`方法，在`get`方法里面打印了`获取名字/性别`，然后通过`Reflect.get(target, key)`的返回值拿到属性值，相当于`target[key]`

### set

```javascript
const target = {
  name: 'jacky',
  sex: 'man',
}
const handler = {
  get(target, key) {
    console.log('获取名字/性别')
    return Reflect.get(target, key)
  },
  set(target, key, value) {
    return Reflect.set(target, key, `强行设置为 ${value}`)
  },
}
const proxy = new Proxy(target, handler)
proxy.name = 'monkey'
console.log(proxy.name)
```

运行输出：

```
获取名字/性别
强行设置 monkey
```

设置`proxy.name = 'monkey'`，这是修改属性的值，则会触发到`set`方法， 然后我们强行更改设置的值再返回，达到拦截对象属性的设置的目的。

---

```
Reflect对象与Proxy对象一样，也是 ES6 为了操作对象而提供的新 API。
Reflect.get(target, name, receiver) ：查找并返回target对象的name属性，如果没有该属性，则返回undefined。
Reflect.set(target, name, value, receiver) ：设置target对象的name属性等于value。
```

## this 指向

proxy 会改变 target 中的 this 指向，一旦 Proxy 代理了 target，target 内部的 this 则指向了 Proxy 代理

```javascript
const target = new Date('2021-01-03')
const handler = {
  get(target, prop) {
    if (prop === 'getDate') {
      return target.getDate(target)
    }
    return Reflect.get(target, prop)
  },
}
const proxy = new Proxy(target, handler)

console.log(proxy.getDate())
```

运行代码，会发现报错，提示`TypeError: proxy.getDate is not a function`，即 this 不是 Date 对象的实例，这时需要我们手动绑定原始对象即可解决：

```javascript
const target = new Date('2021-01-03')
const handler = {
  get(target, prop) {
    if (prop === 'getDate') {
      return target.getDate.bind(target) // 绑定
    }
    return Reflect.get(target, prop)
  },
}
const proxy = new Proxy(target, handler)

console.log(proxy.getDate()) // 3
```

## 应用场景

- 警告或阻止特定操作
- get 方法取不到对应值可以返回我们想指定的其它值
- 数据校验。判断数据是否满足条件

等等...

## 参考

- [https://es6.ruanyifeng.com/#docs/proxy](https://es6.ruanyifeng.com/#docs/proxy)

<br>

- ps：
  - [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)

觉得不错的话赏个 star，给我持续创作的动力吧！

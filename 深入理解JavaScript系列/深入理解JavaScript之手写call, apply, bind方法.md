# 深入理解 JavaScript 之手写 call, apply, bind 方法

这是老生常谈的手写了，今天想自己试着实现一下，做个笔记。

<!--more-->

## call 方法

```javascript
Function.prototype.myCall = function (context) {
  if (context === undefined || context === null) {
    context = window // 指定为 null 和 undefined 的 this 值会自动指向全局对象
  } else {
    context = Object(context) // 值为原始值（数字，字符串，布尔值）的 this 会指向该原始值的实例对象
  }

  const fn = Symbol('uniqueFn') // 用 Symbol 是防止跟上下文的原属性冲突
  context[fn] = this
  let arg = [...arguments].slice(1)
  let result = context[fn](...arg)
  delete context[fn]
  return result
}
```

判断函数上下文网上也有写法是：

```javascript
context = context ? Object(context) : window
context = context || window
```

但不是很严谨，因为遇到空字符串，`0`，`false`时，`context`也会被判断为`window`。

## apply 方法

```javascript
Function.prototype.myApply = function (context) {
  if (context === undefined || context === null) {
    context = window
  } else {
    context = Object(context)
  }

  // 判断是否为类数组对象
  function isArrayLike(o) {
    if (
      o && // o 不是null、undefined等
      typeof o === 'object' && // o是对象
      isFinite(o.length) && // o.length是有限数值
      o.length >= 0 && // o.length为非负值
      o.length === Math.floor(o.length) && // o.length是整数
      o.length < Math.pow(2, 32)
    ) {
      return true
    }
    return false
  }
  const fn = Symbol('uniqueFn')
  context[fn] = this
  let args = arguments[1]
  let result
  if (!Array.isArray(args) && !isArrayLike(args)) {
    throw new Error('CreateListFromArrayLike called on non-object') // 第二个参数不为数组且不为类对象数组
  } else {
    args = Array.from(args) // 转为数组
    result = context[fn](...args)
  }
  delete context[fn]
  return result
}
```

## bind 方法

实现 bind 要额外考虑一个问题：方法一个绑定函数也能使用 new 操作符创建对象：这种行为就像把原函数当成构造器。提供的 this 值被忽略，同时调用时的参数被提供给模拟函数

```javascript
Function.prototype.myBind = function (context) {
  if (typeof this !== 'function') {
    throw new Error('Expected this is a function')
  }
  let self = this
  let args = [...arguments].slice(1)
  let fn = function (...innerArg) {
    const isNew = this instanceof fn // 返回的fn是否通过new调用
    return self.apply(isNew ? this : Object(context), args.concat(innerArg)) // new调用就绑定到this上,否则就绑定到传入的context上
  }
  // 复制原函数的prototype给fn，一些情况下函数没有prototype，比如箭头函数
  if (self.prototype) {
    fn.prototype = Object.create(self.prototype)
  }
  return fn // 返回拷贝的函数
}
```

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，你的鼓励是我创作的动力~

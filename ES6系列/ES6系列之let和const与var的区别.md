# ES6 系列之 let 和 const 与 var 的区别

ES6 规范新增了 let、const 两种变量声明方式，项目中也经常要用到，今天借着温习 ES6 语法，来总结 let 、const、var 的区别。

## 一、变量提升

来看下面三段代码

```javascript
console.log(a) // undefined
var a = 2
```

```javascript
console.log(b) // Uncaught ReferenceError: b is not defined
let b = 2
```

```javascript
console.log(c) // Uncaught ReferenceError: c is not defined
const c = 2
```

用 var 声明的变量，会在其作用域中发生变量提升，js 默认给变量一个 undefined 值。

但是，在 ES6 中使用 let/const 声明的变量，不存在变量提升过程。也就是说，在使用 let/const 声明的变量，声明前访问它，都会报错。

## 二、暂时性死区

> 代码块内，使用 let 命令声明变量之前，该变量都是不可用的。这在语法上，称为“暂时性死区”（temporal dead zone，简称 TDZ）。

```javascript
let a = 'global'
if (true) {
  a = 'block' // Uncaught ReferenceError: a is not defined
  let a
}
```

上述代码中，if 代码块里，在 let 声明变量 a 之前，都是 a 的"暂时性死区",在该范围内访问 a 都会报错。
由此可见，块级作用域内存在 let 声明的话，它所声明的变量就绑定在这个块级作用域，不再受外部影响。

## 三、重复声明

let 和 const 命令声明的变量不允许重复声明；而使用 var 声明变量，可以多次重复声明一个同名变量，但最终变量的值为最后一次声明赋值的结果。

```javascript
var a = 10
var a = 'abc'
var a = 'last value'
console.log(a) // last value
```

let 与 const 在相同作用域声明重复的变量会报错

```javascript
let a = 10
let a = 20 // Uncaught SyntaxError: Identifier 'a' has already been declared
```

var 和 let 同时声明同一个变量，也是报同样的错误

```javascript
let b = 10
var b = 20 // Uncaught SyntaxError: Identifier 'b' has already been declared
```

或先 var，再 let

```javascript
var c = 10
let c = 20 // Uncaught SyntaxError: Identifier 'c' has already been declared
```

## 四、作用域

这里就涉及到一个最经典的问题：

```javascript
for (var i = 0; i < 5; i++) {
  setTimeout(function () {
    console.log(i) // 5 5 5 5 5
  })
}
```

这个问题的解决可以使用闭包等，ES6 的 let 为这个问题提供了新的解决方法：

```javascript
for (let i = 0; i < 5; i++) {
  setTimeout(function () {
    console.log(i) // 0 1 2 3 4
  })
}
```

使用 let 声明的变量仅在块级作用域内有效。 i 在循环体内的局部作用域，不受外界影响。

在 ES6 之前，都是用 var 来声明变量，而且 JS 只有函数作用域和全局作用域，没有块级作用域，所以花括号{}限定不了 var 声明变量的访问范围。

```javascript
{
  var a = 10
}
console.log(a) // 10

{
  let b = 9 // es6的let: b变量只在花括号内有效
}
console.log(b) // Uncaught ReferenceError: b is not defined
```

## 五、let、const 声明的全局变量不会作为 window 对象的一个属性

使用 var 声明的全局变量，会被 JS 自动添加在全局对象 window 上，但 let 和 const 不会

```javascript
let a = 10
console.log(window.a) // undefined

var b = 20
console.log(window.b) // 20

const c = 30
console.log(window.c) // undefined
```

## 六、const 声明常量

由于 const 用来声明常量，一旦声明，就必须立即初始化，而且声明之后值不能改变。

```javascript
const a = 10
a = 20 // Uncaught TypeError: Assignment to constant variable.
```

常量的值不变，实际上是指常量指向的那个内存地址中所保存的数据不可更改。

对于基本数据类型（数值，字符串、布尔值），他们本身具体的值就保存在常量所指向对应的栈内存地址中，所以修改值就等于修改栈内存地址，这显然不允许会报错。

但是，如果一个常量的值是一个引用类型值，那么常量所指向的内存地址（堆内存）中实际保存的是指向该引用类型值的一个指针（也就是引用类型值在内存中的地址）。所以 const 只能保证该引用类型地址不变，但该地址中的具体数据是可以变化的。

如下代码：

```javascript
const obj = {}
obj.a = 3
console.log(obj) // {a: 3}

// 当obj指向了另一个对象，即obj中保存的地址发生了变化，即会报错
obj = {} // Uncaught TypeError: Assignment to constant variable.
```

## 如何正确使用

在开发的时候，声明变量我们应该少使用 var，避免产生不必要的全局变量。当需要改变变量的值时声明用 let，对于需要写保护的变量或定义常量使用 const。

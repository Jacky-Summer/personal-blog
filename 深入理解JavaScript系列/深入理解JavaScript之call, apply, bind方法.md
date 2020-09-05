# 深入理解之 JavaScript 之 call, apply, bind 方法

在 JavaScript 中，call、apply 和 bind 是 Function 对象自带的三个方法，这三个方法的主要作用是改变函数执行时的上下文，再具体一点就是**改变函数运行时的 this 指向**。

- ## Function.prototype.call()

call() 方法调用一个函数, 其具有一个指定的 this 值和多个参数(参数的列表)。

> fun.call(thisArg, arg1, arg2, ...)

thisArg 的取值有以下 4 种情况：

1. 不传，或者传 null, undefined， 函数中的 this 指向 window 对象（非严格模式下）
2. 传递另一个函数的函数名，函数中的 this 指向这个函数的引用
3. 传递字符串、数值或布尔类型等基础类型，函数中的 this 指向其对应的包装对象，如 String、Number、Boolean
4. 传递一个对象，函数中的 this 指向这个对象.

我们可以用如下代码验证一下：

```javascript
function func1() {
  console.log(this)
}

function func2() {}

var obj = { name: 'jacky' }

func1.call() // Window
func1.call(null) // Window
func1.call(undefined) // Window
func1.call(1) // Number {1}
func1.call('') // String {""}
func1.call(true) // Boolean {true}
func1.call(func2) // ƒ func2() {}
func1.call(obj) // {name: "jacky"}
```

我们再来看个例子，理解怎么改变 this 的指向：

```javascript
function Animal() {
  this.name = 'animal'
  this.sayName = function () {
    console.log(this.name)
  }
}

function Cat() {
  this.name = 'cat'
}

var animal = new Animal()
var cat = new Cat()

animal.sayName.call(cat) // cat
// this 永远指向最后调用它的那个对象
// 该例子中sayName方法的this指向Cat
```

- ## Function.prototype.apply()

call 和 apply 的第一个参数都是要改变上下文的对象，而 call 从第二个参数开始以参数列表的形式展现，apply 则是把除了改变上下文对象的参数放在一个**数组**里面作为它的第二个参数，他们俩之间的差别在于**参数**的区别。

如下代码：

```javascript
function Animal(...args) {
  console.log(`${this.name} 和其他参数 ${args}`)
}

let cat = {
  name: 'xiaomao',
}

// 1. 使用 call
Animal.call(cat, 1, 2, 3) // xiaomao 和其他参数 1,2,3

// 2. 使用 apply
Animal.apply(cat, [1, 2, 3]) // xiaomao 和其他参数 1,2,3
```

- ## call、apply 使用

由于两个方法实际效果是一样的，对于两者平时用该如何选择呢？

1. 参数数量/顺序确定就用 call，参数数量/顺序不确定的话就用 apply。
2. 考虑可读性：参数数量不多就用 call，参数数量比较多的话，把参数整合成数组，使用 apply。
3. 参数集合已经是一个数组的情况，用 apply。

- ## call, apply 的应用

### 1. 数组拼接

```javascript
let arr1 = [1, 2, 3]
let arr2 = [4, 5, 6]

// 用 apply方法
;[].push.apply(arr1, arr2) // 给arr1添加arr2
console.log(arr1) // [1, 2, 3, 4, 5, 6]
```

### 2. 获取数组中的最大值或最小值

```javascript
var numbers = [1, 4, 6, 2, 3, 100, 98]
console.log(Math.max.apply(Math, numbers)) // 100
console.log(Math.max.call(Math, 1, 4, 6, 2, 3, 100, 98)) // 100
console.log(Math.min.call(Math, ...numbers)) // 1
```

### 3. 判断数据类型

```javascript
let arr = [1, 2, 3, 4]
let str = 'string',
  obj = { name: 'jacky' }
// 判断传入参数是否为数组
function isArray(obj) {
  return Object.prototype.toString.call(obj) === '[object Array]'
}
console.log(isArray(arr)) // true
console.log(isArray(str)) // false
console.log(Object.prototype.toString.call(arr)) // [object Array]
console.log(Object.prototype.toString.call(str)) // [object String]
console.log(Object.prototype.toString.call(obj)) // [object Object]
console.log(Object.prototype.toString.call(null)) // [object Null]
```

### 4. 调用父类构造函数实现继承

```javascript
function Animal(name) {
  this.name = name
  this.showName = function () {
    console.log(this.name)
  }
}

function Cat(name) {
  Animal.call(this, name)
}

var cat = new Cat('xiaomao')
cat.showName() // xiaomao
```

缺点：

1. 只能继承父类的实例属性和方法，不能继承原型属性/方法
2. 每次子类实例化都要执行父类函数，重新声明父类 this 里所定义的方法，因此父类方法无法复用。

### 5. 类数组对象转数组

```javascript
function func() {
  var args = [].slice.call(arguments)
  console.log(args)
}
func(1, 2, 3) // [1, 2, 3]
```

还有像调用 getElementsByTagName , document.childNodes 之类的，它们返回 NodeList 对象都属于伪数组。

### 6. 代理 console.log 方法

注意这里传入多少个参数是不确定的，所以使用 apply 是最好的。

```javascript
function log() {
  console.log.apply(console, arguments)
}
log(1) // 1
log(1, 2, 3) // 1 2 3
```

- ## Function.prototype.bind()

MDN 的解释是：

> bind()方法会创建一个新函数，称为绑定函数，当调用这个绑定函数时，绑定函数会以创建它时传入 bind()方法的第一个参数作为 this，传入 bind() 方法的第二个以及以后的参数加上绑定函数运行时本身的参数按照顺序作为原函数的参数来调用原函数。

说直白一点，bind 方法是创建一个函数，然后可以在需要调用的时候再执行函数，并非是立即执行函数；而 call，apply 是在改变了上下文中的 this 指向后并立即执行函数。

bind 接收的参数类型与 call 是一样的，给它传递的是一组用逗号分隔的参数列表。

```javascript
var bar = function () {
  console.log(this.x)
}
var foo = {
  x: 2,
}
bar() // undefined

var func = bar.bind(foo)
func() // 2
```

- ## bind 的应用

### bind 绑定回调函数的 this 指向

```javascript
class PageA {
  constructor(callBack) {
    this.className = 'PageA'
    this.MessageCallBack = callBack
    this.MessageCallBack('执行了')
  }
}
class PageB {
  constructor() {
    this.className = 'PageB'
    this.pageClass = new PageA(this.handleMessage)
  }
  // 回调函数
  handleMessage(msg) {
    console.log('处理' + this.className + '的回调 ' + msg) // 处理PageA的回调 执行了
  }
}
new PageB()
```

运行上面的代码，我们发现回调函数 this 丢失了？问题出在这行代码

```javascript
this.pageClass = new PageA(this.handleMessage)
```

传递过去的 this.handleMessage 是一个函数内存地址，没有上下文对象，也就是说该函数没有绑定它的 this 指向。

解决方案：用 bind 绑定 this 的指向

```javascript
this.pageClass = new PageA(this.handleMessage.bind(this))
```

这也是为什么 react 的 render 函数在绑定回调函数的时候，也要使用 bind 绑定一下 this 的指向，也是因为同样的问题以及原理。

- 多个 bind 的情况

```javascript
var bar = function () {
  console.log(this.x)
}

var obj1 = {
  x: 2,
}
var obj2 = {
  x: 3,
}
var obj3 = {
  x: 4,
}

var func1 = bar.bind(obj1).bind(obj2)
var func2 = bar.bind(obj1).bind(obj2).bind(obj3)
func1() // 2
func2() // 2
```

输出结果都为第一个绑定的 obj 对象的 x 值。原因是，在 Javascript 中，bind()方法返回的外来的绑定函数对象仅在创建的时候记忆上下文（如果提供了参数），多次 bind() 是无效的。更深层次的原因， bind() 的实现，相当于使用函数在内部包了一个 call / apply ，第二次 bind() 相当于再包住第一次 bind() ,故第二次以后的 bind 是无法生效的。

- ## 总结

call 和 apply 的第一个参数都是要改变上下文的对象，而 call 从第二个参数开始以参数列表的形式展现; apply 则是把除了改变上下文对象的参数放在一个数组里面作为它的第二个参数，他们俩之间的差别在于参数的区别。

call 和 apply 改变了函数的 this 上下文后便执行该函数,而 bind 则是返回改变了上下文后的一个函数,其内的 this 指向为创建它时传入 bind 的第一个参数，而传入 bind 的第二个及以后的参数作为原函数的参数来调用原函数。bind 的参数可以在函数执行的时候再次添加。

# 深入理解 JavaScript 之原型与原型链

## 1.原型 prototype

原型是一个对象，把 prototype 称为原型对象，prototype 可以让所有的对象实例共享它包含的属性和方法。

JavaScript 规定，每一个函数都有一个 prototype 对象属性，指向另一个对象。prototype 对象属性的所有属性和方法都会被构造函数的实例继承。

## 2.只有函数有 prototype 属性

js 分为函数对象和普通对象，每个对象都有**proto**属性，但是只有函数对象才有 prototype 属性

```javascript
let a = {}
let func = function () {}
console.log(a.prototype) // undefined
console.log(func.prototype) // { constructor: function(){...} }
```

JS 通过 new 关键字，即靠构造函数生成对象，每次生成的对象都不一样。因为常常需要在两个对象之间共享属性，由于 JS 在设计之初没有类的概念，所以 JS 使用函数的 prototype 属性来处理这部分需要被共享的属性。

## 3.构造函数创建对象

```javascript
function Animal() {}
Animal.prototype.name = 'kiki'
var animal1 = new Animal()
var animal2 = new Animal()
console.log(animal1.name, animal2.name) // kiki kiki
```

在上述代码中，函数的 prototype 属性指向了一个对象，这个对象正是调用该构造函数而创建的实例的原型，也就是这个例子中的 animal1 和 animal2 的原型

每一个 JavaScript 对象(null 除外)在创建的时候就会与之关联另一个对象，这个对象就是我们所说的原型，每一个对象都会从原型"继承"属性。

构造函数和实例原型关系如下图：

![](https://user-gold-cdn.xitu.io/2020/2/15/17049498b6d308cb?w=885&h=207&f=png&s=21292)

## 4.**proto**

那么如何表示实例与实例原型之间的关系呢，这里需要讲到另一个属性**proto**

每一个 JavaScript 对象(除了 null)都具有的一个属性，叫**proto**，这个属性会指向该对象的原型

```javascript
function Animal() {}

var dog = new Animal()
console.log(dog.__proto__ === Animal.prototype) // true
```

于是更新实例与实例原型之间的关系如下图：

![](https://user-gold-cdn.xitu.io/2020/2/15/17049522bfa417ec?w=866&h=336&f=png&s=32709)

## 5.constructor

每个原型都有一个 constructor 属性指向关联的构造函数

```javascript
function Animal() {}
console.log(Animal === Animal.prototype.constructor) // true
```

![](https://user-gold-cdn.xitu.io/2020/2/15/170496796080e487?w=849&h=347&f=png&s=35480)

则可以得出：

```javascript
function Animal() {}
var dog = new Animal()
console.log(dog.__proto__ === Animal.prototype) // true
// Object.getPrototypeOf() 方法返回指定对象的原型
console.log(Object.getPrototypeOf(dog) === Animal.prototype) // true

console.log(Animal.prototype.constructor == Animal) // true
```

再提一点：

```javascript
function Animal() {}
var dog = new Animal()
console.log(dog.constructor === Animal) // true
/* 当获取 dog.constructor 时，其实 dog 中并没有 constructor 属性,
当不能读取到constructor 属性时，
会从 dog 的原型也就是 Animal.prototype 中读取，正好原型中有该属性, 所以下列代码为true */
dog.constructor === Animal.prototype.constructor
```

## 6.实例和原型

当读取实例的属性时，如果找不到，就会查找与对象关联的原型中的属性，如果还查不到，就去找原型的原型，一直找到最顶层为止。

```javascript
function Animal() {}

Animal.prototype.name = 'kiki'

var dog = new Animal()

dog.name = 'xiaobai'
console.log(dog.name) // xiaobai

delete dog.name
console.log(dog.name) // kiki
```

上述代码。首先构造函数原型设置了 name 值，接着 dog 实例添加了 name 属性，覆盖原型的 name 值，打印 dog.name 时结果自然是 xiaobai。

当删除了 dog 的 name 属性时，读取 dog.name 时，从对象中无法找到 name 属性，就会从 dog 的原型，也就是 dog.**proto**，即 Animal.prototype 中查找，找到结果为 kiki。

如果原型没有设置 name 值，找不到呢？原型的原型又是谁？

## 7.原型的原型

原型本身也是一个对象，既然是对象，我们就可以用最原始的方式创建它：

```javascript
var obj = new Object()
obj.name = 'kiki'
console.log(obj.name) // kiki
```

其实原型对象就是通过 Object 构造函数生成的，所有函数的 默认原型 都是 Object 的实例，因此默认原型都会包含一个内部指针，指向 Object.prototype。

由此，更新关系图：

![](https://user-gold-cdn.xitu.io/2020/2/16/1704bf72a35f1a74?w=861&h=505&f=png&s=54960)

## 8.原型链

1. 每个对象都拥有一个原型对象: dog 的原型是 Animal.prototype。
2. 对象的原型可能也是继承其他原型对象的: Animal.prototype 也有它的原型 Object.prototype。
3. 一层一层的，以此类推，这种关系就是原型链。

那么`Object.prototype`的原型是什么呢？

```javascript
console.log(Object.prototype.__proto__ === null) // true
const proto = Object.getPrototypeOf(Object.prototype) // null
```

所以 Object.prototype.**proto** 的值为 null 跟 Object.prototype 没有原型，其实表达了一个意思。null 表示此处不应该有值，也就是原型链的终点了，所以查找属性的时候查到 Object.prototype 就可以停止查找了。

![](https://user-gold-cdn.xitu.io/2020/2/16/1704bffc23da9aaf?w=800&h=592&f=png&s=54550)

上图中的红色线组成的链就可以称之为原型链。

由上述关系图我们可以重新描述一下原型链：

> 从一个实例对象开始往上找，这个实例对象的**proto**属性所指向的则是这个实例对象的原型对象，如果用 dog 表示这个实例，则原型对象表示为`dog.__proto__`。同时，这个原型也是一个对象，而且它也有上一级的原型对象，相对于上一级原型对象而言，它也是一个实例对象，那么它也拥有**proto**属性，它的**proto**属性也指向它的原型对象，后面也以此类推，一直到 Object.prototype 这个原型为止，这就是整个原型链。

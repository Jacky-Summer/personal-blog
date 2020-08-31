## 1.原型链继承

核心：**将父类的实例作为子类的原型**

首先，要知道构造函数、原型和实例之间的关系：构造函数都有一个原型对象，原型对象都包含一个指向构造函数的指针，而实例都包含一个原型对象的指针。

```javascript
function Father() {
  this.name = '父类的名字'
}
Father.prototype.getFatherName = function () {
  console.log('父类的方法')
}

function Son() {
  this.name = '子类的名字'
}
// 如果此时有Son的原型对象有方法或属性，下面Son.prototype = new Father()，由于原型重定向，原型上的方法和属性会丢失
Son.prototype.getAge = function () {
  console.log('子类的年龄')
}

Son.prototype = new Father() // 核心:创建父类的实例，并将该实例赋值给子类的prototype

Son.prototype.getSonName = function () {
  console.log('子类的方法')
}

var son = new Son()
son.getFatherName() // 父类的方法
Son.prototype.__proto__.getFatherName = function () {
  // 缺点：如果有多个实例对其父类原型，则会互相影响
  console.log('子类改变父类的方法')
}
son.getFatherName() // 子类改变父类的方法
```

缺点：

1. 父类使用 this 声明的属性(私有属性和公有属性)被所有实例共享,在多个实例之间对引用类型数据操作会互相影响。

2. 创建子类实例时，无法向父类构造函数传参。

## 2.借用构造函数继承(call)

核心：**使用父类的构造函数来增强子类实例**，即复制父类的实例属性给子类

```javascript
function Father(name, age) {
  this.name = name
  this.age = age
}
Father.prototype.getFatherName = function () {
  console.log('父类的方法')
}

function Son(name, age, job) {
  Father.call(this, name, age) // 继承自Father
  this.job = job
}
var son = new Son('jacky', 22, '前端开发')
//son.getFatherName(); // Uncaught TypeError: son.getFatherName is not a function
```

优点：

1. 可以向父类传递参数,而且解决了原型链继承中：父类属性使用 this 声明的属性会在所有实例共享的问题。

缺点：

1. 只能继承父类通过 this 声明的属性/方法，不能继承父类 prototype 上的属性/方法。
2. 每次子类实例化都要执行父类函数，重新声明父类 this 里所定义的方法，因此父类方法无法复用。

## 3.组合继承

核心：**组合上述两种方法，用原型链实现对原型属性和方法的继承，用借用构造函数技术来实现实例属性的继承**。

```javascript
function Father(name, age) {
  this.name = name
  this.age = age
  this.sex = 'man'
}
Father.prototype.getFatherName = function () {
  console.log('父类的方法')
}

function Son(name, age, job) {
  Father.call(this, name, age) // 第二次调用:创建子类型实例的时候
  this.job = job
}
Son.prototype = new Father() // 第一次调用:设置子类型实例的原型的时候
Son.prototype.constructor = Son // prototype构造器指回自己

var son = new Son('jacky', 22, '前端开发')
son.getFatherName()
console.log(son)
```

![](https://user-gold-cdn.xitu.io/2019/11/20/16e89812367319c7?w=517&h=227&f=png&s=19707)

优点：

1. 可以继承父类原型上的属性，可以传参，可复用。
2. 每个新子类对象实例引入的构造函数属性是私有的。

缺点：

1. 两次调用父类函数(new fatherFn()和 fatherFn.call(this))，造成一定的性能损耗。
2. 在使用子类创建实例对象时，其原型中会存在两份相同属性/方法的问题。

拓展：

### constructor 的作用

**返回创建实例对象的 Object 构造函数的引用**。

> 当我们只有实例对象没有构造函数的引用时：
> 某些场景下，我们对实例对象经过多轮导入导出，我们不知道实例是从哪个函数中构造出来或者追踪实例的构造函数，较为艰难。(它主要防止一种情况下出错，就是你显式地去使用构造函数。比如，我并不知道 instance 是由哪个函数实例化出来的，但是我想 clone 一个，这时就可以这样——>instance.constructor)
> 这个时候就可以通过实例对象的 constructor 属性来得到构造函数的引用

```javascript
let instance = new sonFn() // 实例化子类
export instance;
// 多轮导入+导出，导致sonFn追踪非常麻烦，或者不想在文件中再引入sonFn
let  fn = instance.constructor
```

因此每次**重写**函数的 prototype 都应该修正一下 constructor 的指向，以保持读取 constructor 指向的一致性

## 4.原型式继承（Object.create()）

核心：利用一个空对象作为中介，将某个对象直接赋值给空对象构造函数的原型，然后返回这个函数的调用，这个函数就变成了个可以随意增添属性的实例或对象。

```javascript
/* Object.create() 的实现原理 */
// cloneObject()对传入其中的对象执行了一次浅拷贝，将构造函数F的原型直接指向传入的对象。
function cloneObject(obj) {
  function F() {}
  F.prototype = obj // 将传进来obj对象作为空函数的prototype
  return new F() // 此对象的原型为被继承的对象, 通过原型链查找可以拿到被继承对象的属性
}

var father = {
  name: 'jacky',
  age: 22,
  courses: ['前端'],
}

// var son1 = Object.create(father); // 效果一样
var son1 = cloneObject(father)
son1.courses.push('后端')

var son2 = cloneObject(father)
son2.courses.push('全栈')

console.log(father.courses) //  ["前端", "后端", "全栈"]
```

优点：

从已有对象衍生新对象，不需要创建自定义类型

缺点：

与原型链继承一样。多个实例共享被继承对象的属性，存在篡改的可能；也无法传参。

## 5.寄生式继承

核心：**在原型式继承的基础上，创建一个仅用于封装继承过程的函数，该函数在内部以某种形式来做增强对象(增加了一些新的方法和属性)，最后返回对象。**

使用场景：专门为对象来做某种固定方式的增强。

```javascript
function createAnother(obj) {
  var clone = Object.create(obj)
  clone.skill = function () {
    // 以某种方式来增强这个对象
    console.log('run')
  }
  return clone
}

var animal = {
  eat: 'food',
  drink: 'water',
}

var dog = createAnother(animal)
dog.skill()
```

优点：没有创建自定义类型，因为只是套了个壳子增加特定属性/方法返回对象，以达到增强对象的目的

缺点：

同原型式继承：原型链继承多个实例的引用类型属性指向相同，存在篡改的可能，也无法传递参数

## 6.寄生组合式继承

核心：结合借用构造函数传递参数和寄生模式实现继承

1. 通过借用构造函数(call)来继承父类 this 声明的属性/方法
2. 通过原型链来继承方法

```javascript
function Father(name, age) {
  this.name = name
  this.age = age
}
Father.prototype.getFatherName = function () {
  console.log('父类的方法')
}

function Son(name, age, job) {
  Father.call(this, name, age) // 借用构造继承: 继承父类通过this声明属性和方法至子类实例的属性上
  this.job = job
}

// 寄生式继承：封装了son.prototype对象原型式继承father.prototype的过程，并且增强了传入的对象。
function inheritPrototype(son, father) {
  var clone = Object.create(father.prototype) // 原型式继承：浅拷贝father.prototype对象
  clone.constructor = son // 增强对象，弥补因重写原型而失去的默认的constructor 属性
  son.prototype = clone // 指定对象，将新创建的对象赋值给子类的原型
}
inheritPrototype(Son, Father) // 将父类原型指向子类

// 新增子类原型属性
Son.prototype.getSonName = function () {
  console.log('子类的方法')
}

var son = new Son('jacky', 22, '前端开发')
console.log(son)
```

![](https://user-gold-cdn.xitu.io/2019/11/23/16e97782ff645ca0?w=361&h=182&f=png&s=24333)

- 寄生组合式继承相对于组合继承有如下优点：

1. 只调用一次父类 Father 构造函数。不必为了指定子类的原型而调用构造函数，而是间接的让 Son.prototype 访问到 Father.prototype。
2. 避免在子类 prototype 上创建不必要多余的属性。
   使用原型式继承父类的 prototype，保持了原型链上下文不变, instanceof 和 isPrototypeOf()也能正常使用。 3.寄生组合式继承是最成熟的继承方法, 也是现在最常用的继承方法，众多 JS 库采用的继承方案也是它。

缺点：

硬要说的话，就是给子类原型添加属性和方法的时候，一定要放在 inheritPrototype()方法之后

## 7.ES6 extends 继承（最优方式）

核心： 类之间通过 extends 关键字实现继承，清晰方便。 class 仅仅是一个语法糖，它的核心思想仍然是寄生组合式继承。

```javascript
class Father {
  constructor(name, age) {
    this.name = name
    this.age = age
  }
  skill() {
    console.log('父类的技能')
  }
}

class Son extends Father {
  constructor(name, age, job) {
    super(name, age) // 调用父类的constructor,只有调用super之后，才可以使用this关键字
    this.job = job
  }

  getInfo() {
    console.log(this.name, this.age, this.job)
  }
}

let son = new Son('jacky', 22, '前端开发')
son.skill() // 父类的技能
son.getInfo() // jacky 22 前端开发
```

- 如果子类没有定义 constructor 方法，这个方法会被默认添加，代码如下。也就是说，不管有没有显式定义，任何一个子类都有 constructor 方法。

> 子类必须在 constructor 方法中调用 super 方法，否则新建实例时会报错。这是因为子类自己的 this 对象，必须先通过父类的构造函数完成塑造，得到与父类同样的实例属性和方法，然后再对其进行加工，加上子类自己的实例属性和方法。如果不调用 super 方法，子类就得不到 this 对象。

## ES5 继承与 ES6 继承的区别

1. ES5 的继承实质上是先创建子类的实例对象，再将父类的方法添加到 this 上( Father.call(this) )。
2. ES6 的继承是先创建父类的实例对象 this，再用子类的构造函数修改 this。
3. 因为子类没有自己的 this 对象，所以必须先调用父类的 super()方法。

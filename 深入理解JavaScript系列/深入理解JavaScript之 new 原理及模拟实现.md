# 深入理解 JavaScript 之 new 原理及模拟实现

## 1.定义

new 运算符创建一个用户定义的对象类型的实例或具有构造函数的内置对象的实例

先看看 new 实现了哪些功能, 先来看一段代码：

```javascript
function Person(age) {
  this.age = age
}
Person.prototype.getAge = function () {
  console.log('年龄为:' + this.age)
}

var person = new Person(18)
person.age // 访问构造函数里的属性
// 18

person.getAge() // 访问原型里的属性
// 年龄为:18
```

从上面代码可以知道，实例 person 可以：

1. 访问到 Person 构造函数里的属性
2. 访问到 Person.prototype 中的属性

这个是最基本的了，也是刚学会 new 一个对象就知道这是 new 的特点

## 3.探讨 new 还做了什么？

探讨完上面，接下来看看 new 还做了什么？来看几个例子

### 1. 没有 return 语句

```javascript
function Person(age) {
  this.age = age
}

var person = new Person(18)
console.log(person) // Person {age: 18}
```

从构造函数直观看，最后是没有 return 语句的，但我们从返回结果也可以看出构造函数时默认情况会返回一个新对象

### 2. return 对象数据类型

我们尝试在构造函数最后返回一个对象

```javascript
function Person(age) {
  this.age = age
  return { name: '手动返回一个对象' }
}

var person = new Person(18)
console.log(person) // {name: "手动返回一个对象"}
```

打印出来的结果可以看书：return 之前的代码片段都被覆盖了，最后返回 return 后面的对象。

### 3. return 基本数据类型

如果构造函数最后 return 的不是对象呢，试下基本数据类型

```javascript
function Person(age) {
  this.age = age
  return 1
}

var person = new Person(18)
console.log(person) // Person {age: 18}
```

从打印出来的结果可知，和没有 return 效果一样。

## 4.new 原理

mdn 上把内部操作大概分为 4 步：

> 1. 创建一个空的简单 JavaScript 对象（即{ } ）；
> 2. 链接该对象（即设置该对象的构造函数）到另一个对象 ；(因此 this 就指向了这个新对象)
> 3. 执行构造函数中的代码（为这个新对象添加属性）；
> 4. 如果该函数没有返回对象，则返回 this。

## 5.模拟实现 new

new 是关键词，不可以直接覆盖。这里使用 create 来模拟实现 new 的效果。

```javascript
function myNew() {
  // 创建一个空的对象
  var obj = new Object(),
    // 获得构造函数，arguments中去除第一个参数
    Con = [].shift.call(arguments)
  // 链接到原型，obj 可以访问到构造函数原型中的属性
  obj.__proto__ = Con.prototype
  // 绑定 this 实现继承，obj 可以访问到构造函数中的属性
  var ret = Con.apply(obj, arguments)
  // 优先返回构造函数返回的对象
  return ret instanceof Object ? ret : obj
}
```

来看看上面是怎么一步步模拟实现的：

1. 用 new Object() 的方式新建了一个空对象 obj
2. 取出第一个参数，即是传入的构造函数。shift 会修改原数组，数组原来的第一个元素的值。
3. 将 obj 的原型指向构造函数，这样 obj 就可以访问到构造函数原型中的属性
4. 使用 apply，改变构造函数 this 的指向。将 this 指向 obj 对象 就可以访问到构造函数中的属性
5. 处理返回值。

- 构造函数返回值有三种情况：

1. 没有 return，默认返回之前创建的对象
2. 返回一个对象
3. 返回的不是对象，默认返回之前创建的空对象

在上面我们用 instanceof 方法来判断是否为对象

> instanceof 运算符用于检测构造函数的 prototype 属性是否出现在某个实例对象的原型链上。

测试：

```javascript
function Person(age) {
  this.age = age
}
Person.prototype.getAge = function () {
  console.log('年龄为:' + this.age)
}

var person = create(Person, 18)
person.age // 访问构造函数里的属性
// 18

person.getAge() // 访问原型里的属性
// 年龄为:18
```

模拟实现成功！

# ES6 系列之变量的解构赋值

## 1.什么是解构？

ES6 允许按照一定模式，从数组和对象中提取值，对变量进行赋值，这被称为解构。它在语法上比 ES5 所提供的更加简洁、紧凑、清晰。它不仅能减少你的代码量，还能从根本上改变你的编码方式。

## 2.数组解构

以前，为变量赋值，我们只能直接指定值，比如

```javascript
let a = 1
let b = 2
let c = 3
```

现在可以用数组解构的方式来进行赋值

```javascript
let [a, b, c] = [1, 2, 3]
console.log(a, b, c) // 1, 2, 3
```

这是数组解构最基本类型的用法，还可以解构对象数组

```javascript
// 对象数组解构
let [a, b, c] = [{ name: 'jacky' }, { name: 'monkey' }, { name: 'houge' }]
console.log(a, b, c) // {name: 'jacky'}, {name: 'monkey'}, {name: 'houge'}
```

## 3.数组模式和赋值模式统一

这条可以理解为等号左边和等号右边的形式要统一，如果不统一解构将失败。

```javascript
let [a, [b, c], d] = [1, [2, 3], 4]
console.log(a, b, c, d) // 1 2 3 4

// 提取除第二、三个外的所有数值
let [a, , , d] = [1, 2, 3, 4]
console.log(a, d) //1 4

let [a, ...b] = [1, 2, 3, 4]
console.log(a, b) // 1 [2, 3, 4]

let [a, , , ...d] = [1, 2, 3, 4, 5]
console.log(a, d) // 1 [4, 5]
```

如果解构不成功，变量的值就等于`undefined`

```javascript
let [a, b, c] = [2, 3]
console.log(a, b, c) // 2 3 undefined

let [c] = []
console.log(c) // undefined
```

上述是完全解构的情况，还有一种是不完全解构，即等号左边的模式，只匹配一部分的等号右边的数组，解构依然可以成功。

```javascript
let [x, y] = [1, 2, 3]
console.log(x, y) // 1 2

let [a, [b], d] = [1, [2, 3], 4]
console.log(a, b, d) // 1 2 4
```

## 4.解构的默认值

解构赋值允许指定默认值。

```javascript
let [a, b = 2] = [1]
console.log(a, b) // 1 2

let [a = 1, b = 2, c, d = 13] = [10, 11, 12]
console.log(a, b, c, d) // 10 11 12 13
```

## 5.对象的解构赋值

对象的解构与数组有一个重要的不同。数组的元素是按次序排列的，变量的取值由它的位置决定；而对象的属性没有次序，变量必须与属性同名，才能取到正确的值。

```javascript
// 对象解构赋值的内部机制，是先找到同名属性，然后再赋给对应的变量。真正被赋值的是后者，而非前者。
let obj = { a: 'aaa', b: 'bbb' }
let { a: x, b: y } = obj
console.log(x, y) // aaa bbb

let { a, b } = { a: 'aaa', b: 'bbb' }
console.log(a, b) // aaa bbb

// 不按照顺序
let { b, a } = { a: 'test1', b: 'test2' }
console.log(a, b) // test1 test2

// 嵌套解构
let {
  obj: { name },
} = { obj: { name: 'jacky', age: '22' } }
console.log(name) // jacky

// 稍微复杂的嵌套
let obj = {
  p: ['Hello', { y: 'World' }],
}

let {
  p: [x, { y }],
} = obj
console.log(x, y) // Hello World
```

如果变量名与属性名不一致，必须写成下面这样。

```javascript
var { foo: rename } = { foo: 'aaa', bar: 'bbb' }
console.log(rename) // aaa
console.log(foo) // Uncaught ReferenceError: foo is not defined
```

如果在解构之前就定义了变量，这时候再解构会出现问题。下面是错误的代码，编译会报错（因为 js 引擎会将{a}理解成一个代码块，从而发生语法错误。只有不将大括号写在行首，避免 js 将其解释成代码块，才能解决这个问题）

```javascript
let a;
let obj = { a: "aaa" };
{a} = obj; // Uncaught SyntaxError: Unexpected token '='
```

要解决报错，使程序正常，这时候只要在解构的语句外边加一个圆括号就可以了

```javascript
let a
let obj = { a: 'aaa' }
;({ a } = obj)
console.log(a) // aaa
```

## 6.函数参数

函数的参数也可以使用解构赋值。

```javascript
function add([x, y]) {
  return x + y
}

add([1, 2]) // 3
```

函数参数的解构也可以使用默认值。

```javascript
function fn(x, y = 7) {
  return x + y
}
console.log(fn(3)) // 10
```

## 7.字符串解构

字符串被转换成了一个类似数组的对象。

```javascript
const [a, b, c, d, e, f] = 'hello'
console.log(a) //h
console.log(b) //e
console.log(c) //l
console.log(d) //l
console.log(e) //o
console.log(f) //undefined
```

## 8.数值和布尔值的解构赋值

解构赋值时，如果等号右边是数值和布尔值，则会先转为对象。

```
let {toString: s} = 0;
console.log(s === Number.prototype.toString); // true

let {toString: s} = true;
console.log(s === Boolean.prototype.toString); // true
```

解构赋值的规则是，只要等号右边的值不是对象或数组，就先将其转为对象。由于`undefined`和`null`无法转为对象，所以对它们进行解构赋值，都会报错。

```javascript
let { prop: x } = undefined // TypeError
let { prop: y } = null // TypeError
```

## 9.解构赋值的应用

### 1.交换变量的值

通常交换两个变量的方法需要一个额外的临时变量,如下

```javascript
let a = 1
let b = 2
let temp

temp = a
a = b
b = temp

console.log(a, b) // 2 1
```

用 ES6 解构赋值的话，会变得很简洁

```javascript
let a = 1
let b = 2
;[a, b] = [b, a]

console.log(a, b) // 2 1
```

### 2.从函数返回多个值

函数只能返回一个值，如果要返回多个值，只能将它们放在数组或对象里返回。有了解构赋值，取出这些值就非常方便。

```javascript
// 返回一个数组
function example() {
  return [1, 2, 3]
}
let [a, b, c] = example()

// 返回一个对象
function example() {
  return {
    foo: 1,
    bar: 2,
  }
}
let { foo, bar } = example()
```

### 3.访问数组中元素

有种场景，比如有一个数组（可能为空）。并且希望访问数组的第一个、第二个或第 n 个项，但如果该项不存在，则使用指定默认值。
通常会使用数组的`length`属性来判断

```javascript
const list = []

let firstItem = 'hello'
if (list.length > 0) {
  firstItem = list[0]
}

console.log(firstItem) // hello
```

如果用 ES6 解构赋值来实现上述逻辑

```javascript
const list = []
const [firstItem = 'hello'] = list

console.log(firstItem) // 'hello'
```

### 4.提取 JSON 数据

```javascript
let jsonData = {
  id: 42,
  status: 'OK',
  data: [867, 5309],
}

let { id, status, data: number } = jsonData

console.log(id, status, number)
// 42, "OK", [867, 5309]
```

### 5.遍历 Map 结构

任何部署了 Iterator 接口的对象，都可以用`for...of`循环遍历。Map 结构原生支持 Iterator 接口，配合变量的解构赋值，获取键名和键值就非常方便。

```javascript
const map = new Map()
map.set('first', 'hello')
map.set('second', 'world')

for (let [key, value] of map) {
  console.log(key + ' is ' + value)
}
// first is hello
// second is world
```

如果只想获取键名，或者只想获取键值，可以写成下面这样。

```javascript
// 获取键名
for (let [key] of map) {
  // ...
}

// 获取键值
for (let [, value] of map) {
  // ...
}
```

文章部分内容参考：阮一峰老师的《ECMAScript 6 入门》一书

# ES6系列之箭头函数全解析

## 引言

ES6中允许使用箭头=>来定义箭头函数，是ES6中较受欢迎也较常使用的新增特性。本文将从箭头函数的基本语法，与普通函数对比，箭头函数不适用场景三个方面进行梳理。

## 基本语法

```javascript
// 箭头函数
let func = (name) => {
    // 函数体
    return `Hello ${name}`;
};

// 等同于
let func = function (name) {
    // 函数体
    return `Hello ${name}`;
};   
```
从上面可以看出，定义箭头函数语法上要比普通函数简洁得多。箭头函数省去了`function`关键字，采用箭头`=>`来定义函数。函数的参数放在`=>`前面的括号中，函数体跟在`=>`后的花括号中，箭头函数在参数和箭头之间不能换行。


### 箭头函数的参数

1. 如果箭头函数没有参数，直接写一个空括号即可。
2. 如果箭头函数的参数只有一个，可以省略包裹参数的括号。
3. 如果箭头函数有多个参数，将参数依次用逗号分隔，参数必须被包裹在括号中。

### 箭头函数的函数体

如果箭头函数的函数体只有一句代码，即返回某个变量或者返回一个简单的JS表达式，可以省去函数体的大括号{ }。

```javascript
let func = val => val;
// 等同于
let func = function (val) { return val };

let sum = (num1, num2) => num1 + num2;
// 等同于
let sum = function(num1, num2) {
  return num1 + num2;
};

let mulFunction = (num1, num2 ,num3) => num1 * num2 * num3;
// 等同于
let mulFunction = function(num1, num2 ,num3) {
    return num1 * num2 * num3;
}
```

### 箭头函数返回一个对象

如果箭头函数的函数体只有一句代码且返回一个对象（对象字面量）时，直接写一个表达式是不行的。

```javascript
let func = () => { foo: 1 }; 
console.log(func()); // 执行后返回undefined

// 如果是这样还会直接报错
let func = () => { foo: 1, bar: 2 };
```
原因是花括号被解释为函数体的大括号，解决办法：**用圆括号把对象字面量包起来**
```javascript
let func = () => ({ foo: 1 });
console.log(func()); // {foo: 1}

// 不过上面那样解决的缺点是可读性变差了，所以更推荐直接当成多条语句的形式来写，可读性高  
let func = () => {
    return {
        foo: 1
    }
}
```

### 简化回调函数

这是箭头函数比较常见的用法
```javascript
// 普通函数写法
[1, 2, 3, 4].map(function (x) {
    return x * x;
});

let result = [5, 4, 1, 3, 2].sort(function (a, b) {
    return a - b;
});

// 箭头函数写法
[1, 2, 3, 4].map(x => x * x);

let result = [5, 4, 1, 3, 2].sort((a, b) => a - b);
```
## 跟普通函数的区别

### 1.没有this绑定

箭头函数没有自己的this，它会捕获自己在*定义时*）所处的外层执行环境的this，并继承这个this值。所以，箭头函数中this的指向在它被定义的时候就已经确定了，之后永远不会改变。

```javascript
const obj = {
	a: function() { console.log(this) }    
}
obj.a();  // 打印结果：obj对象

const obj = {
	a:() => {
        console.log(this);
    }    
}
obj.a();  // 打印结果： Window对象
```
上述代码中，箭头函数与外层的this保持一致，最外层的this就是Window对象。

### 2.没有arguments

```javascript
function func1(a, b) {
    console.log(arguments);
}
let func2 = (a, b) => {
    console.log(arguments);
}
func1(1, 2); // Arguments(2) [1, 2, callee: ƒ, Symbol(Symbol.iterator): ƒ]
func2(1, 2); // Uncaught ReferenceError: arguments is not defined
```
如果非要打印函数参数，可以在箭头函数中使用rest参数代替arguments对象
```javascript
let func2 = (...rest) => {
    console.log(rest); // (2) [1, 2]
}
```

### 3.不能通过 new 关键字调用

在构造函数中，this指向新创建的对象实例

而箭头函数没有 [[Construct]]方法，箭头函数不可以当作构造函数，如果这样做会抛出异常
```javascript
var Person = (name) => {
    this.name = name;
}

// Uncaught TypeError: Person is not a constructor
var person = new Person('jacky'); 
```
箭头函数在创建时this对象就绑定了，故不会指向对象实例。

### 4.没有 new.target

new.target是ES6新引入的属性，普通函数如果通过new调用，new.target会返回该函数的引用。
```javascript
function Cat() {
    console.log(new.target); 
}
let cat = new Cat(); // ƒ Cat() { console.log(new.target); }
```
此属性主要：用于确定构造函数是否为new调用的。 

箭头函数的this指向全局对象，在箭头函数中使用箭头函数会报错。
```javascript
// 普通函数
let a = function() {
    console.log(new.target);
}
a(); // undefined

// 箭头函数
let b = () => {
    console.log(new.target); // 报错：Uncaught SyntaxError: new.target expression is not allowed here
};
b();
```

### 5.没有原型

由于不能通过 new 关键字调用，不能作为构造函数，所以箭头函数不存在 prototype 这个属性。
```javascript
let func = () => {};
console.log(func.prototype) // undefined
```

### 6.没有 super

箭头函数没有原型，故也不能通过 super 来访问原型的属性，所以箭头函数也是没有 super 的。同this、arguments、new.target 一样，这些值由外围最近一层非箭头函数决定。

### 7.call/apply/bind方法无法改变箭头函数中this的指向

call()、apply()、bind()方法的共同特点是可以改变this的指向，用来动态修改函数执行时this的指向。但由于箭头函数的this定义时就已经确定了且不会改变。所以这三个方法永远也改变不了箭头函数this的指向。

```javascript
var name = 'global name';
var obj = {
    name: 'jacky'
}
// 箭头函数定义在全局作用域
let func = () => {
    console.log(this.name);
};

func();     // global name
// this的指向不会改变，永远指向Window对象,放到到window下的全局变量
func.call(obj);     // global name
func.apply(obj);    // global name
func.bind(obj)();   // global name
```

### 8.箭头函数的解析顺序相对靠前

虽然箭头函数中的箭头不是运算符，但箭头函数具有与常规函数不同的特殊运算符优先级解析规则。
```javascript
let callback;

callback = callback || function() {}; // ok

callback = callback || () => {};      
// SyntaxError:非法箭头函数属性

callback = callback || (() => {});    // ok
```

### 9.箭头函数不支持重名参数

```javascript
function foo(a, a) {
    console.log(a, arguments); // 2 Arguments(2) [1, 2, callee: ƒ, Symbol(Symbol.iterator): ƒ]
}

var boo = (a, a) => { // 直接报错：Uncaught SyntaxError: Duplicate parameter name not allowed in this context
    console.log(a);
};
foo(1, 2);
boo(1, 2);
```
### 10.使用 yield 关键字

yield 关键字通常不能在箭头函数中使用（除非是嵌套在允许使用的函数内）。因此，箭头函数不能用作生成器（ Generator ）。

## 箭头函数不适用的场景

**1.不应被用在定义对象的方法上**

```javascript
var obj = {
  x: 10,
  b: function() {
    console.log( this.x, this)
  },
  c: () => console.log(this.x, this)
}
obj.b(); // 10  {x: 10, b: ƒ, c: ƒ}

obj.c(); // undefined Window
```
因为它内部this的指向原因，当使用obj.c()的时候，我们希望c方法里面的this指向obj，但是它却指向了obj所在上下文中的this（即window），违背了我们的需求，所以箭头函数不适合作为对象的方法。

**2.具有动态上下文的回调函数，也不应使用箭头函数**

```javascript
var btn = document.getElementById('btn');
btn.addEventListener('click', () => {
  console.log(this);
});
为btn的监听函数是一个箭头函数，导致里面的this就是全局对象,而不符合我们想操作按钮本身的需求。如果改成普通函数，this就会动态指向被点击的按钮对象
```

除了前面两点，剩下的跟上面讲的与普通函数的区别重复了，故只作总结不贴代码了:

1. 不应被用在定义对象的方法上
2. 具有动态上下文的回调函数，也不应使用箭头函数
3. 不能应用在构造函数中
4. 避免在 prototype 上使用
5. 避免在需要 arguments 上使用

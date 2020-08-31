# 深入理解 JavaScript 之执行上下文和变量对象

继续接着上篇文章，上篇我们说到函数上下文的结构可表示为

```javascript
const ExecutionContextObj = {
  VO: window, // 变量对象
  ScopeChain: {}, // 作用域链
  this: window,
}
```

即每个函数上下文，都要有这三个重要属性：

- 变量对象(Variable object, VO)
- 作用域链(Scope chain)
- this

今天再细说执行上下文中的变量对象

## 变量对象

### 1.什么是变量对象

变量对象是与执行上下文的相关的数据作用域，存储了在上下文中定义的变量和函数声明。因为不同执行上下文的变量对象略有不同，所以变量对象一般分为全局上下文下的变量对象和函数上下文下的变量对象。

### 2.变量对象(VO)的创建过程

变量对象的创建，属于执行上下文中的创建阶段，依次经过以下三个过程：

创建执行上下文有两个阶段：一个是创建阶段，一个是执行阶段。变量对象的创建，属于执行上下文中的创建阶段，会依次经过三个过程：

#### 1.为函数的形参赋值（函数上下文）

在进入函数执行上下文时，会首先检查实参个数，接着对实参对象和形参进行赋值。如果没有实参，属性值设为 undefined；当传入的实参数量小于形参数量，则会将没有被赋值的形参赋值为 undefined。

```javascript
function fn(a, b, c) {
  console.log(a, b, c) // 1 2 undefined
}
fn(1, 2)
```

此时变量对象的结构为:

```javascript
VO = {
  a: 1,
  b: 2,
  c: undefined,
}
```

#### 2.函数声明

遇到同名的函数时，后面函数会覆盖前面的函数。

```javascript
function fn() {
  console.log('先声明的')
}
function fn() {
  console.log('后声明的')
}

console.log(fn) //ƒ fn() { console.log('后声明的'); }
```

#### 3.变量声明

检查当前环境中通过变量声明(var)并赋值为 undefined（变量提升产生的原因）

```javascript
console.log(fn) // ƒ fn() { console.log('后声明的');}
console.log(b) // undefined
function fn() {
  console.log('先声明的')
}
function fn() {
  console.log('后声明的')
}
var b = 10
var fn = 20

console.log(b) // 10
```

由上面我们看出，当变量名称与函数名称同名时，会忽略此变量声明，即同名时，函数声明优先

js 虽然单线程的语言，执行顺序为顺序执行，但 JS 引擎并不是一行一行地分析和执行程序，而是一段一段地分析执行。

让我们从 JS 引擎的角度理一理上述三个过程：**函数提升和变量提升，是全局执行上下文做的准备工作；当执行函数时，又会创建一个执行上下文，做的是这个函数内部的准备工作；这就是 JS 引擎分析代码的"预编译阶段"，做完该工作才进入执行阶段。**

### 3.全局上下文的变量对象

在客户端 JavaScript 中，全局对象就是 Window 对象;

```javascript
var a = 2 //在全局上下文使用var定义变量a，作为全局变量的宿主
console.log(this) // window对象
console.log(this.a) // 2
```

对于全局上下文中来说，变量对象就是全局对象，

### 4.变量对象变为活动对象

执行上下文的第二个阶段为执行阶段，此时会进行变量赋值，执行其他代码等工作，此时，变量对象变为活动对象(Active object, AO)。

> 活动对象和变量对象其实是同个东西，只是规范概念上的差异。只有到当进入一个执行上下文中，这个执行上下文的变量对象才会被激活，所以才叫活动对象。而只有被激活的变量对象，也就是活动对象上的各种属性才能被访问。

所以明确，活动对象是在进入函数上下文时刻被创建的，它通过函数的 arguments 属性初始化。

```javascript
console.log(fn) // ƒ fn() { console.log('后声明的');}
console.log(b) // undefined
function fn() {
  console.log('先声明的')
}
function fn() {
  console.log('后声明的')
}
var b = 10
console.log(b) // 10
var fn = 20
console.log(fn) //20
```

上述代码，真正开始执行是从第一行 console.log(fn)。在此之前，变量对象 VO 是这样的：

```javascript
// 创建过程
EC= {
  VO：{}, // 创建变量对象
  scopeChain: [{VO}], // 作用域链
  this: window // this绑定
}
VO = {
  // argument: {}, // 当前为全局上下文，不存在arguments
  fn: reference to function fn(){}, // 函数fn的引用地址
  b: undefiend  // 变量提升
}
```

根据变量对象创建的三个过程，

1. 首先是 arguments 对象的创建（全局上下文没有则忽略）
2. 其次，是检查函数的声明。此时，函数 fn 声明了两次，则后一次的声明会覆盖上一次的声明。
3. 最后，是检查变量的声明，先声明了变量 b，将它赋值为 undefined；接着遇到 fn 的变量声明，由于 fn 已经被声明为一个函数，故忽略该变量声明。

到此，变量对象的创建阶段完成，接下来进行执行阶段：

```
1.执行console.log(fn);此时fn为声明的第二个函数，故输出结果："后声明的"。
2.执行console.log(b)，此时b已被赋值为undefined，故输出结果："undefined"。
3.执行赋值操作： b = 10;
4.执行console.log(b) ，故输出b为10。
5.执行赋值操作： fn = 20;
6.执行console.log(fn) ，故输出fn为20。
```

执行到最后一步时，执行上下文如下：

```javascript
// 执行阶段
EC = {
  VO = {};
  scopeChain: {};
  this: window;
}
 // VO ---- AO
AO = {
  argument: {};
  fn: 20;
  b: 10;
}
```

以上，就是变量对象在代码执行前及执行后的变化。

### 总结

1. 全局上下文的变量对象初始化是全局对象

2. 函数上下文的变量对象初始化只包括 arguments 对象

3. 在进入执行上下文时会给变量对象添加形参、函数声明、变量声明等初始的属性值

4. 在代码执行阶段，会再次修改变量对象的属性值

# 深入理解JavaScript之作用域链与闭包

## 作用域

作用域是指程序源代码中定义变量的区域。

实际上描述的就是查找变量的范围，作用域必须有的两个功能就是**存储变量**以及**查找变量**，作用域就是发挥这两个作用以及更多作用的规则。

作用域规定了如何查找变量，也就是确定当前执行代码对变量的访问权限。

### 词法作用域和动态作用域

- 词法作用域：（静态作用域）函数的作用域在函数定义的时候就决定了。
- 动态作用域：函数的作用域是在函数调用的时候才决定的。

JavaScript采用的是词法作用域

### 全局作用域

在代码中任何地方都能访问到的对象拥有全局作用域

全局变量：
> 1. 生命周期将存在于整个程序之内。
> 2. 能被程序中任何函数或者方法访问。
> 3. 在 JavaScript 内默认是可以被修改的。

JavaScript 采用词法作用域(lexical scoping)，也就是静态作用域。

#### 显式声明

带有关键字 var 的声明

```javascript
var winValue = 10;
console.log(window.winValue); // 10
```
 
#### 隐式声明

不带有声明关键字的变量，JS 会默认帮你声明一个全局变量
```javascript
function foo(value) {
    result = value + 1;	 // 没有用 var 声明
    return result;
};
foo(1);	
console.log(window.result);	// 2 <=  挂在了 window全局对象上 
```

### 函数作用域

函数作用域内，对外是封闭的，从外层的作用域无法直接访问函数内部的作用域。

在函数内部的变量权限称为函数作用域，有以下特点：
1. 每个函数都有自己的作用域，而且调用一次就会生成新的作用域
2. 只能在函数内部才能访问，外部是没有权限访问的
3. 进入函数内部时开启，函数执行完毕后销毁

```javascript
function bar() {
    var foo = 'test';
}

console.log(foo); // Uncaught ReferenceError: foo is not defined
```

### 块级作用域（ES6新增）

凡是由{}符号包裹起来的都是块作用域
```javascript  
for(let i = 0; i < 5; i++) {
    // ...
}
console.log(i); // Uncaught ReferenceError: i is not defined
```
在 for 循环执行完毕之后 i 变量就被释放了

## 作用域链

作用域链：当访问一个变量时，解释器会首先在当前作用域查找，如果没有找到，就去父作用域找，直到找到该变量或者不在父作用域中，这就是作用域链。

```javascript
var a = 1

function foo () {
    var b = 2
    console.log(a)
}

foo() // 1
console.log(b) // Uncaught ReferenceError: b is not defined
```
从上面代码的执行结果可以看出，foo 函数取到了它外部的变量 a, 而最外层的 console.log(b) 操作并没能取得 foo 函数里面的变量 b。

- 作用域链和原型继承查找时的区别：

如果去查找一个普通对象的属性，但是在当前对象和其原型中都找不到时，会返回undefined；但查找的属性在作用域链中不存在的话就会抛出ReferenceError。

## 闭包

### 定义

闭包是指**有权访问另外一个函数作用域中的变量的函数**

在javascript中，只有函数内部的子函数才能读取局部变量，所以闭包可以理解成"定义在一个函数内部的函数"。在本质上，闭包是将函数内部和函数外部连接起来的桥梁。

- 闭包由两部分构成：
1. 函数
2. 能访问另外一个函数作用域中的变量

### 为什么有闭包?

> 之所以出现闭包是因为JS的垃圾回收机制，JS本身为了避免解释器过量消耗内存，造成系统崩溃，自带有一套垃圾回收机制，垃圾回收机制能够检测到一个对象是不是无用的。检测到之后，就会把它占用的内存释放掉。但是实际工作中，我们也会需要一些变量不那么及时的被清理，所以就出现了闭包，用来达成这个效果。

### 闭包的特点

- 闭包可以访问当前函数以外的变量
- 即使外部函数已经返回，闭包仍能访问外部函数定义的变量
- 参数和变量不会被垃圾回收机制收回。

```javascript
function getOuter(){
    var name = 'jacky';
    function getName(str){
        console.log(str + name);  // 可以访问getName函数外部的name
    }
    return getName('名字是：'); 
}
getOuter(); // 名字是：jacky
```
再来看下面一段代码：
```javascript
function bar() {
    var x = 1;
    return function () {
        var y = 2;
        return x + y;
    }
}
var foo = bar(); // 这一句执行完，变量x并没有被回收，因为要内部函数还需要引用
console.log(foo()) // 3  执行内部函数，引用外部变量x
```
上面代码中，在全局执行上下文中定义了一个函数bar和变量foo，函数bar内部返回一个匿名函数，所以此刻匿名函数的作用域链初始化为包含了全局变量对象和bar中的变量对象。

当执行var foo = bar()时，把函数bar的执行上下文压入栈，当bar执行完后，其执行上下文应该弹出栈，但是因为bar内部的匿名函数作用域链还引用这bar函数内的变量x，所以bar的执行上下文得不到释放，这样就形成了闭包。

### 闭包的应用

1. 设计私有的方法和变量（封装，定义模块）。

```javascript
var counter = (function(){
    var privateCounter = 0; //私有变量
    function change(val){
        privateCounter += val;
    }
    return {
        increment:function(){  
            change(1);
        },
        decrement:function(){
            change(-1);
        },
        value:function(){
            return privateCounter;
        }
    };
})();
```
2. 匿名函数最大的用途是创建闭包。减少全局变量的使用。从而使用闭包模块化代码，减少全局变量的污染。

```javascript
var objEvent = objEvent || {};
(function() {
    var addEvent = function() {
        // some code
    }
    function removeEvent() {
        // some code
    }
    objEvent.addEvent = addEvent
    objEvent.removeEvent = removeEvent
})()
```
addEvent 和 removeEvent 都是局部变量，但我们可以通过全局变量 objEvent 使用它

### 如何销毁闭包

javascript中，如果一个对象不再被引用，那么这个对象就会被垃圾回收机制回收。如果两个对象互相引用，而不再被第3者所引用，那么这两个互相引用的对象也会被回收。

即释放对闭包的引用，使引用变量为null。

### 闭包的优缺点

优点：

1. 闭包里的变量不会污染全局，因为变量被封在闭包里；
2. 所有变量都在闭包里保证了隐私性和私有性；
3. 可以让这些局部变量保存在内存中，实现变量数据共享。

缺点：

形成闭包即要把一个函数当成值传递，而且该函数还引用这另一个函数的作用域链使得被引用的函数不能被回收，使用不当容易造成内存泄漏；

## 作用域

作用域是指程序源代码中定义变量的区域。

实际上描述的就是查找变量的范围，作用域必须有的两个功能就是**存储变量**以及**查找变量**，作用域就是发挥这两个作用以及更多作用的规则。

作用域规定了如何查找变量，也就是确定当前执行代码对变量的访问权限。

### 词法作用域和动态作用域

- 词法作用域：（静态作用域）函数的作用域在函数定义的时候就决定了。
- 动态作用域：函数的作用域是在函数调用的时候才决定的。

JavaScript采用的是词法作用域

### 全局作用域

在代码中任何地方都能访问到的对象拥有全局作用域

全局变量：
> 1. 生命周期将存在于整个程序之内。
> 2. 能被程序中任何函数或者方法访问。
> 3. 在 JavaScript 内默认是可以被修改的。

JavaScript 采用词法作用域(lexical scoping)，也就是静态作用域。

#### 显式声明

带有关键字 var 的声明

```javascript
var winValue = 10;
console.log(window.winValue); // 10
```
 
#### 隐式声明

不带有声明关键字的变量，JS 会默认帮你声明一个全局变量
```javascript
function foo(value) {
    result = value + 1;	 // 没有用 var 声明
    return result;
};
foo(1);	
console.log(window.result);	// 2 <=  挂在了 window全局对象上 
```

### 函数作用域

函数作用域内，对外是封闭的，从外层的作用域无法直接访问函数内部的作用域。

在函数内部的变量权限称为函数作用域，有以下特点：
1. 每个函数都有自己的作用域，而且调用一次就会生成新的作用域
2. 只能在函数内部才能访问，外部是没有权限访问的
3. 进入函数内部时开启，函数执行完毕后销毁

```javascript
function bar() {
    var foo = 'test';
}

console.log(foo); // Uncaught ReferenceError: foo is not defined
```

### 块级作用域（ES6新增）

凡是由{}符号包裹起来的都是块作用域
```javascript  
for(let i = 0; i < 5; i++) {
    // ...
}
console.log(i); // Uncaught ReferenceError: i is not defined
```
在 for 循环执行完毕之后 i 变量就被释放了

## 作用域链

作用域链：当访问一个变量时，解释器会首先在当前作用域查找，如果没有找到，就去父作用域找，直到找到该变量或者不在父作用域中，这就是作用域链。

```javascript
var a = 1

function foo () {
    var b = 2
    console.log(a)
}

foo() // 1
console.log(b) // Uncaught ReferenceError: b is not defined
```
从上面代码的执行结果可以看出，foo 函数取到了它外部的变量 a, 而最外层的 console.log(b) 操作并没能取得 foo 函数里面的变量 b。

- 作用域链和原型继承查找时的区别：

如果去查找一个普通对象的属性，但是在当前对象和其原型中都找不到时，会返回undefined；但查找的属性在作用域链中不存在的话就会抛出ReferenceError。

## 闭包

### 定义

闭包是指**有权访问另外一个函数作用域中的变量的函数**

在javascript中，只有函数内部的子函数才能读取局部变量，所以闭包可以理解成"定义在一个函数内部的函数"。在本质上，闭包是将函数内部和函数外部连接起来的桥梁。

- 闭包由两部分构成：
1. 函数
2. 能访问另外一个函数作用域中的变量

### 为什么有闭包?

> 之所以出现闭包是因为JS的垃圾回收机制，JS本身为了避免解释器过量消耗内存，造成系统崩溃，自带有一套垃圾回收机制，垃圾回收机制能够检测到一个对象是不是无用的。检测到之后，就会把它占用的内存释放掉。但是实际工作中，我们也会需要一些变量不那么及时的被清理，所以就出现了闭包，用来达成这个效果。

### 闭包的特点

- 闭包可以访问当前函数以外的变量
- 即使外部函数已经返回，闭包仍能访问外部函数定义的变量
- 参数和变量不会被垃圾回收机制收回。

```javascript
function getOuter(){
    var name = 'jacky';
    function getName(str){
        console.log(str + name);  // 可以访问getName函数外部的name
    }
    return getName('名字是：'); 
}
getOuter(); // 名字是：jacky
```
再来看下面一段代码：
```javascript
function bar() {
    var x = 1;
    return function () {
        var y = 2;
        return x + y;
    }
}
var foo = bar(); // 这一句执行完，变量x并没有被回收，因为要内部函数还需要引用
console.log(foo()) // 3  执行内部函数，引用外部变量x
```
上面代码中，在全局执行上下文中定义了一个函数bar和变量foo，函数bar内部返回一个匿名函数，所以此刻匿名函数的作用域链初始化为包含了全局变量对象和bar中的变量对象。

当执行var foo = bar()时，把函数bar的执行上下文压入栈，当bar执行完后，其执行上下文应该弹出栈，但是因为bar内部的匿名函数作用域链还引用这bar函数内的变量x，所以bar的执行上下文得不到释放，这样就形成了闭包。

### 闭包的应用

1. 设计私有的方法和变量（封装，定义模块）。

```javascript
var counter = (function(){
    var privateCounter = 0; //私有变量
    function change(val){
        privateCounter += val;
    }
    return {
        increment:function(){  
            change(1);
        },
        decrement:function(){
            change(-1);
        },
        value:function(){
            return privateCounter;
        }
    };
})();
```
2. 匿名函数最大的用途是创建闭包。减少全局变量的使用。从而使用闭包模块化代码，减少全局变量的污染。

```javascript
var objEvent = objEvent || {};
(function() {
    var addEvent = function() {
        // some code
    }
    function removeEvent() {
        // some code
    }
    objEvent.addEvent = addEvent
    objEvent.removeEvent = removeEvent
})()
```
addEvent 和 removeEvent 都是局部变量，但我们可以通过全局变量 objEvent 使用它

### 如何销毁闭包

javascript中，如果一个对象不再被引用，那么这个对象就会被垃圾回收机制回收。如果两个对象互相引用，而不再被第3者所引用，那么这两个互相引用的对象也会被回收。

即释放对闭包的引用，使引用变量为null。

### 闭包的优缺点

优点：

1. 闭包里的变量不会污染全局，因为变量被封在闭包里；
2. 所有变量都在闭包里保证了隐私性和私有性；
3. 可以让这些局部变量保存在内存中，实现变量数据共享。

缺点：

形成闭包即要把一个函数当成值传递，而且该函数还引用这另一个函数的作用域链使得被引用的函数不能被回收，使用不当容易造成内存泄漏；


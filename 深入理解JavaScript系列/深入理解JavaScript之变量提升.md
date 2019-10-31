# 变量提升

> 原理：JS引擎的工作方式是先解析代码，获取所有被声明的变量；然后在运行。JS代码自上而下执行之前，浏览器首先会把所有带 “VAR”/“FUNCTION” 关键词的进行提前 “声明” 或者 “定义” ，这种预先处理机制称之为 “变量提升”。
```javascript
console.log(a, b);//undefined undefined
var a = 12,
    b = 12;

function fn() {
    console.log(a, b);//=>undefined 12
    var a = b = 13;
    console.log(a, b);//=>13 13
}

fn();
console.log(a, b);//=>12 13
```

 - undefined undefined：首先输出这个结果是因为变量提升，即前三行变成
```javascript
var a;
var b;
console.log(a,b);//undefined undefined
a = 12;
b = 12;
```
 - undefined 12：接下来执行函数fn（fn一开始也被执行了变量提升,只不过函数中存储的都是字符串而已），对fn内部进行分析，即内部代码变成：

```javascript
var a;
console.log(a,b);//undefined 12
a = 13;
b = 13; //不加var的本质是WIN的属性，即相当于window.b = 13; 
console.log(a,b);//13 13
```
一开始b在fn内部找不到，便会开始往上一层找，找到了全局的b，于是b输出12。
当经过var a = b = 13; 后，b被赋值为13，于是输出先从函数内部找b，找到了b=13，第二次输出b为13。（！同时，最先定义的全局变量b = 12也被赋值为13，故最后的b也等于13）

> 私有作用域中带var和不带var的区别：
1.带var的在私有作用于变量提升阶段，都声明为私有变量，和外界没有任何的关系
2.不带var不是私有变量，会向它的上级作用于查找，一直找到window为止（这种查找机制叫做：`作用域链`），也就是在私有作用域中操作的这个非私有变量，是一直操作别人的

## 只对等号左边进行变量提升

```javascript
sum();
fn();// Uncaught TypeError: fn is not a function 

//=>匿名函数之函数表达式
var fn = function(){
    console.log('fn');
}//=>代码执行到此处会把函数值赋值给fn

//=>普通的函数
function sum(){
    console.log('sum');
}
```

## 条件判断下的变量提升

```javascript
/*
 * 在当前作用域下，不管条件是否成立都要进行变量提升
 *   =>带VAR的还是只声明
 *   =>带FUNCTION的在老版本浏览器渲染机制下，声明和定义都处理，但是为了迎合ES6中的块级作用 
 *     域，新版浏览器对于函数（在条件判断中的函数），
 *     不管条件是否成立，都只是先声明，没有定义，类似于var
 */
console.log(a);//undefined
   if(1 === 2){
   var a = 3;
   }
   console.log(a);//undefined
```

## 重名问题的处理

```javascript
fn();//=>4
function fn() {console.log(1);}
fn();//=>4
function fn() {console.log(2);}
fn();//=>4
var fn=100;//=>带VAR的在提升阶段只把声明处理了,赋值操作没有处理,所以在代码执行的时候需要完成赋值 FN=100
fn();//=>100() Uncaught TypeError: fn is not a function
function fn() {console.log(3);}
fn();
function fn() {console.log(4);}
fn();
```
带VAR和FUNCTION关键字声明相同的名字，这种也算是重名了（其实是一个FN，只是存储值的类型不一样）
关于重名的处理：如果名字重复了，不会重新的声明，但是会重新的定义（重新赋值）[不管是变量提升还是代码执行阶段皆是如此]
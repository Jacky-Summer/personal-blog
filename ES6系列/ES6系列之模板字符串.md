# ES6系列之模板字符串

模板字符串是ES6中非常重要的一个新特性，这个特性使得处理相关业务变得更加容易。

## 基础用法

```javascript
let a = `hello world`;
console.log(a); // hello world
```
注意这里不是双引号，而是反撇号`

在模板字符串中，还可以拼接html元素，同时空格、缩进、换行都会被保留，并且如果模板字符串中的变量没有声明，将报错。
```javascript
let str = `
    <div>
        <ul>
            <li>11</li>
            <li>22</li>
        </ul>
    </div>
`;
```
![](https://user-gold-cdn.xitu.io/2020/1/5/16f75f81fdc039fd?w=283&h=118&f=png&s=3077)

上面代码中，`<div>`标签前面会有一个换行。如果你不想要这个换行，可以使用trim方法消除它。
```javascript
let str = `
    <div>
        <ul>
            <li>11</li>
            <li>22</li>
        </ul>
    </div>
`.trim();
console.log(str);
```
![](https://user-gold-cdn.xitu.io/2020/1/5/16f75fa6f941fbc2?w=285&h=115&f=png&s=3098)

在模版字符串内使用反引号`时，需要在它前面加转义符\
```javascript
let a = `hello \\n`;
console.log(a);
```

## 嵌入变量

```javascript
let name = 'jacky';
let str = '我叫' + name + '，大家好';
console.log(str); // 我叫jacky，大家好
```
当有变量参与拼接时，ES5下必须用+号这样的形式进行拼接，这样很麻烦而且很容易出错。
ES6新增了字符串模版，可以很好的解决这个问题。这时再引用str变量就需要用${name}这种形式了。
```javascript
let name = 'jacky';
let str = '我叫${name}，大家好';
console.log(str); // 我叫jacky，大家好
```
如果大括号内部是一个字符串，将会原样输出：
```javascript
let world = '666';
let a = `hello ${'world'}`;
console.log(a); // hello world
```

## 对运算的支持

在${}里面，可以写任意的JS表达式，比如我们用它进行运算
```javascript
let a = 1;
let b = 2;
let c = `${a+b}`;
console.log(c); //3

let obj = { x: 1, y: 2 };
console.log(`${obj.x + obj.y}`); // 3
```
## 模板字符串调用函数
```javascript
function fn() {
    return "Hello World";
}
console.log(`foo ${fn()} bar`); // foo Hello World bar
```
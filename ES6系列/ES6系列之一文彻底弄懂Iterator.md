# ES6系列之一文彻底弄懂Iterator

## 前言

本文主要来深入剖析ES6的Iterator（迭代器/遍历器），在了解它之前，我们首先要知道为什么需要Iterator? 它出现的原因是什么?

## 从循环说起

平时开发中，我们经常会用到循环，拿最基本的来说，比如循环一个数组:
```javascript
// for循环
var arr = [1, 2, 3, 4];
for(let i = 0; i < arr.length; i++) {
    console.log(arr[i]); // 1 2 3 4
}

// 也可以用forEach循环
arr.forEach(item => {
    console.log(item); // 1 2 3 4
})

// 也可以用for in循环
for(let i in arr) {
    console.log(arr[i]); // 1 2 3 4
}
```
如果是循环输出字符串的每一个字符
```javascript
// for循环
var str = 'abcde';
for(var i = 0; i < str.length; i++) {
    console.log(str[i]); // a b c d e
}

// for in循环
for(var k in str){
    console.log(str[k]); // a b c d e
}

// forEach不能遍历字符串，如果硬要用则只能先把字符串转为数组再用数组方式循环
```


如果是循环输出一个map对象呢, 妥妥的就一个foreach循环，普通的for循环和for...in都不行
```javascript
var map = new Map();
map.set('first', '第一');
map.set('second', '第二');
map.set('third', '第三');

// forEach循环
map.forEach((val,key) => {
    console.log(val, key);
    // 第一 first
    // 第二 second
    // 第三 third
});
```

对于上述各集合数据，我们发现没有一个循环方法是可以一次性解决的。

虽然forEach循环不能循环字符串，但字符串可以转为数组再使用forEach即可输出，但这操作并不舒服每次使用都要转换。而且forEach循环存在缺点：`不能使用break，continue语句跳出循环，或者使用return从函数体返回`。

而for循环在有些情况写代码会增加复杂度，而且不能循环对象。

相比下，for...in的缺点是不仅遍历数字键名，还会遍历手动添加的自定义键，甚至包括原型链上的键。for...in主要还是为遍历对象而设计的，并不太适用于遍历数组。

如下代码
```javascript
Array.prototype.protoValue = 'hello';
var arr = [1, 2, 3, 4];
arr.test = 'test';
for(let i in arr) {
    console.log(arr[i]); // 1 2 3 4 test hello
}
```

话说回来，有没有一种更好的循环能一统上述循环问题？ ES6就有了，用for...of循环

## for...of循环出现

它的出现解决了什么问题呢？首先是当然是能解决了上述我们的问题

对上述数据进行循环输出
```javascript
for(let v of arr) {
    console.log(v); // 1 2 3 4
}
for(let v of str) {
    console.log(v); // a b c d e
}
for(let v of map) {
    console.log(v);
    // (2) ["first", "第一"]
    // (2) ["second", "第二"]
    // (2) ["third", "第三"]
}
```
来看看它的优点：

1. 简洁直接的遍历数组语法
2. 它避开了 for-in 循环的所有缺点
3. 与forEach循环不同的是，它可以使用break、continue 和 return 语句



阮一峰老师的《ECMAScript 6 入门》提到：
> ES6中引入了for...of循环，作为遍历所有数据结构的统一的方法。

> Iterator是一种接口，为各种不同的数据结构提供统一的访问机制。任何数据结构只要部署 Iterator 接口，就可以完成遍历操作（即依次处理该数据结构的所有成员）。

一个数据结构只要具有 iterator 接口，就可以用for...of循环遍历它的成员。

也就是说，并不是所有的对象都能使用for...of循环，只有实现了Iterator接口的对象，才能够for...of来进行遍历取值。

那么，Iterator是什么呢？我们接着说

## Iterator迭代器

### 概念

> 它是一种接口，为各种不同的数据结构提供统一的访问机制。任何数据结构只要部署 Iterator 接口，就可以完成遍历操作（即依次处理该数据结构的所有成员）。

### 作用

1. 为各种数据结构，提供一个统一的、简便的访问接口；
2. 使得数据结构的成员能够按某种次序排列；
3. 创造一种新的遍历命令for...of循环，Iterator 接口主要供for...of消费。

现在明白了，Iterator的产生主要是为了使用for...of方法。但具体Iterator概念还是有些抽象，如果要直接具体的描述的话：

Iterator其实就是一个具有 next()方法的对象，并且每次调用next()方法都会返回一个结果对象，这个结果对象有两个属性, 如下
```
{
    value: 表示当前的值,
    done: 表示遍历是否结束
}
```

这里多了一些概念，我们来梳理一下：

Iterator是一个特殊的对象：

1. 它具有next()方法，调用该方法就会返回一个结果对象
2. 结果对象有两个属性值：`value`和`done`。
3. `value`表示具体的返回值；done是布尔类型，表示集合是否完成遍历，没有则返回true，否则返回false
4. 内部有一个指针，指向数据结构的起始位置。每调用一次next()方法，指针都会向后移动一个位置，直到指向最后一个位置。

### 模拟Iterator

根据上述描述，我们来模拟一个迭代器，代码如下
```javascript
function createIterator(items) {
    var i = 0;
    return {
        next: function() {
            var done = (i >= item.length);
            var value = !done ? items[i++] : undefined;

            return {
                done: done,
                value: value
            };
        }
    };
}

// iterator 就是一个迭代器对象
var iterator = createIterator([1, 2, 3]);

console.log(iterator.next()); // { done: false, value: 1 }
console.log(iterator.next()); // { done: false, value: 2 }
console.log(iterator.next()); // { done: false, value: 3 }
console.log(iterator.next()); // { done: true, value: undefined }
console.log(iterator.next()); // { done: true, value: undefined }
console.log(iterator.next()); // { done: true, value: undefined }
```

过程:

1. 创建一个指针对象，指向当前数据结构的起始位置。也就是说，遍历器对象本质上，就是一个指针对象。
2. 第一次调用指针对象的next方法，next 方法内部通过闭包来保存指针`i` 的值，每次调用`i`都会`+1`，指向下一个。故第一次指针指向数据结构的第一个成员，输出1。
3. 第二次调用指针对象的next方法，指针就指向数据结构的第二个成员，输出2。
4. 第三次调用指针对象的next方法，数组长度为3，此时数据结构的结束位置，输出3。
5. 第四次调用指针对象的next方法，此时已遍历完成了，输出done为true表示完成，value为undefined，此后第n次调用都是该结果。

看到这里大家大概知道`Iterator`了，但Iterator接口主要供for...of消费。我们试着用for...of循环上面创建Iterator对象
```
var iterator = makeIterator([1, 2, 3]);

for (let value of iterator) {
    console.log(value); // Uncaught TypeError: iterator is not iterable
}
```
结果报错，说明我们生成的 iterator 对象并不是可遍历的，这样的结构还不能够被for...of循环

那什么样的结构才是可遍历的呢？

## 可迭代对象Iterable

### 如何实现

ES6还引入了一个新的Symbol对象，symbol值是唯一的。

一个数据结构只要部署了Symbol.iterator属性，就被视为具有 iterator 接口；调用这个接口，就会返回一个遍历器对象。这样的数据结构才能被称为可迭代对象(Iterator)，该对象可被for...of遍历。

照着上面的规定来创建一个可迭代对象
```javascript
var arr = [1, 2, 3];

arr[Symbol.iterator] = function() {
    var _this = this;
    var i = 0;
    return {
        next: function() {
            var done = (i >= _this.length);
            var value = !done ? _this[i++] : undefined;
            return {
                done: done,
                value: value
            };
        }
    };
}

// 此时可以for...of遍历
for(var item of arr){
    console.log(item); // 1 2 3 
}
```
由此，我们也可以知道 for...of 遍历的其实是对象的 Symbol.iterator 属性。

### 原生具备Iterator接口的数据结构

在ES6中，所有的集合对象，包括数组，类数组对象（arguments对象、DOM NodeList 对象），Map和Set，还有字符串都是可迭代的，都可以被for...of遍历，因为他们都有默认的迭代器。

下面就挑其中几个类型来举例子：

#### 数组

```javascript
var arr = [1, 2, 3];
 
var iteratorObj = arr[Symbol.iterator]();

console.log(iteratorObj.next());
console.log(iteratorObj.next());
console.log(iteratorObj.next());
console.log(iteratorObj.next());
```
输出结果：
![](https://user-gold-cdn.xitu.io/2020/1/8/16f8433c4b0b041a?w=294&h=282&f=png&s=23046)
由上述对迭代器的概念认识，输出结果也完全符合我们预期。

#### arguments

我们知道普通对象是默认没有部署这个接口的，所以arguments这个属性没有在原型上，而是在对象自身的属性上。

```javascript
function test(){
   var obj = arguments[Symbol.iterator]();
   console.log(arguments);
   console.log(obj.next());
   console.log(obj.next());
   console.log(obj.next());
}

test(1, 2, 3);
```
输出结果：

![](https://user-gold-cdn.xitu.io/2020/1/8/16f8446fec7e2102?w=508&h=197&f=png&s=20591)

#### NodeList

```html
<div class="test">1</div>
<div class="test">2</div>
<div class="test">3</div>
```
```javascript
const nodeList = document.getElementsByClassName('test')
for (const node of nodeList) {
    console.log(node);
}
```
输出结果：

![](https://user-gold-cdn.xitu.io/2020/1/8/16f84531078f206d?w=236&h=61&f=png&s=3489)
#### map
这次直接从原型上找来证明
```javascript
console.log(Map.prototype.hasOwnProperty(Symbol.iterator)); // true
```

至于其他类型的可迭代对象大家可举一反三。


## 调用 Iterator 接口的场合

有一些场合会默认调用 Iterator 接口（即Symbol.iterator方法）

### 解构赋值
```javascript
let set = new Set().add('a').add('b').add('c');

let [x, y] = set;
console.log(x, y); // a b
```

### 扩展运算符

扩展运算符（...）也会调用默认的 Iterator 接口，可以将当前迭代对象转换为数组。
```javascript
var str = 'hello';
console.log([...str]); // ["h", "e", "l", "l", "o"]

let arr = ['b', 'c'];
console.log(['a', ...arr, 'd']); // ["a", "b", "c", "d"]
```

### yield*

yield*后面跟的是一个可遍历的结构，它会调用该结构的遍历器接口。
```javascript
let generator = function* () {
  yield 1;
  yield* [2,3,4];
  yield 5;
};

var iterator = generator();

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: 3, done: false }
console.log(iterator.next()); // { value: 4, done: false }
console.log(iterator.next()); // { value: 5, done: false }
console.log(iterator.next()); // { value: undefined, done: true }
```
关于yield之后会再写一篇专题

### 其他场合

例如：for...of、Set()、Map()、Array.from()等。我们主要是为了证明调用上面这些函数时着实是用到了Iterator接口。

我们贴出上面的实现可迭代对象的代码，进行改动。想一想，尽管我们没有手动添加 Symbol.iterator属性，但因为 ES6 默认部署了 Symbol.iterator，数组还是可以遍历成功，那当然我们也可以手动修改这个属性，重新部署也可以。

如果我们手动修改生效，影响了输出，证明某方法调用需要用到Iterator接口
```javascript
var arr = [1, 2, 3];

arr[Symbol.iterator] = function() {
    var _this = this;
    var i = 0;
    return {
        next: function() {
            var done = (i >= _this.length);
            var value = !done ? _this[i++] : undefined;
            return {
                done: done,
                value: value + ' 手动添加的属性' // 添加自定义值
            };
        }
    };
}

// 输出结果显示手动修改生效，证明for...of调用了Iterator接口
for(var item of arr){
    console.log(item);
    // 1 手动添加的属性
    // 2 手动添加的属性
    // 3 手动添加的属性
}

var set = new Set(arr);
console.log(set); // Set(3) {"1 手动添加的属性", "2 手动添加的属性", "3 手动添加的属性"}
```
上述其他方法证明也是如此，就不一一列出了。

## 计算生成的数据结构

有些数据结构是在现有数据结构的基础上，计算生成的。比如，ES6 的数组、Set、Map 都部署了以下三个方法，调用后都返回遍历器对象。

1. entries() 返回一个遍历器对象，用来遍历[键名,键值]组成的数组。对于数组，键名就是索引值。
2. keys() 返回一个遍历器对象，用来遍历所有的键名。
3. values() 返回一个遍历器对象，用来遍历所有的键值。

```javascript
var arr = ["first", "second", "third"];

for (let index of arr.keys()) {
    console.log(index);
}

// 0
// 1
// 2

for (let color of arr.values()) {
    console.log(color);
}

// first
// second
// third

for (let item of arr.entries()) {
    console.log(item);
}

// [ 0, "first" ]
// [ 1, "second" ]
// [ 2, "third" ]
```


## 判断对象是否可迭代

```javascript
const isIterable = obj => obj != null && typeof obj[Symbol.iterator] === 'function';
console.log(isIterable(new Set())); // true 
console.log(isIterable(new Map())); // true  
console.log(isIterable([1, 2, 3])); // true 
console.log(isIterable("hello world")); // true 
```


### for...of循环不支持遍历普通对象 

梳理了半天，回到最初，Iterator的产生主要是为了使用for...of方法。而对象不像数组的值是有序的，遍历的时候根本不知道如何确定他们的先后顺序，所以要注意的是for...of循环不支持遍历对象。

如果非要遍历对象，同理对象也必须包含[Symbol.iterator]的属性并实现迭代器方法，可以通过手动利用Object.defineProperty方法添加该属性。

```javascript
var obj = { a: 2, b: 3 }
for (let i of obj) {
    console.log(i) // Uncaught TypeError: obj is not iterable
}
```

关于Iterator就写到这里了，下一篇写ES6的Generator生成器。

如果该文对你有帮助，点个赞哦，如有错误请指正~~~

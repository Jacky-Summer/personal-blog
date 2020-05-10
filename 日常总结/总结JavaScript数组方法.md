# 总结 JavaScript 数组方法

## 前言

在日常开发中，使用数组方法的情况是很多的。但总有几个方法用得不是很多，就容易忘记具体用法，于是，今天打算做下总结笔记，回顾数组方法。

## 改变原数组的方法（9 个）

### push()和 pop()

共同点：**数组尾部操作的方法**

#### push()

- 用法：push() 方法可向数组的末尾添加一个或多个元素，并返回新的长度。
- 返回值：把指定的值添加到数组后的新长度。

```javascript
arrayObject.push(newEle1, newEle2, ..., newEleX)  要添加到数组末尾的元素
```

```javascript
let arr = [1, 2]
arr.push(4) // 3
console.log(arr) // [1, 2, 4]
```

#### pop()

- 用法：pop() 方法用于删除并返回数组的最后一个元素。
- 返回值：arrayObject 的最后一个元素
- 说明：如果数组已经为空，则 pop() 不改变数组，并返回 undefined 值。

```javascript
arrayObject.pop()
```

```javascript
let arr = [1, 2, 3]
arr.pop() // 3
console.log(arr) // [1, 2]
```

### shift()和 unshift()

共同点：**数组首部操作的方法**

#### shift()

- 用法：用于把数组的第一个元素从其中删除，并返回第一个元素的值。
- 返回值：数组原来的第一个元素的值。
- 说明：如果数组是空的，那么 shift() 方法将不进行任何操作，返回 undefined 值。

```javascript
arrayObject.shift()
```

```javascript
let arr = [1, 2, 3]
arr.shift() // 1
console.log(arr) // [2, 3]
```

#### unshift()

- 用法：可向数组的开头添加一个或更多元素，并返回新的长度。
- 返回值：arrayObject 的新长度。

```javascript
arrayObject.unshift(newEle1, newEle2, ..., newEleX)  要添加到数组首部的元素
```

```javascript
let arr = [1, 2, 3]
arr.unshift(100) // 4
console.log(arr) // [100, 1, 2, 3]
```

### reverse()和 sort()

共同点：**重排序的方法**

#### reverse()

- 用法：用于颠倒数组中元素的顺序。

```javascript
arrayObject.reverse()
```

```javascript
let arr = ['a', 'b', 'c']
arr.reverse()
console.log(arr) // ['c', 'b', 'a']
```

#### sort()

- 用法：用于对数组的元素进行排序。
- 说明：如果调用该方法时没有使用参数，将按字母顺序对数组中的元素进行排序，说得更精确点，是按照字符编码的顺序进行排序。要实现这一点，首先应把数组的元素都转换成字符串（如有必要），以便进行比较。

```javascript
arrayObject.sort(sortby) 可选。规定排序顺序。必须是函数。
```

```javascript
let arr = ['monkey', 'jacky', 'tom', 'john']
arr.sort()
console.log(arr) // ['jacky', 'john', 'monkey', 'tom']
```

- 比较函数的两个参数

```
sort的比较函数有两个默认参数，接受参数然后返回一个用于说明这两个值的相对顺序的数字。比较函数应该具有两个参数 a 和 b，其返回值如下：

若 a 小于 b，在排序后的数组中 a 应该出现在 b 之前，则返回一个小于 0 的值。
若 a 等于 b，则返回 0。
若 a 大于 b，则返回一个大于 0 的值。
```

```javascript
let arr = [1, 5, 7, 3, 100, 4, 8]
// 升序
arr.sort(function (a, b) {
  return a - b
})
console.log(arr) // [1, 3, 4, 5, 7, 8, 100]
// 降序
arr.sort(function (a, b) {
  return b - a
})
console.log(arr) // [100, 8, 7, 5, 4, 3, 1]
```

### splice()

- 用法：向/从数组中添加/删除项目，然后返回被删除的项目。
- 返回值：包含被删除项目的新数组，如果有的话。

```javascript
arrayObject.splice(index, howmany, item1, ....., itemX)
// index:必需。整数，规定添加/删除项目的位置，使用负数可从数组结尾处规定位置。
// howmany:必需。要删除的项目数量。如果设置为 0，则不会删除项目。
// item, ..., itemX:可选。向数组添加的新项目。
```

- 删除元素

```javascript
let arr = [1, 2, 3, 4, 5, 6, 7, 8]
arr.splice(1, 1) // 删除数组下标为 1 的元素，删除一个
console.log(arr) // [1, 3, 4, 5, 6, 7, 8]
arr.splice(-3, 2) // 从倒数第三个元素开始删除，删除两个
console.log(arr) // [1, 3, 4, 5, 8]
```

- 删除并添加

```javascript
let arr = [1, 2, 3, 4, 5, 6, 7, 8]
arr.splice(0, 2, 100) // 删除数组下标为 0 的元素，删除两个；并在数组下标为0的位置添加100
console.log(arr) // [100, 3, 4, 5, 6, 7, 8]
```

- 不删除只添加

```javascript
let arr = [1, 2, 3, 4, 5, 6, 7, 8]
arr.splice(0, 0, 9, 10, 11) // 在数组下标为0的位置添加9 10 11
console.log(arr) // [9, 10, 11, 1, 2, 3, 4, 5, 6, 7, 8]
```

### ES6：fill()

- 用法：用于将一个固定值替换数组的元素。
- 返回值：修改后的数组。

```javascript
arrayObject.fill(value, start, end)
// value:必需。填充的值。
// start:可选。开始填充位置。
// end:	可选。停止填充位置 (默认为 arrayObject.length)
```

```javascript
// 填充 "replace" 到数组的第三和第四个元素：
let arr = ['a', 'b', 'c', 'd', 'e']
arr.fill('replace', 2, 4)
console.log(arr) // ["a", "b", "replace", "replace", "e"]
```

### ES6：copyWithin()

- 用法：用于从数组的指定位置拷贝元素到数组的另一个指定位置中。
- 返回值：修改后的数组。

```javascript
arrayObject.copyWithin(target, start, end)
// value:必需。复制到指定目标索引位置。。
// start:可选。元素复制的起始位置。
// end:	可选。停止复制的索引位置 (默认为 arrayObject.length)。如果为负值，表示倒数。
```

```javascript
let arr = ['a', 'b', 'c', 'd', 'e']
arr.copyWithin(2, 0, 2)
console.log(arr) // ["a", "b", "a", "b", "e"]
```

## 不改变原数组的方法（8 个）

- 注: 数组的每一项是简单数据类型，且未直接操作数组的情况下

### slice()

- 用法：从已有的数组中返回选定的元素。
- 返回值：返回一个新的数组，包含从 start 到 end （不包括该元素）的 arrayObject 中的元素。

```javascript
arrayObject.slice(start, end)
// start:必需。规定从何处开始选取。如果是负数，那么它规定从数组尾部开始算起的位置。
// end:可选。规定从何处结束选取。该参数是数组片断结束处的数组下标。如果没有指定该参数，那么切分的数组包含从 start 到数组结束的所有元素。
```

```javascript
let arr = ['a', 'b', 'c', 'd', 'e']
console.log(arr.slice()) // ["a", "b", "c", "d", "e"]
console.log(arr.slice(2)) // ["c", "d", "e"]
console.log(arr.slice(0, 3)) // ["a", "b", "c"]
```

### indexOf()

- 用法：方法返回在数组中可以找到一个给定元素的第一个索引，如果不存在，则返回-1。

```javascript
arrayObject.indexOf(item, start)
// start:必须。查找的元素。
// end:	可选的整数参数。规定在数组中开始检索的位置。它的合法取值是 0 到 stringObject.length - 1。如省略该参数，则将从字符串的首字符开始检索。
```

```javascript
let arr = ['a', 'b', 'c', 'd', 'e', 'a']
console.log(arr.indexOf('c')) // 2
console.log(arr.indexOf('a', 1)) // 5
```

### lastIndexOf()

- 用法：返回一个指定的元素在数组中最后出现的位置，从该字符串的后面向前查找。

```javascript
arrayObject.lastIndexOf(item, start)
// start:必须。查找的元素。
// end:	可选的整数参数。规定在数组中开始检索的位置。它的合法取值是 0 到 stringObject.length - 1。则将从字符串的最后一个字符处开始检索。
```

```javascript
let arr = ['a', 'b', 'c', 'b', 'e', 'a']
console.log(arr.lastIndexOf('a')) // 5
console.log(arr.lastIndexOf('b', 2)) // 1
```

### join()

- 用法：用于把数组中的所有元素放入一个字符串。元素是通过指定的分隔符进行分隔的。
- 返回值：返回一个字符串。该字符串是通过把 arrayObject 的每个元素转换为字符串，然后把这些字符串连接起来，在两个元素之间插入 separator 字符串而生成的。

```javascript
arrayObject.join(separator)
// separator:可选。指定要使用的分隔符。如果省略该参数，则使用逗号作为分隔符。
```

```javascript
let arr = ['a', 'b', 'c', 'b', 'e', 'a']
console.log(arr.join()) // a,b,c,b,e,a
console.log(arr.join('.')) // a.b.c.b.e.a
```

### concat()

- 用法：用于连接两个或多个数组。
- 返回值：返回一个新的数组。该数组是通过把所有 arrayX 参数添加到 arrayObject 中生成的。如果要进行 concat() 操作的参数是数组，那么添加的是数组中的元素，而不是数组。

```javascript
arrayObject.concat(arrayX, arrayX, ......, arrayX)
// arrayX: 必需。该参数可以是具体的值，也可以是数组对象。可以是任意多个。
```

```javascript
let arr1 = [1, 2, 3]
let arr2 = [6, 7, 8]
console.log(arr1.concat(4, 5)) // [1, 2, 3, 4, 5]
console.log(arr1.concat(arr2)) // [1, 2, 3, 6, 7, 8]
```

### toString()

- 用法：把数组转换为字符串，并返回结果。
- 返回值：arrayObject 的字符串表示。返回值与没有参数的 join() 方法返回的字符串相同。

```javascript
arrayObject.toString()
```

```javascript
let arr = ['a', 'b', 'c', 'b', 'e', 'a']
console.log(arr.toString()) // a,b,c,b,e,a
```

相对 join() 方法，更推荐使用 join() 方法，不推荐使用该方法

### toLocaleString()

- 用法：把数组转换为本地字符串。
- 返回值：arrayObject 的本地字符串表示。
- 说明：首先调用每个数组元素的 toLocaleString() 方法，然后使用地区特定的分隔符把生成的字符串连接起来，形成一个字符串。

```javascript
arrayObject.toLocaleString()
```

```javascript
let arr = ['a', 23, { name: 'jacky' }, new Date()]
console.log(arr.toLocaleString()) // a,23,[object Object],2020/5/10 上午10:58:37
```

### includes()

- 用法：用来判断一个数组是否包含一个指定的值，根据情况，如果包含则返回 true，否则返回 false。
- 返回值：arrayObject 的本地字符串表示。
- 说明：首先调用每个数组元素的 toLocaleString() 方法，然后使用地区特定的分隔符把生成的字符串连接起来，形成一个字符串。

```javascript
arrayObject.includes(searchElement, fromIndex)
// searchElement：必须。需要查找的元素值。
// fromIndex：可选。从该索引处开始查找 searchElement。如果为负值，则按升序从 array.length + fromIndex 的索引开始搜索。默认为 0。
```

```javascript
let arr = [1, 2, 3]
console.log(arr.includes(2)) // true
console.log(arr.includes(4)) // false
console.log(arr.includes(3, 3)) // false
console.log(arr.includes(3, -1)) // true
```

## 遍历方法

### forEach()

- 用法：用于调用数组的每个元素，并将元素传递给回调函数。

```javascript
arrayObject.forEach(function(currentValue, index, arr), thisValue)
// currentValue：必需。当前元素
// index：可选。当前元素的索引值。
// arr：可选。当前元素所属的数组对象。
// thisValue：可选。传递给函数的值一般用 "this" 值。如果这个参数为空， "undefined" 会传递给 "this" 值
```

```javascript
let arr = [1, 2, 4, 6, 8]
arr.forEach((item) => {
  console.log(item) // 1 2 4 6 8
})
```

```javascript
let arr = [1, 2, 4, 6, 8]
arr.forEach((item, index, arr) => {
  console.log(item, index, arr)
})
// 1 0 [1, 2, 4, 6, 8]
// 2 1 [1, 2, 4, 6, 8]
// 4 2 [1, 2, 4, 6, 8]
// 6 3 [1, 2, 4, 6, 8]
// 8 4 [1, 2, 4, 6, 8]
```

### every()

- 用法：用于检测数组所有元素是否都符合指定条件（通过函数提供）。

```javascript
arrayObject.every(function(currentValue, index, arr), thisValue)
// currentValue：必需。当前元素
// index：可选。当前元素的索引值。
// arr：可选。当前元素所属的数组对象。
// thisValue：可选。对象作为该执行回调时使用，传递给函数，用作 "this" 的值。
如果省略了 thisValue ，"this" 的值为 "undefined"
```

- every() 方法使用指定函数检测数组中的所有元素：

  - 如果数组中检测到有一个元素不满足，则整个表达式返回 false ，且剩余的元素不会再进行检测。
  - 如果所有元素都满足条件，则返回 true。

```javascript
let arr = [4, 6, 8, 10, 12]
let res1 = arr.every((num) => {
  // 检测数组中的所有元素是否大于2
  return num > 2
})
let res2 = arr.every((num) => {
  // 检测数组中的所有元素是否大于10
  return num > 10
})
console.log(res1) // true
console.log(res2) // false
```

### some()

- 用法：用于检测数组中的元素是否满足指定条件（函数提供）。

```javascript
arrayObject.some(function(currentValue, index, arr), thisValue)
// currentValue：必需。当前元素
// index：可选。当前元素的索引值。
// arr：可选。当前元素所属的数组对象。
// thisValue：可选。对象作为该执行回调时使用，传递给函数，用作 "this" 的值。
如果省略了 thisValue ，"this" 的值为 "undefined"
```

- some() 方法会依次执行数组的每个元素：

  - 如果有一个元素满足条件，则表达式返回 true , 剩余的元素不会再执行检测。
  - 如果没有满足条件的元素，则返回 false。

```javascript
let arr = [4, 6, 8, 10, 12]
let res1 = arr.some((num) => {
  // 检测数组中是否有一个元素是否大于10
  return num > 10
})
let res2 = arr.some((num) => {
  // 检测数组中是否有一个元素是否大于100
  return num > 100
})
console.log(res1) // true
console.log(res2) // false
```

### filter()

- 用法：创建一个新的数组，新数组中的元素是通过检查指定数组中符合条件的所有元素。
- 返回值：返回数组，包含了符合条件的所有元素。如果没有符合条件的元素则返回空数组。

```javascript
arrayObject.filter(function(currentValue, index, arr), thisValue)
// currentValue：必需。当前元素
// index：可选。当前元素的索引值。
// arr：可选。当前元素所属的数组对象。
// thisValue：可选。对象作为该执行回调时使用，传递给函数，用作 "this" 的值。
如果省略了 thisValue ，"this" 的值为 "undefined"
```

```javascript
let arr = [4, 6, 8, 10, 12]
let res1 = arr.filter((num) => {
  // 返回数组中所有大于5的元素
  return num > 5
})
let res2 = arr.filter((num) => {
  // 返回数组中所有小于5的元素
  return num < 5
})
console.log(res1) // [6, 8, 10, 12]
console.log(res2) // [4]
```

### map()

- 用法：返回一个新数组，数组中的元素为原始数组元素调用函数处理后的值。

```javascript
arrayObject.map(function(currentValue, index, arr), thisValue)
// currentValue：必需。当前元素
// index：可选。当前元素的索引值。
// arr：可选。当前元素所属的数组对象。
// thisValue：可选。对象作为该执行回调时使用，传递给函数，用作 "this" 的值。
如果省略了 thisValue ，"this" 的值为 "undefined"
```

```javascript
let arr = [4, 6, 8, 10, 12]
let res = arr.map((num) => {
  // 数组每个元素都乘10
  return num * 10
})
console.log(res) // [40, 60, 80, 100, 120]
```

## 数组归并方法

### reduce()

- 用法：接收一个函数作为累加器，数组中的每个值（从左到右）开始缩减，最终计算为一个值。
- reduce() 可以作为一个高阶函数，用于函数的 compose。

```javascript
arrayObject.reduce(function(total, currentValue, currentIndex, arr), initialValue)
// total：必需。初始值, 或者计算结束后的返回值。
// currentValue：必需。当前元素
// currentIndex：可选。当前元素的索引
// arr：可选。可选。当前元素所属的数组对象。
// initialValue：可选。传递给函数的初始值
```

```javascript
let arr = [4, 6, 8, 10, 12]
let res = arr.reduce((total, num) => {
  return total + num
})
console.log(res) // 40
```

### reduceRight()

- 用法：方法的功能和 reduce() 功能是一样的，不同的是 reduceRight() 从数组的末尾向前将数组中的数组项做累加。

```javascript
arrayObject.reduceRight(function(total, currentValue, currentIndex, arr), initialValue)
// total：必需。初始值, 或者计算结束后的返回值。
// currentValue：必需。当前元素
// currentIndex：可选。当前元素的索引
// arr：可选。可选。当前元素所属的数组对象。
// initialValue：可选。传递给函数的初始值
```

```javascript
let arr = [2, 4, 6, 20]
let res = arr.reduceRight((total, num) => {
  return total - num
})
console.log(res) // 8
```

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~

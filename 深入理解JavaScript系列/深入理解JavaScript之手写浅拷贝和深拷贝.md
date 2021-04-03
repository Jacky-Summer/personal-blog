# 深入理解 JavaScript 之手写浅拷贝和深拷贝

## 浅拷贝

自己创建一个新的对象，来接受你要重新复制或引用的对象值。如果对象属性是基本的数据类型，复制的就是基本类型的值给新对象；但如果属性是引用数据类型，复制的就是内存中的地址，如果其中一个对象改变了这个内存中的地址，肯定会影响到另一个对象。

### Object.assign

Object.assign 循环遍历原对象的属性，通过复制的方式将其赋值给目标对象的相应属性

- 它不会拷贝对象的继承属性；
- 它不会拷贝对象的不可枚举的属性；
- 可以拷贝 Symbol 类型的属性。

### 扩展运算符方式

```js
let cloneObj = { ...obj }
```

### concat 拷贝数组

数组的 concat 方法其实也是浅拷贝，所以连接一个含有引用类型的数组时，需要注意修改原数组中的元素的属性，因为它会影响拷贝之后连接的数组。

```js
let arr = [1, 2, 3]
let newArr = arr.concat()
newArr[1] = 100
console.log(arr) // [ 1, 2, 3 ]
console.log(newArr) // [ 1, 100, 3 ]
```

### slice 拷贝数组

slice 方法会返回一个新的数组对象，这一对象由该方法的前两个参数来决定原数组截取的开始和结束时间，是不会影响和改变原始数组的。

```js
let arr = [1, 2, { val: 4 }]
let newArr = arr.slice()
newArr[2].val = 1000
console.log(arr) //[ 1, 2, { val: 1000 } ]
```

### 手写浅拷贝

1. 对基础类型做一个最基本的一个拷贝；
2. 对引用类型开辟一个新的存储，并且拷贝一层对象属性。

```js
const shallowClone = target => {
  if (typeof target === 'object' && target !== null) {
    const cloneTarget = Array.isArray(target) ? [] : {}
    for (let prop in target) {
      if (target.hasOwnProperty(prop)) {
        cloneTarget[prop] = target[prop]
      }
    }
    return cloneTarget
  } else {
    return target
  }
}
```

## 深拷贝

将一个对象从内存中完整地拷贝出来一份给目标对象，并从堆内存中开辟一个全新的空间存放新对象，且新对象的修改并不会改变原对象，二者实现真正的分离。

### JSON.stringify

JSON.stringify() 是目前开发过程中最简单的深拷贝方法，其实就是把一个对象序列化成为 JSON 的字符串，并将对象里面的内容转换成字符串，最后再用 JSON.parse() 的方法将 JSON 字符串生成一个新的对象。

1. 拷贝的对象的值中如果有函数、undefined、symbol 这几种类型，经过 JSON.stringify 序列化之后的字符串中这个键值对会消失；
2. 拷贝 Date 引用类型会变成字符串；
3. 无法拷贝不可枚举的属性；
4. 无法拷贝对象的原型链；
5. 拷贝 RegExp 引用类型会变成空对象；
6. 对象中含有 NaN、Infinity 以及 -Infinity，JSON 序列化的结果会变成 null；
7. 无法拷贝对象的循环应用，即对象成环 (obj[key] = obj)。

### 手写深拷贝

1. 针对能够遍历对象的不可枚举属性以及 Symbol 类型，我们可以使用 Reflect.ownKeys 方法； Reflect.ownKeys 返回一个包含所有自身属性（不包含继承属性）的数组。(类似于 Object.keys(), 但不会受 enumerable 影响，且能包含 Symbol 属性)
2. 当参数为 Date、RegExp 类型，则直接生成一个新的实例返回；
3. 利用 Object 的 getOwnPropertyDescriptors 方法可以获得对象的所有属性，以及对应的特性，顺便结合 Object 的 create 方法创建一个新对象，并继承传入原对象的原型链；

   > Object.create(object, [,propertiesObject])创建一个新对象，继承 object 的属性，可添加 propertiesObject 添加属性，并对属性作出详细解释(此详细解释类似于 defineProperty 第二个参数的结构)

4. 解决循环引用问题，我们可以额外开辟一个存储空间，来存储当前对象和拷贝对象的对应关系，当需要拷贝当前对象时，先去存储空间中找，有没有拷贝过这个对象，如果有的话直接返回，如果没有的话继续拷贝，这样就巧妙化解的循环引用的问题。这个存储空间，需要可以存储 key-value 形式的数据，且 key 可以是一个引用类型，我们可以选择 Map 这种数据结构：

- 检查 map 中有无克隆过的对象
- 有 - 直接返回
- 没有 - 将当前对象作为 key，克隆对象作为 value 进行存储
- 继续克隆

```js
const isComplexDataType = obj => (typeof obj === 'object' || typeof obj === 'function') && obj !== null

const deepClone = function (target, hash = new Map()) {
  if (target instanceof Date) return new Date(target) // 日期对象直接返回一个新的日期对象
  if (target instanceof RegExp) return new RegExp(target) // 正则对象直接返回一个新的正则对象
  // 循环引用
  if (hash.has(target)) return hash.get(target)
  // 遍历传入参数所有键的特性
  let allDesc = Object.getOwnPropertyDescriptors(target)
  // 继承原型链
  let cloneTarget = Object.create(Object.getPrototypeOf(target), allDesc)

  hash.set(target, cloneTarget)
  // Reflect.ownKeys：返回一个包含所有自身属性（不包含继承属性）的数组，能包含enumerable(false)、Symbol属性
  for (let key of Reflect.ownKeys(target)) {
    console.log('hash', hash)
    cloneTarget[key] =
      isComplexDataType(target[key]) && typeof target[key] !== 'function' ? deepClone(target[key], hash) : target[key]
  }
  return cloneTarget
}
```

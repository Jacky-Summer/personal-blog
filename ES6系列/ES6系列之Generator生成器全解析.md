# ES6 系列之 Generator 生成器全解析

上一篇我们讲完了 Iterator 迭代器，这次我们来讲下 Generator 生成器，如果不了解 Iterator 的话建议先看我上一篇博客，因为这篇会涉及迭代器的内容。

## 1.为什么需要 Generator

在 JavaScript 中，异步编程场景使用非常多，经常会现需要逐步完成多个异步操作的情况。之前用回调函数实现异步编程如果碰到了这种问题就需要嵌套使用回调函数，异步操作越多，嵌套得就越深，导致代码的可维护性较差，代码阅读起来也很困难。

Generator 函数是 ES6 提出的一种异步编程解决方案，它可以避免回调的嵌套，语法行为与传统函数完全不同。除此之外，Generator 的特性在某些场景使用也十分方便。

## 2.概念

语法上，首先可以把它理解成，Generator 函数是一个状态机，还是一个 Iterator 对象生成函数。它返回的遍历器对象可以依次遍历 Generator 函数内部的每一个状态。Generator 函数是生成一个对象，但是调用的时候前面不能加 new 命令。

通俗来说，Generator 函数它可以随时暂停函数运行，并可以在任意时候恢复函数运行。

## 3.与 Iterator 接口的关系

任意一个对象的 Symbol.iterator 方法，等于该对象的遍历器生成函数，调用该函数会返回该对象的一个遍历器对象。

由于 Generator 函数就是 Iterator 迭代器生成函数，因此可以把 Generator 赋值给对象的 Symbol.iterator 属性，从而使得该对象具有 Iterator 接口。

```javascript
var myIterable = {}
myIterable[Symbol.iterator] = function* () {
  yield 1
  yield 2
  yield 3
}

console.log([...myIterable]) // [1, 2, 3]
```

上面代码中，Generator 函数赋值给 Symbol.iterator 属性，从而使得 myIterable 对象具有了 Iterator 接口，可以被...运算符遍历了。

Generator 函数执行后，返回一个迭代器对象。该对象本身也具有 Symbol.iterator 属性，执行后返回自身。

```javascript
function* gen() {
  // some code
}

var g = gen()

console.log(g[Symbol.iterator]() === g) // true
```

上面代码中，gen 是一个 Generator 函数，调用它会生成一个迭代器对象 g。它的 Symbol.iterator 属性，也是一个遍历器对象生成函数，执行后返回它自己。

## 4.特点

1. function 关键字与函数名之间有一个`*`号，而且这个`*`前后允许有空白字符，如：

```javascript
function* foo() {}
function* foo() {}
function* foo() {}
function* foo() {}
```

以上四种写法都可以，根据个人不同习惯，一般第三和第四种用得比较多

2. 函数体内部使用 yield 表达式，定义不同的内部状态
3. 普通函数的执行模式是: **执行-结束**, 生成器的执行模式是: **执行-暂停-结束**。生成器可以在执行当中暂停自身，可以立即恢复执行也可以过一段时间之后恢复执行。最大的区别就是它不像普通函数那样保证运行到完毕。

## 5.yield 关键字

生成器函数中，有一个特殊的新关键字：`yield`。由于 Generator 函数返回的是一个 Iterator 对象，只有调用 next 方法才会遍历下一个内部状态,而`yield`关键字就是**暂停标志**。因为有它，所以才能实现执行-暂停-结束的执行模式。

`yield` 后面可以是任意合法的 JavaScript 表达式，`yield`语句可以出现的位置可以等价于一般的赋值表达式（比如 a=2）能够出现的位置。如下：

```javascript
b = 2 + a = 2 // 不合法
b = 2 + yield 2 // 不合法

b = 2 + (a = 2) // 合法
b = 2 + (yield 2) // 合法
```

`yield`关键字的优先级比较低，几乎`yield`之后的任何表达式都会先进行计算，然后再通过`yield`向外界产生值。而且`yield`是右结合运算符，也就是说

```
yield yield 2 等价于 (yield (yield 2))
```

**总结**：

1. 它可以指定调用 next 方法时的返回值以及调用顺序。
2. 每当执行完 yield 语句，函数就会停止执行，直到再次调用 next 方法才会继续执行
3. yield 关键字只能在生成器内部使用，其他地方会导致语法错误

## 6.运行生成器函数

说了这么多，我们来举个简单的例子

```javascript
function* gen() {
  yield 1
  yield 2
  yield 3
}
```

上述为一个生成器函数，如何运行它呢？我们知道，生成器还是一个遍历器/迭代器生成函数，也就是说，返回的 Iterator 对象，可以依次遍历生成器函数内部的每一个状态，所以我们可以使用 next()方法让它运行。

```javascript
let generator = gen() // 生成器返回的是一个指向内部状态的generator对象
console.log(generator.next()) // {value: 1, done: false}
console.log(generator.next()) // {value: 2, done: false}
console.log(generator.next()) // {value: 3, done: false}
console.log(generator.next()) // {value: undefined, done: true}
```

首先要知道的是，Generator 函数，不管内部有没有`yield`语句，调用函数时都不会执行任何语句，也不返回函数执行结果，而是返回一个指向内部状态的 generator 对象，也可以看作是一个 Iterator 对象。只有当调用 next(),内部语句才会执行。

在该函数内部有 3 个`yield`表达式，即该函数有三个状态：1、2、3。而 Generator 函数在此分段执行，调用 next 方法函数内部逻辑开始执行，遇到`yield`表达式停止，返回 Iterator 对象，再次调用 next 方法，会从上一次停止时的`yield`处开始，直到最后。所以我们也不难理解`yield`语句只是函数暂停执行的一个标记。

再来看个例子：

```javascript
function* gen() {
  yield 'hello'
  yield 'world'
  return 'ending'
}
let generator = gen()
console.log(generator.next()) // { value: 'hello', done: false }
console.log(generator.next()) // { value: 'world', done: false }
console.log(generator.next()) // { value: 'ending', done: true }
console.log(generator.next()) // { value: undefined, done: true }
```

上述生成器函数 gen 有三个状态：hello，world 和 return 语句。

第一次调用 next 方法，生成器函数开始执行，遇到第一个 yield 表达式，返回一个对象，它的 value 属性就是当前 yield 表达式的值 hello，done 属性的值 false，表示遍历还没有结束。

同理第二次调用也是按上述逻辑推

第三次调用 next 方法，遇到了 return 语句（如果没有 return 语句，就执行到函数结束）。此时 next 方法返回的对象的 value 属性，就是紧跟在 return 语句后面的表达式的值，在这里 return 后的值即是"ending"（如果没有 return 语句，则 value 属性的值为 undefined），done 属性的值 true，表示遍历已经结束。

第四次调用，此时 生成器函数已经运行完毕，故 next 方法返回对象的 value 属性为 undefined，done 属性为 true。以后无论调用多少次 next 方法，返回的都是这个值。

## 7.next 传递参数

由上述我们知道总结 next 方法的运行逻辑：

1. 遇到 `yield` 语句暂停执行后面的操作，并将在 `yield` 后面那个表达式的值，作为返回对象的 value 属性值
2. 下次调用 next 方法时，再继续往下执行，直到遇到下一个 `yield` 语句
3. 如果没有遇到新的 `yield` 语句，就一直运行函数结束 到 return 语句为止，将 return 的值作为返回的对象 value 的属性值
4. 如果该函数没有 return 语句，则返回的对象的 value 属性值为 undefined

那 next 方法可以带参数吗？答案当然是可以的。

```javascript
function* gen(a) {
  let b = yield a + 1
  return b * 4
}
let generator1 = gen(1)
console.log(generator1.next()) // {value: 2, done: false}
console.log(generator1.next()) // {value: NaN, done: true}

let generator2 = gen(1)
console.log(generator2.next(2)) // {value: 2, done: false}
console.log(generator2.next(4)) // {value: 16, done: true}
```

上面两个例子中，生成器函数 gen 都接收了一个参数 a,都传入了 1。

第一个例子中：第一次调用 next 方法，let b = yield 1 + 1 = yield 2,即返回`yield`的 value 值为 2；到了第二次调用时，语句 return b _ 2，返回的对象 value 值居然为 NaN。由此可推断此时的 b 并不是上次计算的结果 2，而是 undefined，所以 undefined _ 4 = NaN

看到这里，不懂生成器的小伙伴可能有点懵逼，表面上变量 b 已经用 yield 语句赋值了，但并没有赋值成功。

再第二个例子：第一次调用 next 方法，let b = yield (1 + 1),此时的 next 方法传入了参数，值为 1，这个参数有什么用呢？答：确实没什么用，这里只是为了示范它对结果无影响。

> 第一个 next 方法只是用来启动生成器函数内部的遍历器，传参也没有多大意义。即使传了值，它只是将这个传入的值抛弃而已。ES6 表明，generator 函数在这种情况只是忽略了这些没有被用到的值。

所以 let b = yield (1 + 1) = yield 2, 返回 value 值仍然为 2；第二次调用 next 方法时传入了参数 4，此时 4 覆盖上次 yield 语句的返回值 2，即 let b = 4;故执行 return b _ 4 语句时，即 return 4 _ 4，即返回了 16。

- 上述结果可以说明：
  - **yield 语句没有返回值，或者总是返回 undefined；**
  - **next 方法如果带上一个参数，这个参数就是作为上一个 yield 语句的返回值。
    注意：因为 next 方法表示上一个 yield 语句的返回值，所以必须有上一个 yield 语句的存在，那么第一次调用 next 方法时就不能传参数。第一个 next 只是用来启动 Generator 函数内部的遍历器，传参也没有多大意义。**

对于上述两点，我们需要代码再次加深理解：

yield 后可以不带任何表达式的时候，返回的 value 为 undefined。

```javascript
const gen = (function* () {
  yield
})()
console.log(gen.next()) // { value: undefined, done: false }
```

再举一个例子：

```javascript
function* gen(x) {
  let y = 2 * (yield x + 2)
  let z = yield y / 4
  return x + y + z
}
let generator = gen(2)
console.log(generator.next()) // {value: 4, done: false} 返回yield（2+2）= 4
console.log(generator.next(7)) // {value: 3.5, done: false} 设置yield(x+2) = 7,那么y= 2*7=14,那么yield(y/4) = 3.5
console.log(generator.next(3)) // {value: 19, done: false} 设置z = yield(y/4) = 3 那么 x+y+z = 2+14+3 = 19
```

1. 第一次调用 next 方法，返回 yield（2+2）= 4，返回 value 值为 4
2. 第二次调用 next 方法，next 参数值为 7，覆盖上一次的 yield 值，故 let y = 2 \* 7 = 14，则 let z = yield(14 / 4) = 3.5
3. 第三次调用 next 方法，next 参数值为 3，覆盖上一次的 yield 值，故 z = 3，x + y + z = 2 + 14 + 3 = 19

这里初次看可能有点绕，建议多看多试几次就容易理解了。

## 8.return 与 yield 区别

从以上的例子也可以看出，函数不仅是碰到 yield 语句才会停止执行，碰到 return 语句也会停止执行。普通函数遇到 return 也会停止执行，Generator 函数也是一个函数，所以很好理解。那么，两者的区别是什么呢？先来看个例子：

```javascript
function* gen() {
  return 'hello'
  yield 1
  yield 'world'
}
let generator = gen()
console.log(generator.next()) // {value: "hello", done: true}
console.log(generator.next()) // {value: undefined, done: true}
```

从上面例子可以看出，当碰到 return 语句时，返回对象的 value 值为紧跟 return 后面的值，done 属性值就为 true，代表遍历结束，不管后面是否还有 yield 或者 return 语句。这种区别本质上是因为 yield 语句具备位置记忆功能而 return 语句则没有该功能。

## 9.return 与 next 参数区别

可以总结为三点：

1. return 终结遍历，之后的 yield 语句都失效；next 返回本次 yield 语句的返回值。
2. return 没有参数的时候，返回{ value: undefined, done: true }；next 没有参数的时候返回本次 yield 语句的返回值。
3. return 有参数的时候，覆盖本次 yield 语句的返回值，也就是说，返回{ value: 参数, done: true }；next 有参数的时候，覆盖上次 yield 语句的返回值，返回值可能跟参数有关（参数参与计算的话），也可能跟参数无关（参数不参与计算）。

## 10.return(value)方法

在生成器里使用 return(value)方法，随时终止生成器，如下面代码所示：

```javascript
function* gen() {
  yield 1
  yield 2
  yield 3
}
const generator = gen()
console.log(generator.next()) // {value: 1, done: false}
console.log(generator.return(10)) // {value: 10, done: true}
console.log(generator.next()) // {value: undefined, done: true}
```

从上述代码我们看出，使用 return()方法我们提前终止了生成器，返回 return 里的值，再次调用 next()方法时，done 属性的值为 true,遍历结束，由此可见 return 提前终止了生成器。

## 11. throw(exception)方法

除了用 return(value)方法可以终止生成生成器，我们还可以调用 throw(exception) 进行提前终止生成器，示例代码如下：

```javascript
function* gen() {
  yield 1
  yield 2
  yield 3
}
let generator = gen()
console.log(generator.next())
try {
  generator.throw('throw error msg')
} catch (err) {
  console.log(err)
} finally {
  console.log('必执行的finally语句块')
}
console.log(generator.next())
```

输出结果;

```javascript
{value: 1, done: false}
throw error msg
必执行的finally语句块
{value: undefined, done: true}
```

由此可以看出，在生成器外部调用 try…catch…finally,throw()异常被 try…catch 捕捉并返回，并执行了 finally 代码块中的代码。当再次调用 next 方法，done 属性返回 true,说明生成器已被终止。

我们不仅可以在 next 执行过程中插入 throw()语句，我们还可以在生成器内部插入 try…catch 进行错误处理。

> throw 方法抛出的错误要被内部捕获，前提是必须至少执行过一次 next 方法。

代码如下所示：

```javascript
function* gen() {
  try {
    yield 1
  } catch (e) {
    console.log('first Exception')
  }
  try {
    yield 2
  } catch (e) {
    console.log('second Exception')
  }
}
const generator = gen()
console.log(generator.next())
console.log(generator.throw('exception string'))
console.log(generator.throw('exception string'))
```

输出结果：

```javascript
{value: 1, done: false}
first Exception
{value: 2, done: false}
second Exception
{value: undefined, done: true}
```

从代码输出可以输出，当我们在 generator.throw()方法时，被生成器内部上个暂停点的异常处理代码所捕获，会附带执行下一条`yield`表达式。也就是说，会附带执行一次 next 方法。由此可见在生成器内部使用 try…catch 可以捕获异常，并不会影响到遍历器的状态。

## 12.yield\* 表达式

如果想要在 Generator 函数内部，调用另一个 Generator 函数。需要在前者的函数体内部，自己手动完成遍历。

```javascript
function* gen1() {
  yield 1
  yield 2
}

function* gen2() {
  yield 'a'
  // for...of遍历 gen1()
  for (let i of gen1()) {
    console.log(i)
  }
  yield 'b'
}

for (let v of gen2()) {
  console.log(v)
}
```

输出结果：

```javascript
a
1
2
b
```

如果有多个 Generator 函数嵌套，写起来就非常麻烦。
ES6 提供了`yield*`表达式，作为解决办法，用来在一个 Generator 函数里面执行另一个 Generator 函数。

把上面的 gen2 函数改写如下,如下也能得到相同的输出结果

```javascript
function* gen2() {
  yield 'a'
  yield* gen1()
  yield 'b'
}
```

`yield*`可以将可迭代的对象 iterable 放在一个生成器里，生成器函数运行到`yield*` 位置时，将控制权委托给这个迭代器，直到执行完成为止。举个例子，数组也是可迭代对象，因此`yield*`也可委托给数组：

```javascript
function* gen1() {
  yield 2
  yield 3
}
function* gen2() {
  yield 1
  yield* gen1()
  yield* [4, 5]
}
const generator = gen2()
console.log(generator.next()) // {value: 1, done: false}
console.log(generator.next()) // {value: 2, done: false}
console.log(generator.next()) // {value: 3, done: false}
console.log(generator.next()) // {value: 4, done: false}
console.log(generator.next()) // {value: 5, done: false}
console.log(generator.next()) // {value: undefined, done: true}
```

## 13.Generator 的应用场景

### 1.异步操作的同步化表达

Generator 函数的暂停执行的效果，意味着可以把异步操作写在 yield 表达式里面，等到调用 next 方法时再往后执行。这实际上等同于不需要写回调函数了，因为异步操作的后续操作可以放在 yield 表达式下面，反正要等到调用 next 方法时再执行。所以，Generator 函数的一个重要实际意义就是用来处理异步操作，改写回调函数，避免回调嵌套。

```javascript
function* loadUI() {
  showLoadingScreen()
  yield loadUIDataAsynchronously()
  hideLoadingScreen()
}
var loader = loadUI()
// 加载UI
loader.next()

// 卸载UI
loader.next()
```

上面代码中，第一次调用 next 方法时，则会显示 Loading 界面，并且异步加载数据。等到数据加载完成，再一次使用 next 方法，则会隐藏 Loading 界面。可以看到，这种写法的好处是所有 Loading 界面的逻辑，都被封装在一个函数，按部就班非常清晰。

### 2.异步任务的封装

```javascript
var fetch = require('node-fetch')

function* gen() {
  var url = 'https://api.github.com/users/github'
  var result = yield fetch(url)
  console.log(result.bio)
}

var g = gen()
var result = g.next()

result.value
  .then(function (data) {
    return data.json()
  })
  .then(function (data) {
    g.next(data)
  })
```

Generator 函数封装了一个异步操作，该操作先读取一个远程接口，然后从 JSON 格式的数据解析信息。不过也可以看到，虽然 Generator 函数将异步操作表示得很简洁，但是流程管理却不方便（即何时执行第一阶段、何时执行第二阶段）。

### 3.部署 Iterator 接口

利用 Generator 函数，可以在任意对象上部署 Iterator 接口。

```javascript
function* iterEntries(obj) {
  let keys = Object.keys(obj)
  for (let i = 0; i < keys.length; i++) {
    let key = keys[i]
    yield [key, obj[key]]
  }
}

let myObj = { foo: 3, bar: 7 }

for (let [key, value] of iterEntries(myObj)) {
  console.log(key, value)
}
```

上述代码中，myObj 是一个普通对象，通过 iterEntries 函数，就有了 Iterator 接口。也就是说，可以在任意对象上部署 next 方法。

### 4.抽奖程序

比如当前用户还可以抽奖 5 次，用户点了 5 次抽奖后就不能继续抽奖了，如做一个剩余抽奖次数的限制。

```html
<button id="start">抽奖</button>
```

```javascript
let draw = function (count) {
  // 具体抽奖逻辑
  // ...
  console.log(`剩余${count}次`)
}

let residue = function* (count) {
  while (count > 0) {
    // 抽奖次数的限制
    count--
    yield draw(count) //执行抽奖的逻辑
  }
}

let start = residue(5)
document.getElementById('start').addEventListener('click', function () {
  start.next()
})
```

然后我们通过点击 5 次抽奖按钮,依次输出剩余几次。当剩余 0 次时，再点击按钮不会有任何输出

```
剩余4次
剩余3次
剩余2次
剩余1次
剩余0次
```

上述通过 generator 来控制抽奖的次数限制的好处是：抽奖次数无需保存在全局变量中，而且把抽奖具体逻辑给分离开。

## 14.Generator 函数的语法糖—async 函数

ES2017 标准引入了 async 函数，使得异步操作变得更加方便，而 async 函数是就是 Generator 函数的语法糖。

1. `async` 对应的是 `*`
2. `await` 对应的是 `yield`

`async`和`await`，比起`*`号和`yield`，语义更清楚了。async 表示函数里有异步操作，await 表示紧跟在后面的表达式需要等待结果。

比如我们假装模拟一个请求 ajax 方法返回用户数据并输出显示的例子

```javascript
function ajax() {
  return '用户数据...'
}
function* fetchUser() {
  const user = yield ajax()
  console.log(`输出的user数据:${user}`)
}
// 当然真实项目不可能这么写，在此只是方便拿数据
const generator = fetchUser()
let obj = generator.next()
generator.next(obj.value)
```

如果用 async 函数代替,代码就会优雅很多

```javascript
async function fetchUser() {
  const user = await ajax()
  console.log(`输出的user数据:${user}`)
}
fetchUser()
```

这里可以看出并不需要调用 next()方法，因为 async 函数自带执行器。也就是说，
async 函数的实现，就是将 Generator 函数和自动执行器，包装在一个函数里。

## 15.总结

Generator 是一个可以暂停和继续执行的函数，他可以完全实现 Iterator 的功能，并且由于可以保存上下文，他非常适合实现简单的状态机。另外通过一些流程控制代码的配合，可以比较容易进行异步操作。

Async/Await 就是 generator 进行异步操作的语法糖，该语法糖相比下有更好的应用和语义化，它们搭配 promise，可以通过编写形似同步的代码来处理异步流程，提高代码的简洁性和可读性。

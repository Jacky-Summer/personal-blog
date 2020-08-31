# jquery 源码分析（一）

最近开始阅读 jquery 的源码，首先先提炼出 jquery 的核心结构。

## 自执行函数

```javascript
;(function (window, undefined) {
  //...
})(window)
```

### 为什么传入 window?

**1.代码压缩**

首先从代码压缩混淆的角度考虑，用线上工具来压缩混淆下面这段示例代码：

```javascript
function test() {
  var name = 'hello'
  window.description = 'hi ' + name
}
```

压完混淆完后瘦了一点：

```javascript
function say() {
  var a = 'hello'
  window.description = 'hi ' + a
}
```

用 a 代替了 name，但是 window 既不是声明的局部变量也不是参数，是不会被压缩混淆的。
所以将 window 作为参数传入可解决这个问题。将 window 作为参数传入，可以在压缩代码时进行优化，即参数名可以压缩变短。也可以在函数内另起个简短的变量名代替。

**2.作用域**

访问当前作用域下的变量比访问全局变量要快
如果不传入，每个语句都要去找一次 window。如果将 window 作为参数传递过去，不要每个语句都去找 window，提高了效率。

### 为什么要传入 undefined？

undefined 并不是作为 JavaScript 的保留关键字；undefined 在 IE8 及以下中是可以对其重新赋值的

```javascript
var undefined = 'new value'
alert(undefined) //alert “new value"
```

执行匿名函数的时候，只传递一个参数 window，而不传递 undefined，那么函数体中的 undefined 局部变量的值，刚好就是 undefined

```javascript
var undefined = 8
;(function (window) {
  alert(window.undefined) // 8
  alert(undefined) // 8
})(window)
```

加了 undefined 后

```javascript
var undefined = 8
;(function (window, undefined) {
  alert(window.undefined) // 8
  alert(undefined) // 此处undefined参数为局部的名称为undefined变量，值为undefined
})(window)
```

在 自调用匿名函数 的作用域内，确保 undefined 是真的未定义。因为 undefined 能够被重写，赋予新的值.

## new jQuery.fn.init( selector, context );

```javascript
jQuery = function (selector, context) {
  return new jQuery.fn.init(selector, context)
  //=>创建了init这个类的实例，也相当于创建了jQuery这个类的实例（因为在后面的时候，让init.prototype=jQuery.prototype）
}
jQuery.fn = jQuery.prototype = {}
```

要理解这段代码，首先要理解 jquery 的两种调用

```javascript
$('body').css('background', 'red')
$.parseJSON('{}')
```

$('body')应该是一个实例对象，css是每个实例共享的方法，是原型上的方法。
而$则是一个类，parseJSON 则是类的静态方法。

### 如何不用 new 关键字得到 jQuery 对象？

一般获取实例对象调取实例方法的例子，如果没有使用 new 则会报错

```javascript
var demo = new Demo('jacky')
demo.change()
```

但是获取 jQuery 对象（以下简称 JQ 对象）用 new 和不用 new 都可以，返回的是一样的。

```javascript
console.log($('*').length) //14
console.log(new $('*').length) //14
```

为了做到这点，我们很容易想到需要**在构造函数内部返回对象**。

构造函数有 return 值怎么办？
**构造函数里没有显式调用 return 时，默认是返回 this 对象，也就是新创建的实例对象。**

当构造函数里调用 return 时，分两种情况：

> 1.return 的是五种简单数据类型：String，Number，Boolean，Null，Undefined。 这种情况下，忽视 return 值，依然返回 this 对象。
> 2.return 的是 Object 这种情况下，不再返回 this 对象，而是返回 return 语句的返回值。

所以我们应该在 jQuery 构造函数内部去返回一个对象，这样就可以不用 new 的方式去创建 JQ 对象了，其实这时候，构造函数就相当于一个工厂函数了。

该返回什么样的对象？对于这个对象有何要求？
这个对象必须可以调用 jQuery.prototype 上的方法

我们使用或自己写 jQuery 插件的时候会经常遇到$.fn这个对象，很多插件都是通过扩展这个对象来实现的。
$.fn 其实对应着 jQuery.prototype，$和fn分别是jQuery和prototype的简写方式，只要我们把方法扩展到这个原型对象身上，通过$()获取的 JQ 对象都是可以访问到方法的。

```javascript
$.fn.greeting = function () {
  alert('hi')
}
$('body').greeting() //alert 'hi'
```

jquery 源码

```javascript
jQuery = function( selector, context ) {
    return new jQuery.fn.init( selector, context, rootjQuery );
},
jQuery.fn = jQuery.prototype = { //fn即对应prototype
    constructor: jQuery,
    init: function( selector, context, rootjQuery ) {
        ...
        return this;
    }
    ...
}
```

jQuery.fn.init.prototype = jQuery.fn;

看到上面这段源码，原因就很明显了，"jQuery.fn.init.prototype = jQuery.fn"这句很重要，它将 init 的原型指向 jQuery 的原型，所以 JQ 对象才可以访问‘css'、'show'、'hide'这些写在 jQuery.fn 上的方法。

其实我们所说的 JQ 对象根本就是 init 函数的实例对象，而 init 则是 jQuery 原型上的一个对象，它本身是没有什么方法的，全靠从 jQuery 原型上拿。

为何要从 init 这绕这么一大圈来访问 jQuery 的原型，而不是直接返回一个 jQuery 实例直接通过这个实例来访问自身原型？比如说代码可以写成这样：

```javascript
jQuery = function (selector, context) {
  return new jQuery()
}
```

问题很明显，死在循环里。

```javascript
jQuery = function (selector, context) {
  return jQuery.fn.init() //不同点在于去掉了new关键字
}
```

做点动作来证明加上 new 是有用的

```javascript
;(jQuery = function (selector, context) {
  return jQuery.fn.init()
}),
  (jQuery.fn = jQuery.prototype = {
    init: function () {
      this.name = 'sheila'
      return this
    },
    anotherName: 'sunwukong',
  })

var jq = jQuery()
console.log(jq.anotherName) //"sunwukong"
console.log(jq.name) //"sheila"
```

上面这段代码是为了说明 this 的作用域问题，其不仅能访问 init 函数内部，还能向上一层到 fn 对象，作用域不独立.如果是

```javascript
return new jQuery.fn.init()
```

```javascript
console.log(jq.anotherName) //undefined
console.log(jq.name) //"sheila"
```

加不加 new 还牵涉到一个更重要的问题：返回的对象究竟是谁。

不加 new 的情况下，'jQuery.fn.init()'相当于调用方法，this 指向的以及最后返回的都是同一个 jQuery.fn 对象，$('body')和$('p')就没有区分了。

显然，这是不合理的。而加了 new，就是每次用构造函数实例化了一个新对象，彼此都是不同的

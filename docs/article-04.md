# jquery源码分析（一）

最近开始阅读jquery的源码，首先先提炼出jquery的核心结构。

## 自执行函数
```javascript
(function(window,undefined){
    //...
})(window);
```
### 为什么传入window?

**1.代码压缩**

首先从代码压缩混淆的角度考虑，用线上工具来压缩混淆下面这段示例代码：

```javascript
function test(){
  var name="hello";
  window.description="hi "+name;
}
```
压完混淆完后瘦了一点：

```javascript
function say(){var a="hello";window.description="hi "+a}
```
用a代替了name，但是window既不是声明的局部变量也不是参数，是不会被压缩混淆的。
所以将window作为参数传入可解决这个问题。将window作为参数传入，可以在压缩代码时进行优化，即参数名可以压缩变短。也可以在函数内另起个简短的变量名代替。  

**2.作用域**

访问当前作用域下的变量比访问全局变量要快
如果不传入，每个语句都要去找一次window。如果将window作为参数传递过去，不要每个语句都去找window，提高了效率。

### 为什么要传入undefined？
undefined并不是作为JavaScript的保留关键字；undefined在IE8及以下中是可以对其重新赋值的

```javascript
var undefined="new value";
alert(undefined);//alert “new value"
```

执行匿名函数的时候，只传递一个参数window，而不传递undefined，那么函数体中的undefined局部变量的值，刚好就是undefined
```javascript
var undefined = 8;  
(function( window ) {   
    alert(window.undefined); // 8  
    alert(undefined); // 8  
})(window);  
```

加了undefined后

```javascript
var undefined = 8;  
(function( window, undefined ) {   
    alert(window.undefined);  // 8  
    alert(undefined); // 此处undefined参数为局部的名称为undefined变量，值为undefined  
})(window);  
```
在 自调用匿名函数 的作用域内，确保undefined是真的未定义。因为undefined能够被重写，赋予新的值.
## new jQuery.fn.init( selector, context );
```javascript
jQuery = function( selector, context ) {
	return new jQuery.fn.init( selector, context );
    //=>创建了init这个类的实例，也相当于创建了jQuery这个类的实例（因为在后面的时候，让init.prototype=jQuery.prototype）
};
jQuery.fn = jQuery.prototype = {

}
```
要理解这段代码，首先要理解jquery的两种调用

```javascript
$('body').css('background','red');
$.parseJSON('{}');
```
$('body')应该是一个实例对象，css是每个实例共享的方法，是原型上的方法。
而$则是一个类，parseJSON则是类的静态方法。  

### 如何不用new关键字得到jQuery对象？

一般获取实例对象调取实例方法的例子，如果没有使用new则会报错

```javascript
var demo=new Demo("jacky");
demo.change();
```
但是获取jQuery对象（以下简称JQ对象）用new和不用new都可以，返回的是一样的。

```javascript
console.log($('*').length);//14
console.log(new $('*').length);//14
```
为了做到这点，我们很容易想到需要**在构造函数内部返回对象**。

构造函数有return值怎么办？
**构造函数里没有显式调用return时，默认是返回this对象，也就是新创建的实例对象。**

当构造函数里调用return时，分两种情况：

> 1.return的是五种简单数据类型：String，Number，Boolean，Null，Undefined。 这种情况下，忽视return值，依然返回this对象。
> 2.return的是Object 这种情况下，不再返回this对象，而是返回return语句的返回值。

所以我们应该在jQuery构造函数内部去返回一个对象，这样就可以不用new的方式去创建JQ对象了，其实这时候，构造函数就相当于一个工厂函数了。

该返回什么样的对象？对于这个对象有何要求？
这个对象必须可以调用jQuery.prototype上的方法

我们使用或自己写jQuery插件的时候会经常遇到$.fn这个对象，很多插件都是通过扩展这个对象来实现的。
$.fn其实对应着jQuery.prototype，$和fn分别是jQuery和prototype的简写方式，只要我们把方法扩展到这个原型对象身上，通过$()获取的JQ对象都是可以访问到方法的。

```javascript
$.fn.greeting=function(){alert('hi')};
$('body').greeting();//alert 'hi'
```

jquery源码

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

看到上面这段源码，原因就很明显了，"jQuery.fn.init.prototype = jQuery.fn"这句很重要，它将init的原型指向jQuery的原型，所以JQ对象才可以访问‘css'、'show'、'hide'这些写在jQuery.fn上的方法。

其实我们所说的JQ对象根本就是init函数的实例对象，而init则是jQuery原型上的一个对象，它本身是没有什么方法的，全靠从jQuery原型上拿。

为何要从init这绕这么一大圈来访问jQuery的原型，而不是直接返回一个jQuery实例直接通过这个实例来访问自身原型？比如说代码可以写成这样：

```javascript
jQuery = function( selector, context ) {
    return new jQuery();
} 
```

问题很明显，死在循环里。

```javascript
jQuery = function( selector, context ) {
    return jQuery.fn.init();//不同点在于去掉了new关键字
}
```

做点动作来证明加上new是有用的

```javascript
jQuery = function( selector, context ) {
    return jQuery.fn.init();
},
jQuery.fn = jQuery.prototype = {
    init: function() {
        this.name='sheila';
        return this;
    },
    anotherName:'sunwukong'
};

var jq=jQuery();
console.log(jq.anotherName);//"sunwukong"
console.log(jq.name);//"sheila"

```

上面这段代码是为了说明this的作用域问题，其不仅能访问init函数内部，还能向上一层到fn对象，作用域不独立.如果是

```javascript
return new jQuery.fn.init();
```

```javascript
console.log(jq.anotherName);//undefined
console.log(jq.name);//"sheila"
```
加不加new还牵涉到一个更重要的问题：返回的对象究竟是谁。

不加new的情况下，'jQuery.fn.init()'相当于调用方法，this指向的以及最后返回的都是同一个jQuery.fn对象，$('body')和$('p')就没有区分了。

显然，这是不合理的。而加了new，就是每次用构造函数实例化了一个新对象，彼此都是不同的
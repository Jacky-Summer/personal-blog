# 原生JavaScript DOM操作常见汇总

## 前言

最近在看JS性能优化方面，说到了createDocumentFragment()方法，对该方法有印象，但为了进一步了解，我还是重新查阅一遍，顺便把DOM操作也给总结一下。

## 创建新节点

- **createDocumentFragment()方法**

该方法是用来创建一个虚拟的节点对象，或者说，是用来创建文档碎片节点。它可以包含各种类型的节点，在创建之初是空的。

DocumentFragment 不是真实 DOM 树的一部分，它的变化不会触发 DOM 树的重新渲染，且不会导致性能等问题。

当把一个DocumentFragment节点插入文档树时，插入的不是DocumentFragment自身，而是它的所有子孙节点，即插入的是括号里的节点。这个特性使得DocumentFragment成了占位符，暂时存放那些一次插入文档的节点。

- **createElement()方法**

在 HTML 文档中，document.createElement() 方法用于创建一个由标签名称 tagName 指定的 HTML 元素

来看一个例子, 对于页面已存在的```<ul id="ul"></ul>```元素，我们想往里面添加li标签
```javascript
var ul = document.getElementById("ul");
for (var i = 0; i < 50; i++) {
    var li = document.createElement("li");
    li.innerHTML = "index: " + i;
    ul.appendChild(li);
}
```
上面操作看起来很正常，但实际要很多的插入操作和改动；而每一次的插入都会引起重新渲染，该操作会引发多次渲染，在性能优化方面，有一点是减少DOM操作，因为DOM操作导致了页面的重绘或重排。

- createDocumentFragment()方法和createElement()方法区别

如果是用createDocumentFragment()方法操作
```javascript
var ul = document.getElementById("ul");
var fragment = document.createDocumentFragment();
for (var i = 0; i < 50; i++) {
    var li = document.createElement("li");
    li.innerHTML = "index: " + i;
    fragment.appendChild(li);
}
ul.appendChild(fragment);
```

相比createElement()方法，这次是先将这些元素添加到fragment中，再统一将fragment添加到页面，会减少页面渲染dom的次数，效率会明显提升。因为fragment文档片段存在于内存中，并不在DOM中，所以将子元素插入到文档片段中时不会引起页面回流（新创建的fragment片段在文档内是没有对应的标签的，这里添加的是片段的所有子节点）

- createTextNode()方法

该方法会创建一个文本节点。

HTML元素通常是由元素节点和文本节点组成。创建一个标题 (h1), 你必须创建 "h1" 元素和文本节点:
```javascript
var h = document.createElement("h1")
var t = document.createTextNode("Hello World");
h.appendChild(t);
```

## DOM更改：添加、移除、替换、插入 

```javascript
// 添加、删除子元素
ele.appendChild(el);
ele.removeChild(el);

// 替换子元素
ele.replaceChild(el1, el2);

// 插入子元素
parentElement.insertBefore(newElement, referenceElement);
```
前面三个用法就一目了然，对于insertBefore()方法来举个例子：
```html
<ul id="ul">
    <li>1</li>
    <li>2</li>
    <li>3</li>
</ul>
```
```javascript
var newEle = document.createElement('li');
var textNode = document.createTextNode('insertNode');
newEle.appendChild(textNode);

var ul = document.getElementById('ul');
ul.insertBefore(newEle, ul.firstChild);
```
页面输出：
```
insertNode
1
2
3
```

## DOM查询
元素查询的API返回的的结果是DOM节点或者DOM节点的列表
```javascript
getElementById()  
getElementsByName()  
getElementsByTagName()
getElementsByClassName()
querySelector()
querySelectorAll() 
```

querySelector() 方法返回匹配指定 CSS 选择器元素的第一个子元素， 该方法只返回匹配指定选择器的第一个元素。如果要返回所有匹配元素，需要使用 querySelectorAll() 方法替代．
```javascipt
document.querySelector("#test"); // 获取到id名为test的首个元素
```
- querySelector系列方法与getElementBy系列方法对比
1. 得到的元素不是需要很麻烦的多次getElementBy..的话，尽量使用getElementBy系列方法,因为getElementBy系列执行速度更快。
2. 得到的元素需要很麻烦的多次getElementBy...组合才能得到的话使用querySelector，方便。
3. querySelector()选择的标签是静态的，也就是说在选中之后，能够一直保存，也就是脱离了被选择的成为副本。而getelementsBy系列方法是动态的，相互映射，在调用时，变化可以及时的反映在页面上。
```javascript
// 用 querySelector 操作元素
var ul = document.querySelector('ul');
var list = ul.querySelectorAll('li');
for (var i = 0; i < 3; i++) { // 创建3个新的li标签，添加到ul列表中
    ul.appendChild(document.createElement('li'));
}
console.log(list.length); // 3  
// 输出的是添加前li的数量3，而非此时li的总数量6
```
```javascript
var ul = document.getElementsByTagName('ul')[0];
var list = ul.getElementsByTagName('li');
for (var i = 0; i < 3; i++) { // 创建3个新的li标签，添加到ul列表中
    ul.appendChild(document.createElement('li'));
}
console.log(list.length); // 6
```

- 还有元素的DOM导航方法：
```javascript
// 获取父元素、父节点
var parent = ele.parentElement;
var parent = ele.parentNode;

// 获取子节点，子节点可以是任何一种节点，可以通过nodeType来判断
var nodes = ele.children;    

// 查询子元素
var els = ele.getElementsByTagName('li');
var els = ele.getElementsByClassName('test');

// 当前元素的第一个/最后一个子元素节点
var el = ele.firstElementChild;
var el = ele.lastElementChild;

// 下一个/上一个兄弟元素节点
var el = ele.nextElementSibling;
var el = ele.previousElementSibling;
```
## 属性操作

```
getAttribute(key)
setAttribute(key,value)
hasAttribute(key)
removeAttribute(key)  
```
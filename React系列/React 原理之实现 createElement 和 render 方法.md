## 前言

在 React 中，我们都知道可以写 jsx 代码会被编译成真正的 DOM 插入到要显示的页面上。这具体是怎么实现的，今天我们就自己动手做一下。

## 实现 createElement 方法

这个方法平时开发我们并不会用到，因为它是经 babel 编译后的代码，我们新建一个 React 项目，index.js 最简单的代码结构如下：

```javascript
import React from 'react'
import ReactDOM from 'react-dom'
ReactDOM.render(<h1 className='title'>Hello React</h1>, document.getElementById('root'))
```

这里就 jsx 会变编译成真正的 DOM ，把 html 代码拿到 babel 官网编译

![](https://user-gold-cdn.xitu.io/2020/6/6/17287948badb8312?w=1162&h=182&f=png&s=14232)

于是我们就看到了 React.createElement() 方法，但这只是调用这个方法，它具体做了什么返回什么我们还不知道，我们可以打印这个函数运行的结果：

```javascript
console.log(
  React.createElement(
    'h1',
    {
      className: 'title',
    },
    'Hello React'
  )
)
```

![](https://user-gold-cdn.xitu.io/2020/6/6/1728796fcf7d2054?w=567&h=181&f=png&s=12796)
返回的这个对象就是虚拟 DOM 了。

我们来分析它返回的对象参数，首先第一个是

- `$$typeof: REACT_ELEMENT_TYPE`

这个是 React 元素对象的标识属性

REACT_ELEMENT_TYPE 的值是一个 Symbol 类型，代表了一个独一无二的值。如果浏览器不支持 Symbol 类型，值就是一个二进制值。

> 为什么是 Symbol？主要防止 XSS 攻击伪造一个假的 React 组件。因为 JSON 中是不会存在 Symbol 类型的。

- key：这个比如循环中会用到这个 key 值
- props：传入的属性值，比如 id, className, style, children 等
- ref： DOM 的引用
- 剩下的是私有属性（本篇不展开讨论）

在本篇我们会用自己简单的方式实现这两个方法，而不是根据源码，所以实现上的方法只要能实现它的基本功能即可；有个基本概念在，以后再循序渐进学习源码。

而 createElement 中有三个参数，更确切说是 n 个参数：

- type：表示要渲染的元素类型。这里可以传入一个元素 Tag 名称，也可以传入一个组件（如 div span 等，也可以是是函数组件和类组件）
- props：创建 React 元素所需要的 props。
- childrens（可选参数）：要渲染元素的子元素，这里可以向后传入 n 个参数。可以为文本字符串，也可以为数组

初步 createElement 方法：

```javascript
// 创建 JSX 对象
function createElement(type, props, ...childrens) {
    return {
        type,
        props: {
          ...props,
          children: childrens.length <= 1 ? childrens[0] || '' : childrens,
        },
}
```

参数中 props 和 childrens 是并列关系，然后返回的 props 对象，里面包含了 children，所以我们需要再 props 里面添加 children 参数，然后根据 children 参数为一个或多个的可能在进行取值处理。

调用该方法：

```javascript
console.log(
  createElement(
    'h1',
    {
      className: 'title',
    },
    'Hello React'
  )
)
```

![](https://user-gold-cdn.xitu.io/2020/6/6/17287aecec9a3e91?w=237&h=116&f=png&s=4860)
除去其它本篇我们不讨论的属性，目前算是实现了一半；我们观察原来 React 自身方法输出的结果有 key, ref， 同输出的 props 也是并列关系，于是我们进一步作出处理

```javascript
function createElement(type, props, ...childrens) {
  let ref, key
  if ('ref' in props) {
    ref = props['ref']
    props['ref'] = undefined
  }
  if ('key' in props) {
    key = props['key']
    props['key'] = undefined
  }
  return {
    type,
    props: {
      ...props,
      children: childrens.length <= 1 ? childrens[0] || '' : childrens,
    },
    ref,
    key,
  }
}
```

同样的方式调用结果如下：

![](https://user-gold-cdn.xitu.io/2020/6/6/17287b33de749a06?w=453&h=145&f=png&s=8494)
如果添加多一些属性，我们来看看结果

```javascript
console.log(
  createElement(
    'div',
    { id: 'box', className: 'box', style: { color: 'red' }, key: '20' },
    'this is text',
    createElement('h2', { className: 'title' }, 'hello'),
    createElement('div', { className: 'content' }, 'Hi')
  )
)
```

![](https://user-gold-cdn.xitu.io/2020/6/6/17287c9de8368ddd?w=467&h=280&f=png&s=17373)
用了这种比较粗鲁的方式添加，设置为 undefined 在实现 render 方法的时候我们会根据这个忽略 props 内部的 key 和 props 属性，这里就实现了最基本的 createElement 方法了。

## 实现 render 方法

render 方法的第一个参数接收的是 createElement 返回的对象，也就是虚拟 DOM；
第二个参数则是挂载的目标 DOM。同样的做法，我们用 babel 编译来看：

![](https://user-gold-cdn.xitu.io/2020/6/6/17287ca773a9b742?w=1259&h=362&f=png&s=43471)
执行后，就被挂在到页面了

![](https://user-gold-cdn.xitu.io/2020/6/6/17287cc414f3731c?w=343&h=122&f=png&s=6145)

实现代码如下：

```javascript
/*
 * 功能：把创建的对象生成对应的DOM元素，最后插入到页面中
 * objJSX： createElement 返回的 JSX 对象
 * container：挂载的容器，如 document.getElementById('root')
 */
function render(objJSX, container) {
  let { type, props } = objJSX
  let newElement = document.createElement(type)
  for (let attr in props) {
    // 遍历传入的 props 属性
    if (!props.hasOwnProperty(attr)) break // 不是私有的直接结束遍历
    let value = props[attr] // >如果当前属性没有值,直接不处理即可
    if (value == undefined) continue // NULL OR UNDEFINED

    // 对几个特殊属性单独设置
    switch (attr.toUpperCase()) {
      case 'ID':
        newElement.setAttribute('id', value)
        break
      case 'CLASSNAME':
        newElement.setAttribute('class', value)
        break
      case 'STYLE': // 传入的行内样式 style 是个对象，故需遍历赋值
        for (let styleAttr in value) {
          if (value.hasOwnProperty(styleAttr)) {
            newElement['style'][styleAttr] = value[styleAttr]
          }
        }
        break
      case 'CHILDREN':
        /*
         * 可能是一个值：可能是字符串也可能是一个JSX对象
         * 可能是一个数组：数组中的每一项可能是字符串也可能是JSX对象
         */
        // 首先把一个值也变为数组，这样后期统一操作数组即可
        !(value instanceof Array) ? (value = [value]) : null
        value.forEach((item, index) => {
          // 验证ITEM是什么类型的：如果是字符串就是创建文本节点，如果是对象，我们需要再次执行RENDER方法，把创建的元素放到最开始创建的大盒子中
          if (typeof item === 'string') {
            let text = document.createTextNode(item)
            newElement.appendChild(text)
          } else {
            render(item, newElement)
          }
        })
        break
      default:
        newElement.setAttribute(attr, value)
    }
  }
  container.appendChild(newElement)
}
```

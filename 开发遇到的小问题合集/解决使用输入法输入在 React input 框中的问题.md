# 解决使用输入法输入在 React input 框中的问题

## 问题

在使用 React 绑定 input 输入框的 onChange 方法时，如果使用中文输入法（或者其他输入法），会出现一个问题：还在输入拼音的时候，onChange 方法已经触发了，如下，即输入过程就已经触发了多次 onChange 方法。如果 onChange 方法有较为复杂的逻辑，就可能会带来一些用户体验或者逻辑的问题。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/26ab509fe10d46bc97f80ee6f8728336~tplv-k3u1fbpfcp-zoom-1.image)

## 原因

只要有按下键盘的动作，就会触发 onChange 方法，如果输入英文就没什么问题，但使用中/日/韩等输入法的话，比如输入中文拼音已经开始在触发 onChange 事件了。

## 需求及解决方案

- 需求：等到选择确认输入中文后，才让它触发 onChange 方法的后续操作，如改变 value 的值。

- 解决方案：使用 compositionEvent 事件来解决。

> DOM 接口 CompositionEvent 表示用户间接输入文本（如使用输入法）时发生的事件。此接口的常用事件有 compositionstart, compositionupdate 和 compositionend

## CompositionEvent 事件介绍

- compositionstart

当用户使用输入法如拼音输入汉字时，这个事件就会被触发，即是在用户开始非直接输入的时候触发，在非直接输入的时候结束，整个过程只触发了一次。

- compositionupdate

事件触发于字符被输入到一段文字的时候，如在用户开始输入拼音到确定结束的过程都会触发该事件。

- compositionend

当文本段落的组成完成或取消时, compositionend 事件将被触发 。如用户点击拼音输入法选词确定后，则触发了该事件，此时是直接输入了，整个过程只触发了一次。

可以知道这三个事件就把我们输入中文拼音的三个过程进行了拆分。

## 实现

我们通过监听输入法开始输入到结束的事件，即是去监听`compositionstart`和`compositionend`方法，通过设置一个变量，在两个方法里面设置 true/false，来判断是否处在中文输入拼音这个过程当中，如果是，则不触发 onChange 后续事件。

这个未必是优化。搜索框提示的一个很重要用处，不是帮助省出来那么一点打字的时间，而是为了提示打字人应该写什么。很多时候打字者只有一个模糊的需求，全靠搜索框提示才能明白自己真正想搜什么。而这时候对拼音进行搜索提示，不说提醒用户该拼音词也可以搜到结果，单说在某种可能的用况下会避免用户搜索整个输入法却找不到对应汉字（因为输入错误拼音）这点就非常好用了。

如下代码，我们通过设置个对照：value1 对应的为正常 onChange 操作，value2 对应的则是做了输入法处理

```javascript
import React, { Component } from 'react'
import './style.css'

let isComposition = false

class TestComposition extends Component {
  constructor(props) {
    super(props)
    this.state = {
      value1: '',
      value2: '',
    }
    this.handleChange1 = this.handleChange1.bind(this)
    this.handleChange2 = this.handleChange2.bind(this)
    this.handleComposition = this.handleComposition.bind(this)
  }

  handleChange1 = ev => {
    this.setState({
      value1: ev.target.value,
    })
  }

  handleChange2 = ev => {
    // 未使用输入法或使用输入法完毕才能触发
    if (!isComposition) {
      this.setState({
        value2: ev.target.value,
      })
    }
  }

  handleComposition(ev) {
    if (ev.type === 'compositionend') {
      isComposition = false
    } else {
      isComposition = true
    }
  }

  render() {
    return (
      <div>
        <input type='text' onChange={this.handleChange1} />
        <span>{this.state.value1}</span>
        <input
          type='text'
          onChange={this.handleChange2}
          onCompositionStart={this.handleComposition}
          onCompositionEnd={this.handleComposition}
          placeholder='使用了composition的input框'
        />
        <span>{this.state.value2}</span>
      </div>
    )
  }
}

export default TestComposition
```

那么这样就能解决了吗？还不行。

其他浏览器不会有问题，但谷歌浏览器却不行。这里要注意的是谷歌浏览器跟其他浏览器的执行顺序不同：

> 谷歌浏览器： compositionstart -> onChange -> compositionend

> 其他浏览器： compositionstart -> compositionend -> onChange

所以上述代码运行在谷歌浏览器的话，一开始中文输入我们就将 isComposition 设置为 true，最后一步 compositionend 方法我们才将 isComposition 恢复为 true,而 onChange 已经执行完了， 按这个逻辑中文输入法打字都改不了 input 的 value 值

```javascript
if (!isComposition) {
  // isComposition为false 才可以执行onChange后续逻辑
  this.setState({
    value2: ev.target.value,
  })
}
```

所以我们需要对谷歌浏览器做一次特别处理：

1. 判断是否为谷歌浏览器
2. 如果是，则在 compositionend 方法最后再执行一次 onChange 方法

最后附上完整代码：

```javascript
import React, { Component } from 'react'

let isComposition = false
const isChrome = navigator.userAgent.indexOf('Chrome') > -1

class TestComposition extends Component {
  constructor(props) {
    super(props)
    this.state = {
      value1: '',
      value2: '',
    }
    this.handleChange1 = this.handleChange1.bind(this)
    this.handleChange2 = this.handleChange2.bind(this)
    this.handleComposition = this.handleComposition.bind(this)
  }

  handleChange1 = ev => {
    this.setState({
      value1: ev.target.value,
    })
  }

  handleChange2 = ev => {
    // 未使用输入法或使用输入法完毕才能触发
    if (!isComposition) {
      this.setState({
        value2: ev.target.value,
      })
    }
  }

  handleComposition(ev) {
    if (ev.type === 'compositionend') {
      isComposition = false
      if (!isComposition && isChrome) {
        this.handleChange2(ev)
      }
    } else {
      isComposition = true
    }
  }

  render() {
    return (
      <div>
        <input type='text' onChange={this.handleChange1} />
        <span>{this.state.value1}</span>
        <input
          type='text'
          onChange={this.handleChange2}
          onCompositionStart={this.handleComposition}
          onCompositionEnd={this.handleComposition}
          placeholder='使用了composition的input框'
        />
        <span>{this.state.value2}</span>
      </div>
    )
  }
}

export default TestComposition
```

效果：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ac3e7f8896b4d608c1ae46ee22f0866~tplv-k3u1fbpfcp-zoom-1.image)

注：

- Vue 自带解决了上述问题，v-model 有用到 compositionEvent 事件
- 该做法视乎项目需求场景，因为如果有需要进行搜索中文拼音过程也要提示英文单词的来说，那就没必要了
- 也可以使用 ref 的方式进行绑定，上述只是其中一种解决方式

参考：

- [MDN：CompositionEvent](https://developer.mozilla.org/en-US/docs/Web/API/CompositionEvent)
- [中文输入法在 React 文本输入框的特殊处理](https://www.yukapril.com/2019/11/11/react-input-composition.html)

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，鼓励我继续写作吧~

# React 的状态管理库 —— Recoil

## 为什么使用 Recoil

在学一样东西之前，我们得了解它为什么会诞生，或者是它解决了什么问题。

Recoil 是由 Facebook 推出的一个全新的、实验性的 JavaScript 状态管理库，它解决了使用现有 Context API 在构建大型应用时所面临的很多问题。

使用 React 内置的状态管理能力有这样一些局限性：

- 组件间的状态共享只能通过将 state 提升至它们的公共祖先来实现，但这样做可能导致重新渲染一颗巨大的组件树。
- Context 只能存储单一值，无法存储多个各自拥有 Consumer 的值的集合。
- 以上两种方式都很难将组件树的顶层（state 必须存在的地方）与叶子组件 (使用 state 的地方) 进行代码分割。

尽管像 Redux 和 MobX 这样的库能够确保应用的状态保持一致，但是对于很多应用来讲，它们所带来的开销是难以估量的。

Redux、Mobx 本身并不是 React 库，我们是借助这些库的能力来实现状态管理。像 Redux 它本身虽然提供了强大的状态管理能力，但是使用的成本非常高，你还需要编写大量冗长的代码，另外像异步处理或缓存计算也不是这些库本身的能力，甚至需要借助其他的外部库。

并且，它们并不能访问 React 内部的调度程序，而 Recoil 在后台使用 React 本身的状态，在未来还能提供并发模式这样的能力

## 使用

先来看看 Recoil 是怎么使用的

### 根组件

使用 recoil 状态的组件需要使用 RecoilRoot 包裹起来，一般是根组件直接包裹

```js
import React from 'react'
import ReactDOM from 'react-dom'
import { RecoilRoot } from 'recoil'
import App from './App'

ReactDOM.render(
  <RecoilRoot>
    <App />
  </RecoilRoot>,
  document.getElementById('root')
)
```

### Atoms

Atom 是最小状态单元。它们可以被订阅和更新：当它更新时，所有订阅它的组件都会使用新数据重绘；它可以在运行时创建；它也可以在局部状态使用；同一个 Atom 可以被多个组件使用与共享。

相比 Redux 维护的全局 Store，Recoil 则是采用分散管理原子状态的设计模式，方便进行代码分割。

Atom 和传统的 state 不同，它可以被任何组件订阅，当一个 Atom 被更新时，每个被订阅的组件都会用新的值来重新渲染。

所以 Atom 相当于一组 state 的集合，改变一个 Atom 只会渲染特定的子组件，并不会让整个父组件重新渲染。

```js
import { atom } from 'recoil'

export const todoList = atom({
  key: 'todoList',
  default: [],
})
```

要创建一个 Atom ，必须要提供一个 key ，其必须在 RecoilRoot 作用域中是唯一的，并且要提供一个默认值，默认值可以是一个静态值、函数甚至可以是一个异步函数。

### API

Recoil 采用 Hooks 方式订阅和更新状态，常用的 API 如下：

**useRecoilState**

类似 useState 的一个 Hook，可以对 atom 进行读写

```js
import React, { useState } from 'react'
import { useRecoilState } from 'recoil'
import { TodoListStore } from './store'

export default function OperatePanel() {
  const [inputValue, setInputValue] = useState('')
  const [todoListData, setTodoListData] = useRecoilState(TodoListStore.todoList)

  const addItem = () => {
    const newList = [...todoListData, { thing: inputValue, isComplete: false }]
    setTodoListData(newList)
    setInputValue('')
  }

  return (
    <div>
      <h3>OperatePanel Page</h3>
      <input type='text' value={inputValue} onChange={e => setInputValue(e.target.value)} />
      <button onClick={addItem}>添加</button>
    </div>
  )
}
```

**useSetRecoilState**

只获取 setter 函数，不会返回 state 的值，如果只使用了这个函数，状态变化不会导致组件重新渲染

```js
import React from 'react'
import { useSetRecoilState } from 'recoil'
import { TodoListStore } from './store'
export default function SetPanel() {
  const setTodoListData = useSetRecoilState(TodoListStore.todoList)

  const clearData = () => {
    setTodoListData([])
  }

  return (
    <div>
      <button onClick={clearData}>清空recoil的数组</button>
    </div>
  )
}
```

**useRecoilValue**

只返回 state 的值，不提供修改方法

```js
import React from 'react'
import { useRecoilValue } from 'recoil'
import { TodoListStore } from './store'
export default function ShowPanel() {
  const todoListData = useRecoilValue(TodoListStore.todoList)
  return (
    <div>
      <h3>ShowPanel Page</h3>
      recoil中获取结果展示：
      {todoListData.map((item, index) => {
        return <div key={index}>{item.thing}</div>
      })}
    </div>
  )
}
```

**selector**

selector 表示一段派生状态，它使我们能够建立依赖于其他 atom 的状态。它有一个强制性的 get 函数，其作用与 redux 的 reselect 或 MobX 的 computed 类似。

selector 是一个纯函数：对于给定的一组输入，它们应始终产生相同的结果（至少在应用程序的生命周期内）。这一点很重要，因为选择器可能会执行一次或多次，可能会重新启动并可能会被缓存。

```js
export const completeCountSelector = selector({
  key: 'completeCountSelector',
  get({ get }) {
    const completedList = get(todoList)
    return completedList.filter(item => item.isComplete).length
  },
})
```

selector 还支持异步函数，可以将一个 Promise 作为返回值

## 结语

除了 Facebook，暂时还没有看到有哪些网站已经用了 Recoil。

Recoil 的核心概念都很简单，没有 Redux 那么绕的概念，也不需要写一堆像 action、reducer 之类的模板文件，基于 Hooks 的 API 以及它的直观性。与其他一些库相比，Recoil 的 API 比大多数库更容易，让开发更加简单。

我们现在的项目使用了 Recoil，目前感受是简化版的 Context API，使用较 Redux 简单，暂时没有发现能像 Redux 生态那样方便的时间回溯功能，后续使用有待继续观察。

[本文案例代码](https://github.com/Jacky-Summer/recoil-tutorial)

<br>

- [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)
  觉得不错的话欢迎 star，给我一点鼓励继续写作吧~

# 手写模拟实现 React Hooks

## 前言

首先使用`create-react-app`新建个项目，然后在`index.js`写我们的代码，阅读本文前需要知道常用 React Hooks 的基本用法。

## useState

```javascript
import React from 'react'
import ReactDOM from 'react-dom'

let memorizedState
const useState = initialState => {
  memorizedState = memorizedState || initialState // 初始化
  const setState = newState => {
    memorizedState = newState
    render() // setState 之后触发重新渲染
  }
  return [memorizedState, setState]
}

const App = () => {
  const [count1, setCount1] = useState(0)

  return (
    <div>
      <div>
        <h2>useState： {count1}</h2>
        <button
          onClick={() => {
            setCount1(count1 + 1)
          }}
        >
          添加count1
        </button>
      </div>
    </div>
  )
}

const render = () => {
  ReactDOM.render(<App />, document.getElementById('root'))
}
render()
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd76376f5288400697f3b2b6f29c2034~tplv-k3u1fbpfcp-zoom-1.image)

但到这里会有一个问题，就是当增加第二个 useState 的时候会发现改变两个 state 都是改同一个值，同步变化，于是，我们通过使用数组的方式下标来方式来区分。

```javascript
import React from 'react'
import ReactDOM from 'react-dom'

let memorizedState = [] // 通过数组形式存储有关使用hook的值
let index = 0 // 通过下标记录 state 的值
const useState = initialState => {
  let currentIndex = index
  memorizedState[currentIndex] = memorizedState[index] || initialState
  const setState = newState => {
    memorizedState[currentIndex] = newState
    render() // setState 之后触发重新渲染
  }
  return [memorizedState[index++], setState]
}

const App = () => {
  const [count1, setCount1] = useState(0)
  const [count2, setCount2] = useState(10)

  return (
    <div>
      <div>
        <h2>
          useState： {count1}--{count2}
        </h2>
        <button
          onClick={() => {
            setCount1(count1 + 1)
          }}
        >
          添加count1
        </button>
        <button
          onClick={() => {
            setCount2(count2 + 10)
          }}
        >
          添加count2
        </button>
      </div>
    </div>
  )
}

const render = () => {
  index = 0
  ReactDOM.render(<App />, document.getElementById('root'))
}
render()
```

这样就实现效果了：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f9fd70c764d4b16874dc4f36cd0663a~tplv-k3u1fbpfcp-zoom-1.image)

分析：

- 第一次页面渲染的时候，根据 `useState` 顺序，声明了 count1 和 count2 两个 state，并按下标顺序依次存入数组中
- 当调用`setState`的时候，更新 count1/count2 的值，触发重新渲染的时候，index 被重置为 0。然后又重新按 `useState` 的声明顺序，依次拿出最新 state 的值；由此也可见为什么当我们使用 hook 时，要注意点 hook 不能在循环、判断语句内部使用，要声明在组件顶部。

`memorizedState`这个数组我们下面实现部分 hook 都会用到，现在`memorizedState`数组长度为 2，依次存放着两个使用`useState`后返回的 state 值；

```
0: 0
1: 10
```

每次更改数据后，调用`render`方法，`App`函数组件重新渲染，又重新调用`useState`，但外部变量`memorizedState`之前已经依次下标记录下了 state 的值，故重新渲染是直接赋值之前的 state 值做初始值。知道这个做法，下面的`useEffect`,`useCallback`,`useMemo`也是这个原理。

当然实际源码，`useState`是用链表记录顺序的，这里我们只是模拟效果。

## useReducer

useReducer 接受 Reducer 函数和状态的初始值作为参数，返回一个数组。数组的第一个成员是状态的当前值，第二个成员是发送 action 的 dispatch 函数。

```javascript
let reducerState

const useReducer = (reducer, initialArg, init) => {
  let initialState
  if (init !== undefined) {
    initialState = init(initialArg) // 初始化函数赋值
  } else {
    initialState = initialArg
  }

  const dispatch = action => {
    reducerState = reducer(reducerState, action)
    render()
  }
  reducerState = reducerState || initialState
  return [reducerState, dispatch]
}

const init = initialNum => {
  return { num: initialNum }
}

const reducer = (state, action) => {
  switch (action.type) {
    case 'increment':
      return { num: state.num + 1 }
    case 'decrement':
      return { num: state.num - 1 }
    default:
      throw new Error()
  }
}

const App = () => {
  const [state, dispatch] = useReducer(reducer, 20, init)

  return (
    <div>
      <div>
        <h2>useReducer：{state.num}</h2>
        <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
        <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      </div>
    </div>
  )
}
```

## useEffect

对于 useEffect 钩子，当没有依赖值的时候，很容易想到雏形代码：

```javascript
const useEffect = (callback, dependencies) => {
  if (!dependencies) {
    // 没有添加依赖项则每次执行，添加依赖项为空数组
    callback()
  }
}
```

但如果有依赖 state 值，即是我们使用 useState 后返回的值，这部分我们就需要用上方定义的数组 memorizedState 来记录

```javascript
let memorizedState = [] // 存放 hook
let index = 0 // hook 数组下标位置
/**
 * useState 实现
 */
const useState = initialState => {
  let currentIndex = index
  memorizedState[currentIndex] = memorizedState[index] || initialState
  const setState = newState => {
    memorizedState[currentIndex] = newState
    render() // setState 之后触发重新渲染
  }
  return [memorizedState[index++], setState]
}

/**
 * useEffect 实现
 */
const useEffect = (callback, dependencies) => {
  if (memorizedState[index]) {
    // 不是第一次执行
    let lastDependencies = memorizedState[index] // 依赖项数组
    let hasChanged = !dependencies.every((item, index) => item === lastDependencies[index]) // 循环遍历依赖项是否与上次的值相同
    if (hasChanged) {
      // 依赖项有改变就执行 callback 函数
      memorizedState[index++] = dependencies
      setTimeout(callback) // 设置宏任务，在组件render之后再执行
    } else {
      index++ // 每个hook占据一个下标位置，防止顺序错乱
    }
  } else {
    // 第一次执行
    memorizedState[index++] = dependencies
    setTimeout(callback)
  }
}

const App = () => {
  const [count1, setCount1] = useState(0)
  const [count2, setCount2] = useState(10)

  useEffect(() => {
    console.log('useEffect1')
  }, [count1, count2])

  useEffect(() => {
    console.log('useEffect2')
  }, [count1])

  return (
    <div>
      <div>
        <h2>
          useState： {count1}--{count2}
        </h2>
        <button
          onClick={() => {
            setCount1(count1 + 1)
          }}
        >
          添加count1
        </button>
        <button
          onClick={() => {
            setCount2(count2 + 10)
          }}
        >
          添加count2
        </button>
      </div>
    </div>
  )
}

const render = () => {
  index = 0
  ReactDOM.render(<App />, document.getElementById('root'))
}
render()
```

程序第一次执行完毕后，memorizedState 数组值如下

```
0: 0
1: 10
2: [0, 10]
3: [0]
```

上述代码回调函数执行，本来我们可以用`callback()`执行即可，但因为`useEffect`在渲染时是异步执行，并且要等到浏览器将所有变化渲染到屏幕后才会被执行；

因为是异步且等页面渲染完毕才执行，根据对 JS 事件循环的理解，我们想要它异步执行任务，就在此创建一个宏任务`setTimeout(callback)`让它进入宏任务队列等待执行，当然这其中具体的渲染过程我这里就不细说了。

还有一个 hook 是`useLayoutEffect`，除了执行回调的两处地方代码实现不同，其他代码相同，`callback`这里我用微任务`Promise.resolve().then(callback)`，把函数执行加入微任务队列。

因为`useLayoutEffect`在渲染时是同步执行，会在所有的 DOM 变更之后同步调用，一般可以使用它来读取 DOM 布局并同步触发重渲染。在浏览器执行绘制之前，`useLayoutEffect` 将被同步刷新。

怎么证明呢？如果你在`useLayoutEffect`加了死循环，然后重新打开网页，你会发现看不到页面渲染的内容就进入死循环了；而如果是`useEffect`的话，会看到页面渲染完成后才进入死循环。

```javascript
useLayoutEffect(() => {
  while (true) {}
}, [])
```

## useCallback

`useCallback`和`useMemo`会在组件第一次渲染的时候执行，之后会在其依赖的变量发生改变时再次执行；并且这两个 hooks 都返回缓存的值，useMemo 返回缓存计算数据的值，`useCallback`返回缓存函数的引用。

```javascript
const useCallback = (callback, dependencies) => {
  if (memorizedState[index]) {
    // 不是第一次执行
    let [lastCallback, lastDependencies] = memorizedState[index]

    let hasChanged = !dependencies.every((item, index) => item === lastDependencies[index]) // 判断依赖值是否发生改变
    if (hasChanged) {
      memorizedState[index++] = [callback, dependencies]
      return callback
    } else {
      index++
      return lastCallback // 依赖值不变，返回上次缓存的函数
    }
  } else {
    // 第一次执行
    memorizedState[index++] = [callback, dependencies]
    return callback
  }
}
```

## useMemo

而`useMemo`实现与`useCallback`也很类似，只不过它返回的函数执行后的计算返回值，直接把函数执行了。

```javascript
const useMemo = (memoFn, dependencies) => {
  if (memorizedState[index]) {
    // 不是第一次执行
    let [lastMemo, lastDependencies] = memorizedState[index]

    let hasChanged = !dependencies.every((item, index) => item === lastDependencies[index])
    if (hasChanged) {
      memorizedState[index++] = [memoFn(), dependencies]
      return memoFn()
    } else {
      index++
      return lastMemo
    }
  } else {
    // 第一次执行
    memorizedState[index++] = [memoFn(), dependencies]
    return memoFn()
  }
}
```

## useContext

代码出乎意料的少吧...

```
const useContext = context => {
  return context._currentValue
}
```

## useRef

`useRef` 返回一个可变的 `ref` 对象，其 `.current` 属性被初始化为传入的参数（initialValue）。返回的 `ref` 对象在组件的整个生命周期内保持不变。

```javascript
let lastRef
const useRef = value => {
  lastRef = lastRef || { current: value }
  return lastRef
}
```

后面几个例子我就没展示出 Demo 了，附上 Github 地址：[上述 hooks 实现和案例](https://github.com/Jacky-Summer/mini-react-hooks)
<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~

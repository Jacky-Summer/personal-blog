# 实现一个迷你Redux（基础版）

## 前言

本文从将从Redux原理出发，一步步自己实现一个简单的Redux，主要目的是了解Redux内部之间的联系。看本文之前先要知道Redux是怎么用的，对Redux用法不会讲解太多。

## Redux介绍

首先要知道的是，Redux 和 React 没有关系，Redux 可以用在任何框架中。
Redux 是一个JavaScript 状态管理器，是一种新型的前端“架构模式”。

还有通常与redux一起用的一个库——react-redux， 它就是把 Redux 这种架构模式和 React.js 结合起来的一个库，就是 Redux 架构在 React.js 中的体现。

## 设计思想

1. Web 应用是一个状态机，视图与状态是一一对应的。
2. 所有的状态，保存在一个对象里面。

## 何时使用Redux

1. 用户的使用方式复杂
2. 不同身份的用户有不同的使用方式（比如普通用户和管理员）
3. 多个用户之间可以协作
4. 与服务器大量交互，或者使用了WebSocket
5. View要从多个来源获取数据

从组件的角度看：
1. 某个组件的状态，需要共享
2. 某个状态需要在任何地方都可以拿到
3. 一个组件需要改变全局状态
4. 一个组件需要改变另一个组件的状态

## Redux工作流程

![](https://user-gold-cdn.xitu.io/2020/4/18/1718ce1ceca204e3?w=898&h=421&f=png&s=148790)
1. Redux 将整个应用状态(state)存储到一个地方(通常我们称其为 store)
2. 当我们需要修改状态时，必须派发(dispatch)一个 action( action 是一个带有 type 字段的对象)
3. 专门的状态处理函数 reducer 接收旧的 state 和 action ，并会返回一个新的 state
4. 通过 subscribe 设置订阅，每次派发动作时，通知所有的订阅者。

从这个流程中可以看出，Redux 的核心就是一个 观察者 模式。一旦 store 发生了变化就会通知所有的订阅者，视图(在这里是react组件)接收到通知之后会进行重新渲染。

## Redux案例

为了简化说明，我用和官网差不多的例子改写来作案例
[官网redux demo](https://redux.js.org/introduction/getting-started)

新建一个文件redux.js，然后直接引入，观察控制台输出
```javascript
import { createStore } from 'redux'
const defaultState = {
    value: 10
}
// reducer处理函数
function reducer (state = defaultState, action) {
    console.log(state, action)
    switch (action.type) {
        case 'INCREMENT':
            return {
                ...state,
                value: state.value + 1
            }
        case 'DECREMENT':
            return {
                ...state,
                value: state.value - 1
            }
        default:
            return state
    }
}
const store = createStore(reducer)

const init = store.getState()
console.log(`一开始数字为：${init.value}`)

function listener () {
    const current = store.getState()
    console.log(`当前数字为：${current.value}`)
}
store.subscribe(listener) // 监听state的改变

store.dispatch({ type: 'INCREMENT' })
// 当前数字为：11
store.dispatch({ type: 'INCREMENT' })
// 当前数字为：12
store.dispatch({ type: 'DECREMENT' })
// 当前数字为：11

export default store
```
输出结果：
```
{value: 10} {type: "@@redux/INIT1.a.7.g.7.t"}
一开始数字为：10
{value: 10} {type: "INCREMENT"}
当前数字为：11
{value: 11} {type: "INCREMENT"}
当前数字为：12
{value: 12} {type: "DECREMENT"}
当前数字为：11
```
所有对数据的操作必须通过 dispatch 函数，它接受一个参数action，action是一个普通的JavaScript对象，action必须包含一个`type`字段，告诉它要修改什么，只有它允许才能修改。

在每次调用进来reducer函数我们都打印了state和action，我们手动通过store.dispatch方法派发了三次action，但你会发现输出了四次.这是因为Redux内部初始化就自动执行了一次dispatch方法，可以看到第一次执行它的type对我们数据来说是没有影响的（因为type取值`@@redux/INIT1.a.7.g.7.t`，我们自己redux的数据type不会取名成这个样子，所以不会跟它重复），即默认输出state值

## 动手实现一个Redux

1. 首先要手动实现一个Redux之前，先看看有上述代码涉及到Redux什么方法，首先引入了
```
import { createStore } from 'redux'
// 传入reducer
const store = createStore(reducer) 
```
createStore 会返回一个对象，这个对象包含三个方法，于是我们可以列出Redux雏形。

新建mini-redux.js
```
export function createStore (reducer) {

    const getState = () => {}
    const subscribe = () => {}
    const dispatch = () => {}
    
    return { 
        getState, 
        subscribe, 
        dispatch 
    }
}
```

2. store.getState()用于获取 state 数据，其实就是简单地把 state 参数返回。于是
```javascript
export function createStore (reducer) {
    let currentState = {}
    const getState = () => currentState
    
    return { 
        getState
    }
}
```
3. dispatch方法会接收一个action，执行会调用 `reducer` 返回一个新状态
```javascript
export function createStore (reducer) {
    let currentState = {}
    const getState = () => currentState
    const dispatch = (action) => {
        currentState = reducer(currentState, action) // 覆盖原来的state
    }
    return { 
        getState,
        dispatch
    }
}
```

4. 通过 subscribe 设置监听函数（设置订阅），一旦 state 发生变化，就自动执行这个函数（通知所有的订阅者）。

怎么实现呢？我们可以直接使用subscribe函数把你要监听的事件添加到数组, 然后执行dispatch方法的时候把listeners数组的监听函数给执行一遍。
```javascript
export function createStore (reducer) {
    let currentState = {}
    let currentListeners = [] // 监听函数，可添加多个
    
    const getState = () => currentState
    const subscribe = (listener) => {
        currentListeners.push(listener)
    }
    const dispatch = (action) => {
        currentState = reducer(currentState, action) // 覆盖原来的state
        currentListeners.forEach(listener => listener())
    }
    return { 
        getState,
        subscribe,
        dispatch
    }
}
```
翻开一开始我们那个Redux例子，其实就是把`store.getState()`添加进来，dispatch派发一个action后，reducer执行返回新的state，并执行了监听函数`store.getState()`，state的值就发生变化了。
```javascript
function listener () {
    const current = store.getState()
    console.log(`当前数字为：${current.value}`)
}
store.subscribe(listener) // 监听state的改变
```
上述代码，跟React依然没有关系，只是纯属Redux例子。但想一想当我们把Redux和React一起用的时候，还会多做这么一步。
```javascript
 constructor(props) {
    super(props)
    this.state = store.getState()
    this.storeChange = this.storeChange.bind(this)
    store.subscribe(this.storeChange)
}

storeChange () {
    this.setState(store.getState())
}
```
在React里面监听的方法，还要用`this.setState()`, 这是因为React中state的改变必须依赖于`this.setState`方法。所以对于 React 项目，就是组件的render方法或setState方法放入listen（监听函数），才会实现视图的自动渲染，改变页面中的state值。

最后一步，注意我们上面说的，当初始化的时候，dispatch会先自动执行一次，继续改代码
```javascript
export function createStore (reducer) {
    let currentState = {}
    let currentListeners = [] // 监听器，可监听多个事件
    
    const getState = () => currentState

    const subscribe = (listener) => {
        currentListeners.push(listener)
    }

    const dispatch = (action) => {
        currentState = reducer(currentState, action) // 覆盖原来的state
        currentListeners.forEach(listener => listener())
    }
    // 尽量写得复杂，使不会与我们自定义的action有重复可能
    dispatch({ type: '@@mini-redux/~GSDG4%FDG#*&' })
    return { 
        getState, 
        subscribe, 
        dispatch 
    }
}
```
写到这里，我们把引入的redux替换我们写的文件
```
import { createStore } from './mini-redux'
```
当我们执行的时候，发现结果并不如我们所愿：
```
{} {type: "@@mini-redux/~GSDG4%FDG#*&"}
一开始数字为：undefined
{} {type: "INCREMENT"}
当前数字为：NaN
{type: "INCREMENT"}
当前数字为：NaN
{value: NaN} {type: "DECREMENT"}
当前数字为：NaN
```
这个怎么回事呢？因为我们写的redux一开始就给state赋值为{}，在事实state初始值是由外部传入的，通常我们自己写的时候会设置默认值
```javascript
const defaultState = {
    value: 10
}
function reducer (state = defaultState, action) {
    switch (action.type) {
        // ...
        default:
            return state
    }
}
```
但在我们Redux实现中却把它手动置为空对象，在这里我们暂时解决方法就是不给它赋值，让它为undefined，这样reducer的默认参数就会生效。redux初始化第一次dispatch时，就会让它自动赋值为reducer传入的第一个参数state默认值（ES6函数默认赋值），所以修改如下：
```javascript
export function createStore (reducer) {
    let currentState 
    let currentListeners = [] // 监听器，可监听多个事件
    
    const getState = () => currentState

    const subscribe = (listener) => {
        currentListeners.push(listener)
    }

    const dispatch = (action) => {
        currentState = reducer(currentState, action) // 覆盖原来的state
        currentListeners.forEach(listener => listener())
    }
    dispatch({ type: '@@mini-redux/~GSDG4%FDG#*&' })
    return { 
        getState, 
        subscribe, 
        dispatch 
    }
}
```
这个mini-redux.js，我们就可以实现跟原来的redux完全一样的输出效果了。

## 完善Redux
接下来我们继续补充知识点
1. createStore实际有三个参数，即
```javascript
createStore(reducer, [preloadedState], enhancer)
```

第二个参数 `[preloadedState] (any)`是可选的: initial state

第三个参数`enhancer(function)`也是可选的：用于添加中间件的

通常情况下，通过 preloadedState 指定的 state 优先级要高于通过 reducer 指定的 state。这种机制的存在允许我们在 reducer 可以通过指明默认参数来指定初始数据，而且还为通过服务端或者其它机制注入数据到 store 中提供了可能。

第三个参数我们下篇会说，先继续完善一下代码，我们需要对第二个和第三个可选参数进行判断。
```javascript
export function createStore (reducer, preloadedState, enhancer) {

    // 当第二个参数没有传preloadedState，而直接传function的话，就会直接把这个function当成enhancer
    if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
        enhancer = preloadedState
        preloadedState = undefined
    }
    // 当第三个参数传了但不是function也会报错
    if (typeof enhancer !== 'undefined') {
        if (typeof enhancer !== 'function') {
            throw new Error('Expected the enhancer to be a function.')
        }
        return enhancer(createStore)(reducer, preloadedState)
    }
    // reducer必须为函数
    if (typeof reducer !== 'function') {
        throw new Error('Expected the reducer to be a function.')
    }

    let currentState = preloadedState // 第二个参数没传默认就是undefined赋给currentState
    let currentListeners = [] // 监听器，可监听多个事件
    
    // ...
}
```
关于第三个参数判断为什么返回
`return enhancer(createStore)(reducer, preloadedState)`我们下篇会说，这篇先忽略。
2. 我们实现了store.subscribe()方法，但还是不完整的，subscribe方法可以添加监听函数listener,它还有返回值，返回一个移除listener的函数；另外我们依然要对类型进行判断。

```javascript
export function createStore (reducer, preloadedState, enhancer) {
    // ...
    let currentListeners = [] // 监听器，可监听多个事件

    const subscribe = (listener) => {
        if (typeof listener !== 'function') {
            throw new Error('Expected listener to be a function.')
        }
        currentListeners.push(listener)
        // 通过filter过滤，执行的时候将之前本身已经添加进数组的事件名移除数组
        return () => {
            currentListeners = currentListeners.filter(l => l !== listener);
        }
    }
    // ...
}
```
也可以通过找数组下标的方式移除listener
```javascript
const subscribe = (listener) => {
    if (typeof listener !== 'function') {
        throw new Error('Expected listener to be a function.')
    }
    currentListeners.push(listener)
    // 通过filter过滤，执行的时候将之前本身已经添加进数组的事件名移除数组
    return () => {
        let index = currentListeners.indexOf(listener)
        currentListeners.splice(index, 1)
    }
}

```
移除listener实际就是取消订阅，用的方式如下：
```javascript
let unsubscribe = store.subscribe(() =>
  console.log(store.getState())
);

unsubscribe(); // 取消监听
```

3. diaptch方法执行完返回为action，然后我们同样需要为它作判断
```javascript
export function createStore (reducer, preloadedState, enhancer) {
    // ...
    let isDispatching = false
    const dispatch = (action) => {
        // 用于判断action是否为一个普通对象
        if (!isPlainObject(action)) {
            throw new Error('Actions must be plain objects. ')
        }
        // 防止多次dispatch请求同时改状态，一定是前面的dispatch结束之后，才dispatch下一个
        if (isDispatching) {
            throw new Error('Reducers may not dispatch actions.')
        }
    
        try {
            isDispatching = true
            currentState = reducer(currentState, action) // 覆盖原来的state
        } finally {
            isDispatching = false
        }
    
        currentListeners.forEach(listener => listener())
        return action
    }
}

// 用于判断一个值是否为一个普通的对象(普通对象即直接以字面量形式或调用 new Object() 所创建的对象)
export function isPlainObject(obj) {
    if (typeof obj !== 'object' || obj === null) return false

    let proto = obj
    while (Object.getPrototypeOf(proto) !== null) {
        proto = Object.getPrototypeOf(proto)
    }

    return Object.getPrototypeOf(obj) === proto
}

// ...
```
isPlainObject函数中通过 while 不断地判断 Object.getPrototypeOf(proto) !== null 并执行, 最终 proto 会指向 Object.prototype. 这时再判断 Object.getPrototypeOf(obj) === proto, 如果为 true 的话就代表 obj 是通过字面量或调用 new Object() 所创建的对象了。

保持action对象是简单对象的作用是方便reducer进行处理，不用处理其他的情况（比如function/class实例等）

至此，我们实现了最基本能用的Redux代码，下篇再继续完善Redux代码，最后放出基础版Redux所有代码：
```javascript
export function createStore (reducer, preloadedState, enhancer) {

    // 当第二个参数没有传preloadedState，而直接传function的话，就会直接把这个function当成enhancer
    if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
        enhancer = preloadedState 
        preloadedState = undefined
    }
    // 当第三个参数传了但不是function也会报错
    if (typeof enhancer !== 'undefined') {
        if (typeof enhancer !== 'function') {
            throw new Error('Expected the enhancer to be a function.')
        }
        return enhancer(createStore)(reducer, preloadedState)
    }
    // reducer必须为函数
    if (typeof reducer !== 'function') {
        throw new Error('Expected the reducer to be a function.')
    }

    let currentState = preloadedState // 第二个参数没传默认就是undefined赋给currentState
    let currentListeners = [] // 监听器，可监听多个事件
    let isDispatching = false
    
    const getState = () => currentState

    const subscribe = (listener) => {
        if (typeof listener !== 'function') {
            throw new Error('Expected listener to be a function.')
        }
        currentListeners.push(listener)
        // 通过filter过滤，执行的时候将之前本身已经添加进数组的事件名移除数组
        return () => {
            currentListeners = currentListeners.filter(l => l !== listener);
        }
    }

    const dispatch = (action) => {
        // 用于判断action是否为一个普通对象
        if (!isPlainObject(action)) {
            throw new Error('Actions must be plain objects. ')
        }
        // 防止多次dispatch请求同时改状态，一定是前面的dispatch结束之后，才dispatch下一个
        if (isDispatching) {
            throw new Error('Reducers may not dispatch actions.')
        }
    
        try {
            isDispatching = true
            currentState = reducer(currentState, action) // 覆盖原来的state
        } finally {
            isDispatching = false
        }

        currentListeners.forEach(listener => listener())
        return action
    }
    dispatch({ type: '@@mini-redux/~GSDG4%FDG#*&' })

    return { 
        getState, 
        subscribe, 
        dispatch 
    }
}

// 用于判断一个值是否为一个普通的对象(普通对象即直接以字面量形式或调用 new Object() 所创建的对象)
export function isPlainObject(obj) {
    if (typeof obj !== 'object' || obj === null) return false

    let proto = obj
    while (Object.getPrototypeOf(proto) !== null) {
        proto = Object.getPrototypeOf(proto)
    }

    return Object.getPrototypeOf(obj) === proto
}

```

参考资料：

[Redux 入门教程（一）：基本用法](http://www.ruanyifeng.com/blog/2016/09/redux_tutorial_part_one_basic_usages.html)

<br>

- ps： [个人技术博文Github仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎star，给我一点鼓励吧~

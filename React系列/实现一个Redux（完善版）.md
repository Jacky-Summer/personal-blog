# 实现一个 Redux（完善版）

上次我们已经写了 [实现一个迷你 Redux（基础版）](https://github.com/Jacky-Summer/personal-blog/blob/master/React%E7%B3%BB%E5%88%97/%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AA%E8%BF%B7%E4%BD%A0Redux%EF%BC%88%E5%9F%BA%E7%A1%80%E7%89%88%EF%BC%89.md) ，这次我们来继续完善 Redux，继续上篇的例子续写。

## 中间件

Redux 有个 API 是 applyMiddleware， 专门用来使用中间件的，首先我们得知道，它用来干嘛的。

### 为什么会需要中间件？

假设我们现在需要记录每次的 dispatch 前后 state 的记录， 那要怎么做呢？于是，简单粗暴的在第一个 dispatch 方法前后加代码

```javascript
console.log('prev state', store.getState())
console.log(action)
store.dispatch({ type: 'INCREMENT' })
console.log('next state', store.getState())
```

这部分运行结果：

```javascript
prev state {value: 10}
{type: "INCREMENT"}
当前数字为：11
next state {value: 11}
```

但加完发现情况不对，页面有多个 dispatch 的话，要这样写很多次，会产生大量重复代码。突然，又要加需求了，需要记录每次出错的原因,单独的功能要求如下：

```javascript
try {
  store.dispatch(action)
} catch (err) {
  console.error('错误信息: ', err)
}
```

然后两个需求都要，那就凑合两个，但叠一起看更乱了。

### 中间件的概念

显然，我们不能通过这种方式来做。比较理想的方案是 Redux 本身提供一个功能入口，让我们可以在外面添加功能进去，这样代码就不会复杂。

但如果给我们现有实现的 Redux 添加功能，在哪个环节添加比较合适呢？

- Reducer： 纯函数，只承担计算 State 的功能，不合适承担其他功能，也承担不了，因为理论上，纯函数不能进行读写操作。
- View：与 State 一一对应，可以看作 State 的视觉层，也不合适承担其他功能。
- Action：存放数据的对象，即消息的载体，只能被别人操作，自己不能进行任何操作。

我们发现，以上需求都是和 dispatch 相关，只有发送 action 的这个步骤，即 store.dispatch() 方法，可以添加功能。比如添加日志功能，我们只要把日志放进 dispatch 函数里，不就好了吗，我们只需要改造 dispatch 函数，把 dispatch 进行一层封装。

```javascript
const store = createStore(counter)
const next = store.dispatch
store.dispatch = action => {
  try {
    console.log('prev state', store.getState())
    console.log(action)
    next(action)
    console.log('next state', store.getState())
  } catch (err) {
    console.error('错误信息: ', err)
  }
}
```

上面代码，对 store.dispatch 进行了重新定义，这就是中间件的雏形。

所以说 Redux 的中间件就是一个函数，是对 dispatch 方法的扩展，增强 dispatch 的功能。

### 实现中间件

对于上述 dispatch 的封装，实际上是缺陷很大的。万一又来 n 多个需求怎么办? 那 dispatch 函数就混乱到无法维护了，故需要扩展性强的多中间件合作模式。

1. 我们把 loggerMiddleware 提取出来

```javascript
const store = createStore(counter)
const next = store.dispatch

const loggerMiddleware = action => {
  console.log('prev state', store.getState())
  console.log(action)
  next(action)
  console.log('next state', store.getState())
}

store.dispatch = action => {
  try {
    loggerMiddleware(action)
  } catch (err) {
    console.error('错误信息: ', err)
  }
}
```

2. 把 exceptionMiddleware 提取出来

```javascript
const exceptionMiddleware = action => {
  try {
    loggerMiddleware(action)
  } catch (err) {
    console.error('错误信息: ', err)
  }
}

store.dispatch = exceptionMiddleware
```

3. 现在代码有个问题，就是 exceptionMiddleware 中间件写死 loggerMiddleware，但以后又万一不要记录功能呢，所以我们需要让 next(action) 变成动态的，即换哪个中间件都可以

```javascript
const exceptionMiddleware = next => action => {
  try {
    // loggerMiddleware(action)
    next(action)
  } catch (err) {
    console.error('错误信息: ', err)
  }
}
```

这个写法可能刚开始看不太适应，实际就是函数里面，返回一个函数，即等效于

```javascript
const exceptionMiddleware = function (next) {
  return function (action) {
    try {
      // loggerMiddleware(action)
      next(action)
    } catch (err) {
      console.error('错误信息: ', err)
    }
  }
}
```

传参数的时候即是 exceptionMiddleware(next)(action)

4. 同理，我们让 loggerMiddleware 里面无法扩展别的中间件了！我们也把 next 写成动态的

```javascript
const loggerMiddleware = next => action => {
  console.log('prev state', store.getState())
  console.log(action)
  next(action)
  console.log('next state', store.getState())
}
```

目前为止，整个中间件设计改造如下：

```javascript
const store = createStore(counter)
const next = store.dispatch

const loggerMiddleware = next => action => {
  console.log('prev state', store.getState())
  console.log(action)
  next(action)
  console.log('next state', store.getState())
}

const exceptionMiddleware = next => action => {
  try {
    next(action)
  } catch (err) {
    console.error('错误信息: ', err)
  }
}

store.dispatch = exceptionMiddleware(loggerMiddleware(next))
```

5. 现在又有一个新问题，想想平时使用中间件是从外部引入的，那外部中间件里面怎么会有 store.getState() 这个方法，于是我们把 store 也给独立出去。

```javascript
const store = createStore(counter)
const next = store.dispatch

const loggerMiddleware = store => next => action => {
  console.log('prev state', store.getState())
  console.log(action)
  next(action)
  console.log('next state', store.getState())
}

const exceptionMiddleware = store => next => action => {
  try {
    next(action)
  } catch (err) {
    console.error('错误信息: ', err)
  }
}

const logger = loggerMiddleware(store)
const exception = exceptionMiddleware(store)
store.dispatch = exception(logger(next))
```

6. 如果又有一个新需求，需要在打印日志前输出当前时间戳，我们又需要构造一个中间件

```javascript
const timeMiddleware = store => next => action => {
  console.log('time', new Date().getTime())
  next(action)
}

const logger = loggerMiddleware(store)
const exception = exceptionMiddleware(store)
const time = timeMiddleware(store)
store.dispatch = exception(time(logger(next)))
```

### 中间件使用方式优化

上面的写法可知，中间件的使用方式有点繁琐，故我们需要把细节封装起来，通过扩展 createStore 来实现。
先来看看期望的用法：

```javascript
/* 接收旧的 createStore，返回新的 createStore */
const newCreateStore = applyMiddleware(
  exceptionMiddleware,
  timeMiddleware,
  loggerMiddleware
)(createStore)

/* 返回了一个 dispatch 被重写过的 store */
const store = newCreateStore(reducer)
```

### 实现 applyMiddleware

```javascript
export const applyMiddleware = function (...middlewares) {
  /* 返回一个重写createStore的方法 */
  return function rewriteCreateStoreFunc(oldCreateStore) {
    /* 返回重写后新的 createStore */
    return function newCreateStore(reducer, preloadedState) {
      // 生成 store
      const store = oldCreateStore(reducer, preloadedState)
      let dispatch = store.dispatch

      // 只暴露 store 部分给中间件用的API，而不传入整个store
      const middlewareAPI = {
        getState: store.getState,
        dispatch: action => store.dispatch(action),
      }
      // 给每个中间件传入API
      // 相当于 const logger = loggerMiddleware(store)，即 const logger = loggerMiddleware({ getState, dispatch })
      // const chain = [exception, time, logger]
      const chain = middlewares.map(middleware => middleware(middlewareAPI))
      // 实现 exception(time((logger(dispatch))))
      chain.reverse().map(middleware => {
        dispatch = middleware(dispatch)
      })
      // 重写dispatch
      store.dispatch = dispatch
      return store
    }
  }
}
```

我们来看这一处代码：

```javascript
chain.reverse().map(middleware => {
  dispatch = middleware(dispatch)
})
```

要注意一点，中间件是顺序执行，但是 dispatch 却是反序生成的。所以在这步会把数组顺序给反序（比如 applyMiddleware(A, B, C)，因为 A 在调用时需要知道 B 的 dispatch，B 在执行时需要知道 C 的 dispatch，那么需要先知道 C 的 dispatch。）

官方 Redux 源码，采用了 compose 函数，我们也试试这种方式来写：

```javascript
export const applyMiddleware = (...middlewares) => {
  return createStore => (...args) => {
    // ...
    dispatch = compose(...chain)(store.dispatch)
    // ...
  }
}

// compose(fn1, fn2, fn3)
// fn1(fn2(fn3))
// 从右到左来组合多个函数: 从右到左把接收到的函数合成后的最终函数
export const compose = (...funcs) => {
  if (funcs.length === 0) {
    return arg => arg
  }
  if (funcs.length === 1) {
    return funcs[0]
  }
  return funcs.reduce((ret, item) => (...args) => ret(item(...args)))
}
```

我们再对代码精简：

```javascript
export const applyMiddleware = (...middlewares) => {
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = store.dispatch

    const middlewareAPI = {
      getState: store.getState,
      dispatch: action => dispatch(action),
    }

    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch,
    }
  }
}

export const compose = (...funcs) => {
  if (funcs.length === 0) {
    return arg => arg
  }
  if (funcs.length === 1) {
    return funcs[0]
  }
  return funcs.reduce((ret, item) => (...args) => ret(item(...args)))
}
```

### createStore 的处理

现在的问题是，有两个 createStore 了，这怎么区分，上篇我们其实已经先告知了对中间件代码处理，但具体怎么推出的，我们继续看。

```javascript
// 没有中间件的 createStore
const store = createStore(counter)

// 有中间件的 createStore
const rewriteCreateStoreFunc = applyMiddleware(
  exceptionMiddleware,
  timeMiddleware,
  loggerMiddleware
)
const newCreateStore = rewriteCreateStoreFunc(createStore)
const store = newCreateStore(counter, preloadedState)
```

为了让用户用起来统一一些，我们可以很简单的使他们的使用方式一致，我们修改下 createStore 方法

```javascript
const createStore = (reducer, preloadedState, rewriteCreateStoreFunc) => {
  // 如果有 rewriteCreateStoreFunc，那就采用新的 createStore
  if (rewriteCreateStoreFunc) {
    const newCreateStore = rewriteCreateStoreFunc(createStore)
    return newCreateStore(reducer, preloadedState)
  }
  // ...
}
```

不过 Redux 源码 rewriteCreateStoreFunc 换了个名字，还加了判断，也就是我们上篇的代码：

```javascript
if (typeof enhancer !== 'undefined') {
  if (typeof enhancer !== 'function') {
    throw new Error('Expected the enhancer to be a function.')
  }
  return enhancer(createStore)(reducer, preloadedState)
}
```

所以中间件的用法为

```javascript
const store = createStore(counter, /* preloadedState可选 */ applyMiddleware(logger))
```

## combineReducers

如果我们做的项目很大，有大量 state，那么维护起来很麻烦。Redux 提供了 combineReducers 这个方法，作用是把多个 reducer 合并成一个 reducer， 每个 reducer 负责独立的模块。

我们用一个新例子来举例：

```javascript
import { createStore, applyMiddleware, combineReducers } from 'redux'

const initCounterState = {
  value: 10,
}
const initInfoState = {
  name: 'jacky',
}

const reducer = combineReducers({
  counter: counterReducer,
  info: infoReducer,
})

// counter reducer处理函数
function counterReducer(state = initCounterState, action) {
  switch (action.type) {
    case 'INCREMENT':
      return {
        ...state,
        value: state.value + 1,
      }
    case 'DECREMENT':
      return {
        ...state,
        value: state.value - 1,
      }
    default:
      return state
  }
}

function infoReducer(state = initInfoState, action) {
  switch (action.type) {
    case 'FULL_NAME':
      return {
        ...state,
        name: state.name + ' lin',
      }
    default:
      return state
  }
}

const store = createStore(reducer)

const init = store.getState()
// 一开始counter为：10，info为 jacky
console.log(`一开始counter为：${init.counter.value}，info为 ${init.info.name}`)
function listener() {
  store.getState()
}
store.subscribe(listener) // 监听state的改变

// counterReducer
store.dispatch({ type: 'INCREMENT' })
store.dispatch({ type: 'INCREMENT' })
store.dispatch({ type: 'DECREMENT' })

// infoReducer
store.dispatch({ type: 'FULL_NAME' })

// 执行完counter为：11，info为jacky lin
console.log(
  `执行完counter为：${store.getState().counter.value}，info为${
    store.getState().info.name
  }`
)
export default store
```

我们来尝试下如何实现这个 API，

首先要把一个函数里的所有 reducers 循环执行一遍，并且这个函数要遵循(state, action) => newState 格式。还需要把每个 reducer 的 initState 合并成一个 rootState。
实现如下：

```javascript
export function combineReducers(reducers) {
  // reducerKeys = ['counter', 'info']
  const reducerKeys = Object.keys(reducers)
  // 返回合并后的新的reducer函数
  return function combination(state = {}, action) {
    // 生成的新的state
    const nextState = {}

    // 遍历执行所有的reducers，整合成为一个新的state
    for (let i = 0; i < reducerKeys.length; i++) {
      const key = reducerKeys[i]
      const reducer = reducers[key]
      // 之前的 key 的 state
      const previousStateForKey = state[key]
      // 执行 分 reducer，获得新的state
      const nextStateForKey = reducer(previousStateForKey, action)

      nextState[key] = nextStateForKey
    }
    return nextState
  }
}
```

## replaceReducer

> 在大型 Web 应用程序中，通常需要将应用程序代码拆分为多个可以按需加载的 JS 包。 这种称为“代码分割”的策略通过减小初次加载时的 JS 的包的大小，来提高应用程序的性能。

reducer 拆分后，和组件是一一对应的。我们就希望在做按需加载的时候，reducer 也可以跟着组件在必要的时候再加载，然后用新的 reducer 替换老的 reducer。但实际上只有一个 root reducer 函数, 如果要实现的话就可以用 replaceReducer 这个函数，实现如下：

```javascript
const createStore = function (reducer, initState) {
  // ...
  const replaceReducer = nextReducer => {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }
    reducer = nextReducer
    // 刷新一遍 state 的值，新来的 reducer 把自己的默认状态放到 state 树上去
    dispatch({ type: Symbol() })
  }
  // ...
  return {
    // ...
    replaceReducer,
  }
}
```

使用如下：

```javascript
const reducer = combineReducers({
  counter: counterReducer,
})
const store = createStore(reducer)

/*生成新的reducer*/
const nextReducer = combineReducers({
  counter: counterReducer,
  info: infoReducer,
})
/*replaceReducer*/
store.replaceReducer(nextReducer)
```

## bindActionCreators

bindActionCreators 一般比较少用到，在 react-redux 的 connect 函数实现会用到

会使用到 bindActionCreators 的场景是当你需要把 action creator 往下传到一个组件上，却不想让这个组件觉察到 Redux 的存在，而且不希望把 dispatch 或 Redux store 传给它。

我们通过普通的方式来 隐藏 dispatch 和 actionCreator 试试

```javascript
const reducer = combineReducers({
  counter: counterReducer,
  info: infoReducer,
})
const store = createStore(reducer)

// 返回 action 的函数就叫 actionCreator
function increment() {
  return {
    type: 'INCREMENT',
  }
}

function getName() {
  return {
    type: 'FULL_NAME',
  }
}

const actions = {
  increment: function () {
    return store.dispatch(increment.apply(this, arguments))
  },
  getName: function () {
    return store.dispatch(getName.apply(this, arguments))
  },
}
// 其他地方在实现自增的时候，根本不知道 dispatch，actionCreator等细节
actions.increment() // 自增
actions.getName() // 获得全名
```

把 actions 生成时候的公共代码提取出来：

```javascript
const actions = bindActionCreators({ increment, getName }, store.dispatch)
```

bindActionCreators 的实现如下：

```javascript
// 返回包裹 dispatch 的函数, 将 actionCreator 转化成 dispatch 形式
// eg. { addNumAction }  =>  (...args) => dispatch(addNumAction(args))
export function bindActionCreator(actionCreator, dispatch) {
  return function (...args) {
    return dispatch(actionCreator.apply(this, args))
  }
}

export function bindActionCreators(actionCreators, dispatch) {
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch)
  }
  // actionCreators 必须是 function 或者 object
  if (typeof actionCreators !== 'object' || actionCreators === null) {
    throw new Error()
  }

  const keys = Object.keys(actionCreators)
  const boundActionCreators = {}
  for (let i = 0; i < keys.length; i++) {
    const key = keys[i]
    const actionCreator = actionCreators[key]
    if (typeof actionCreator === 'function') {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch)
    }
  }
  return boundActionCreators
}
```

可能大家看到这里有点懵逼，让我们来回忆下 react-redux 中 connect 函数的用法，
比如有这样一个 actionCreators

```javascript
// actionCreators.js
function addNumAction() {
  return { type: 'ADD_NUM' }
}
// Demo.js：在需要用到 store 数据的组件，如 Demo 组件底部我们用 connect 函数连接，如下:
import { addNumAction } from './actionCreators'
const mapDispatchToProps = dispatch => ({
  addNum() {
    dispatch(addNumAction())
  },
})
export default connect(mapStateToProps, mapDispatchToProps)(Demo)
```

然后通过页面的按钮来出发 action 为 ADD_NUM 对应事件

```javascript
<button onClick={this.props.addNum}>增加1</button>
```

但除了上面的用法，mapDispatchToProps 也可以这样用，直接传入一个对象，都没有 dispatch 方法

```javascript
export default connect(mapStateToProps, { addNumAction })(Demo)
```

然后只需触发 addNumAction 就能实现和上面一样的效果。

为什么可以不传，当你传入对象的时候， connect 函数会判断，大致代码如下：

```javascript
let dispatchToProps

if (typeof mapDispatchToProps === 'function') {
  dispatchToProps = mapDispatchToProps(store.dispatch)
} else {
  // 传递了一个 actionCreator 对象过来
  dispatchToProps = bindActionCreators(mapDispatchToProps, store.dispatch)
}
```

这里就使用了 bindActionCreators 函数，它就是把你传入的 actionCreator 再包一层 dispatch 方法，即

```javascript
{ addNumAction }  =>  (...args) => dispatch(addNumAction(args))
```

## 总结

Redux 实现讲到这里就结束了，把原理搞懂了确实对 Redux 的理解加深了好多，之后会继续写相关插件的实现，如 react-redux 等。

参考资料：

[完全理解 redux（从零实现一个 redux）](https://github.com/brickspert/blog/issues/22)

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励吧~

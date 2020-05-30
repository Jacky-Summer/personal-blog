# 理解 redux-thunk 源码

## 前言

前面几篇我们就 Redux 展开了几篇文章，这次我们来实现 react-thunk，就不是叫实现 redux-thunk 了，直接上源码，因为源码就 11 行。如果对 Redux 中间件还不理解的，可以看我写的 Redux 文章。

- [实现一个迷你 Redux（基础版）](https://github.com/Jacky-Summer/personal-blog/blob/master/React%E7%B3%BB%E5%88%97/%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AA%E8%BF%B7%E4%BD%A0Redux%EF%BC%88%E5%9F%BA%E7%A1%80%E7%89%88%EF%BC%89.md)
- [实现一个 Redux（完善版）](https://github.com/Jacky-Summer/personal-blog/blob/master/React%E7%B3%BB%E5%88%97/%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AARedux%EF%BC%88%E5%AE%8C%E5%96%84%E7%89%88%EF%BC%89.md)
- [浅谈 React 的 Context API](https://github.com/Jacky-Summer/personal-blog/blob/master/React%E7%B3%BB%E5%88%97/%E6%B5%85%E8%B0%88React%E7%9A%84Context%20API.md)
- [带你实现 react-redux](https://github.com/Jacky-Summer/personal-blog/blob/master/React%E7%B3%BB%E5%88%97/%E5%B8%A6%E4%BD%A0%E5%AE%9E%E7%8E%B0%20react-redux.md)

## 为什么要用 redux-thunk

在使用 Redux 过程，通过 dispatch 方法派发一个 action 对象。当我们使用 redux-thunk 后，可以 dispatch 一个 function。redux-thunk 会自动调用这个 function，并且传递 dispatch, getState 方法作为参数。这样一来，我们就能在这个 function 里面处理异步逻辑，处理复杂逻辑，这是原来 Redux 做不到的，因为原来就只能 dispatch 一个简单对象。

## 用法

redux-thunk 作为 redux 的中间件，主要用来处理异步请求，比如：

```javascript
export function fetchData() {
  return (dispatch, getState) => {
    // to do ...
    axios.get('https://jsonplaceholder.typicode.com/todos/1').then(res => {
      console.log(res)
    })
  }
}
```

## redux-thunk 源码

redux-thunk 的源码比较简洁，实际就 11 行。前几篇我们说到 redux 的中间件形式，
本质上是对 store.dispatch 方法进行了增强改造，基本是类似这种形式：

```
const middleware = (store) => next => action => {}
```

在这里就不详细解释了，可以看 [实现一个 Redux（完善版）](https://github.com/Jacky-Summer/personal-blog/blob/master/React%E7%B3%BB%E5%88%97/%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AARedux%EF%BC%88%E5%AE%8C%E5%96%84%E7%89%88%EF%BC%89.md)

先给个缩水版的实现：

```javascript
const thunk = ({ getState, dispatch }) => next => action => {
  if (typeof action === 'function') {
    return action(dispatch, getState)
  }
  return next(action)
}
export default thunk
```

- 原理：即当 action 为 function 的时候，就调用这个 function (传入 dispatch, getState)并返回；如果不是，就直接传给下一个中间件。

完整源码如下：

```javascript
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    // 如果action是一个function，就返回action(dispatch, getState, extraArgument)，否则返回next(action)。
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument)
    }
    // next为之前传入的store.dispatch，即改写前的dispatch
    return next(action)
  }
}

const thunk = createThunkMiddleware()
// 给thunk设置一个变量withExtraArgument，并且将createThunkMiddleware整个函数赋给它
thunk.withExtraArgument = createThunkMiddleware

export default thunk
```

我们发现其实还多了 extraArgument 传入，这个是自定义参数，如下用法：

```javascript
const api = 'https://jsonplaceholder.typicode.com/todos/1'
const whatever = 10

const store = createStore(
  reducer,
  applyMiddleware(thunk.withExtraArgument({ api, whatever }))
)

// later
function fetchData() {
  return (dispatch, getState, { api, whatever }) => {
    // you can use api and something else here
  }
}
```

## 总结

同 redux-thunk 非常流行的库 redux-saga 一样，都是在 redux 中做异步请求等副作用。Redux 相关的系列文章就暂时写到这部分为止，下次会写其他系列。

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~

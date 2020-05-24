# 带你实现 react-redux

## 前言

之前我们实现了 redux 的功能，这次我们来实现一下配合 redux 开发中经常会用到的一个库—— react-redux。本文不会详细介绍 react-redux 的使用，另外需要了解 Context API， 再看此文就很容易理解了。可以看看我之前写的几篇文章

- [实现一个迷你 Redux（基础版）](https://github.com/Jacky-Summer/personal-blog/blob/master/React%E7%B3%BB%E5%88%97/%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AA%E8%BF%B7%E4%BD%A0Redux%EF%BC%88%E5%9F%BA%E7%A1%80%E7%89%88%EF%BC%89.md)
- [实现一个 Redux（完善版）](https://github.com/Jacky-Summer/personal-blog/blob/master/React%E7%B3%BB%E5%88%97/%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AARedux%EF%BC%88%E5%AE%8C%E5%96%84%E7%89%88%EF%BC%89.md)
- [浅谈 React 的 Context API](https://github.com/Jacky-Summer/personal-blog/blob/master/React%E7%B3%BB%E5%88%97/%E6%B5%85%E8%B0%88React%E7%9A%84Context%20API.md)

## react-redux 基本使用

用个简单的加减数字作例子, 把代码贴出来：

- redux.js

```javascript
import { createStore } from 'redux'

// actions
const ADD_NUM = 'ADD_NUM'
const DESC_NUM = 'DESC_NUM'

// action creators
export function addNumAction() {
  return {
    type: ADD_NUM,
  }
}

export function reduceNumAction() {
  return {
    type: DESC_NUM,
  }
}

const defaultState = {
  num: 0,
}

// reducer
const reducer = (state = defaultState, action) => {
  switch (action.type) {
    case ADD_NUM:
      return { num: state.num + 1 }
    case DESC_NUM:
      return { num: state.num - 1 }
    default:
      return state
  }
}

const store = createStore(reducer)

export default store
```

index.js

```javascript
import React from 'react'
import ReactDOM from 'react-dom'
import Demo from './Demo'
import { Provider } from 'react-redux'
import store from './redux.js'

ReactDOM.render(
  <Provider store={store}>
    <Demo />
  </Provider>,
  document.getElementById('root')
)
```

Demo.js

```javascript
import React, { Component } from 'react'
import { connect } from 'react-redux'
import { addNumAction, reduceNumAction } from './redux.js'

class Demo extends Component {
  render() {
    console.log(this.props)
    return (
      <div>
        <p>{this.props.num}</p>
        <button onClick={this.props.addNum}>增加1</button>
        <button onClick={this.props.reduceNum}>减少1</button>
      </div>
    )
  }
}

const mapStateToProps = state => {
  return {
    num: state.num,
  }
}

const mapDispatchToProps = dispatch => {
  return {
    addNum() {
      const action = addNumAction()
      dispatch(action)
    },
    reduceNum() {
      const action = reduceNumAction()
      dispatch(action)
    },
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(Demo)
```

就可以实现 num 的增减：

![](https://user-gold-cdn.xitu.io/2020/5/24/172443ab8612774b?w=143&h=82&f=png&s=1764)
其实一个简单的 react-redux， 主要也就是实现 connect 和 Provider 的基本功能

- **connect**：可以把 state 和 dispatch 绑定到 react 组件，使得组件可以访问到 redux 的数据
- **Provider**：提供的是一个顶层容器的作用，实现 store 的上下文传递

## 使用旧版 Context API 实现

### 实现 Provider

首先我们看它的用法，就知道它不是一个函数，而是一个组件：

```
<Provider store={store}>
    <Demo />
</Provider>
```

React 的 Context API 提供了一种通过组件树传递数据的方法，无需在每个级别手动传递 props 属性。

Provider 的实现比较简单，核心就是**把 store 放到 context 里面，所有的子元素可以直接取到 store**。

```javascript
export class Provider extends Component {
  static childContextTypes = {
    store: PropTypes.object,
  }

  constructor(props) {
    super(props)
    this.store = props.store // 也就是 Provider 组件从外部传入的 store
  }

  getChildContext() {
    return { store: this.store }
  }

  render() {
    return this.props.children // 中间包的组件传入了 context，其余原封不动返回组件
  }
}
```

还有个地方大家知道就好，两种写法一样的，对 context type 的约束

```javascript
export class Provider extends Component {
  // 不用 static 来声明...
}
Provider.childContextTypes = {
  store: PropTypes.object,
}
```

### 实现 connect

connect 用法

```javascript
export default connect(mapStateToProps, mapDispatchToProps)(Demo)
```

connect 是一个高阶组件，就是以组件作为参数，返回一个组件。

**connect 负责连接组件，给到 redux 的数据放到组件的属性里**

1. 负责接收一个组件，把 state 的一些数据放进去，返回一个组件
2. 数据变化的时候，能够通知组件（需要进行监听）

```javascript
export function connect(mapStateToProps, mapDispatchToProps) {
  return function (WrapComponent) {
    return class ConnectComponent extends Component {
      static contextTypes = {
        store: PropTypes.object,
      }

      constructor(props) {
        super(props)
        this.state = {
          props: {},
        }
      }

      componentDidMount() {
        const { store } = this.context
        store.subscribe(() => this.update()) // 监听更新事件
        this.update()
      }

      update() {
        const { store } = this.context
        let stateToProps = mapStateToProps(store.getState()) // 传入 redux 的 state
        let dispatchToProps = mapDispatchToProps(store.dispatch) // 传入 redux 的 dispatch

        this.setState({
          props: {
            ...this.state.props,
            ...stateToProps,
            ...dispatchToProps,
          },
        })
      }

      render() {
        return <WrapComponent {...this.state.props} />
      }
    }
  }
}
```

这样 connect 就实现了，但还有一个问题，像上面的例子，我们其实可以直接传入 action creators， 而不用自己定义函数传入 dispatch

```
export default connect(mapStateToProps, { addNumAction, reduceNumAction })(Demo)
```

调用的时候：

```
<button onClick={this.props.addNumAction}>增加1</button>
<button onClick={this.props.reduceNumAction}>减少1</button>
```

那它的 dispatch 哪里来的，其实是用了 redux 的 bindActionCreators 函数，在我介绍 redux 的文章有提到，它作用是将 actionCreator 转化成 dispatch 形式，即

```
{ addNumAction }  =>  (...args) => dispatch(addNumAction(args))
```

所以我们需要再更改 connect 函数，同时，这次我们用箭头函数的形式简化代码

```javascript
import { bindActionCreators } from 'redux'

export const connect = (mapStateToProps, mapDispatchToProps) => WrapComponent => {
  return class ConnectComponent extends Component {
    static contextTypes = {
      store: PropTypes.object,
    }

    constructor(props) {
      super(props)
      this.state = {
        props: {},
      }
    }

    componentDidMount() {
      const { store } = this.context
      store.subscribe(() => this.update())
      this.update()
    }

    update() {
      const { store } = this.context
      let stateToProps = mapStateToProps(store.getState())
      let dispatchToProps
      if (typeof mapDispatchToProps === 'function') {
        dispatchToProps = mapDispatchToProps(store.dispatch)
      } else {
        // 传递了一个 actionCreator 对象过来
        dispatchToProps = bindActionCreators(mapDispatchToProps, store.dispatch)
      }

      this.setState({
        props: {
          ...this.state.props,
          ...stateToProps,
          ...dispatchToProps,
        },
      })
    }

    render() {
      return <WrapComponent {...this.state.props} />
    }
  }
}
```

以上，我们实现了最基本版的 react-redux，然后接下来，我们用新版的 Context API 再写一次

## 使用新版 Context API 实现

```javascript
import React, { Component } from 'react'
import { bindActionCreators } from 'redux'

const StoreContext = React.createContext(null)

// 这次 Provider 采取更简洁的形式写
export class Provider extends Component {
  render() {
    return (
      <StoreContext.Provider value={this.props.store}>
        {this.props.children}
      </StoreContext.Provider>
    )
  }
}

export function connect(mapStateToProps, mapDispatchToProps) {
  return function (WrapComponent) {
    class ConnectComponent extends Component {
      constructor(props) {
        super(props)
        this.state = {
          props: {},
        }
      }

      componentDidMount() {
        const { store } = this.props
        store.subscribe(() => this.update())
        this.update()
      }

      update() {
        const { store } = this.props
        let stateToProps = mapStateToProps(store.getState())
        let dispatchToProps
        if (typeof mapDispatchToProps === 'function') {
          dispatchToProps = mapDispatchToProps(store.dispatch)
        } else {
          // 传递了一个 actionCreator 对象过来
          dispatchToProps = bindActionCreators(mapDispatchToProps, store.dispatch)
        }

        this.setState({
          props: {
            ...this.state.props,
            ...stateToProps,
            ...dispatchToProps,
          },
        })
      }

      render() {
        return <WrapComponent {...this.state.props} />
      }
    }

    return () => (
      <StoreContext.Consumer>
        {value => <ConnectComponent store={value} />}
      </StoreContext.Consumer>
    )
  }
}
```

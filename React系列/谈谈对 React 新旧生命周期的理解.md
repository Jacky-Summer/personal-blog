# 谈谈对 React 新旧生命周期的理解

## 前言

在写这篇文章的时候，React 已经出了 17.0.1 版本了，虽说还来讨论目前 React 新旧生命周期有点晚了，React 两个新生命周期虽然出了很久，但实际开发我却没有用过，因为 React 16 版本后我们直接 React Hook 起飞开发项目。

但对新旧生命周期的探索，还是有助于我们更好理解 React 团队一些思想和做法，于是今天就要回顾下这个问题和理解总结，虽然还是 React Hook 写法香，但是依然要深究学习类组件的东西，了解 React 团队的一些思想与做法。

本文只讨论 React17 版本前的。

## React 16 版本后做了什么

首先是给三个生命周期函数加上了 UNSAFE：

- UNSAFE_componentWillMount
- UNSAFE_componentWillReceiveProps
- UNSAFE_componentWillUpdate

这里并不是表示不安全的意思，它只是不建议继续使用，并表示使用这些生命周期的代码可能在未来的 React 版本（目前 React17 还没有完全废除）存在缺陷，如 React Fiber 异步渲染的出现。

同时新增了两个生命周期函数：

- getDerivedStateFromProps
- getSnapshotBeforeUpdate

## UNSAFE_componentWillReceiveProps

```
UNSAFE_componentWillReceiveProps(nextProps)
```

先来说说这个函数，`componentWillReceiveProps`

该子组件方法并不是父组件 props 改变才触发，官方回答是：

> 如果父组件导致组件重新渲染，即使 props 没有更改，也会调用此方法。如果只想处理更改，请确保进行当前值与变更值的比较。

先来说说 React 为什么废除该函数，废除肯定有它不好的地方。

`componentWillReceiveProps`函数的一般使用场景是：

- 如果组件自身的某个 state 跟父组件传入的 props 密切相关的话，那么可以在该方法中判断前后两个 props 是否相同，如果不同就根据 props 来更新组件自身的 state。
  类似的业务需求比如：一个可以横向滑动的列表，当前高亮的 Tab 显然隶属于列表自身的状态，但很多情况下，业务需求会要求从外部跳转至列表时，根据传入的某个值，直接定位到某个 Tab。

但该方法缺点是会破坏 state 数据的单一数据源，导致组件状态变得不可预测，另一方面也会增加组件的重绘次数。

而在新版本中，官方将更新 state 与触发回调重新分配到了 `getDerivedStateFromProps` 与 `componentDidUpdate` 中，使得组件整体的更新逻辑更为清晰。

新生命周期方法`static getDerivedStateFromProps(props, state)`怎么用呢？

> getDerivedStateFromProps 会在调用 render 方法之前调用，并且在初始挂载及后续更新时都会被调用。它应返回一个对象来更新 state，如果返回 null 则不更新任何内容。

从函数名字就可以看出大概意思：使用 props 来派生/更新 state。这就是重点了，但凡你想使用该函数，都必须出于该目的，使用它才是正确且符合规范的。

跟`getDerivedStateFromProps`不同的是，它在挂载和更新阶段都会执行（`componentWillReceiveProps`挂载阶段不会执行），因为更新 state 这种需求不仅在 props 更新时存在，在 props 初始化时也是存在的。

而且`getDerivedStateFromProps`在组件自身 state 更新也会执行而`componentWillReceiveProps`方法执行则取决于父组件的是否触发重新渲染，也可以看出`getDerivedStateFromProps`并不是 `componentWillReceiveProps`方法的替代品.

引起我们注意的是，这个生命周期方法是一个静态方法，静态方法不依赖组件实例而存在，故在该方法内部是无法访问 this 的。新版本生命周期方法能做的事情反而更少了，限制我们只能根据 props 来派生 state，官方是基于什么考量呢？

因为无法拿到组件实例的 this，这也导致我们无法在函数内部做 this.fetch()请求，或者不合理的 this.setState()操作导致可能的死循环或其他副作用。有没有发现，这都是不合理不规范的操作，但开发者们都有机会这样用。可如果加了个静态 static，间接强制我们都无法做了，也从而避免对生命周期的滥用。

React 官方也是通过该限制，尽量保持生命周期行为的可控可预测，根源上帮助了我们避免不合理的编程方式，即一个 API 要保持单一性，做一件事的理念。

如下例子：

```javascript
// before
componentWillReceiveProps(nextProps) {
  if (nextProps.isLogin !== this.props.isLogin) {
    this.setState({
      isLogin: nextProps.isLogin,
    });
  }
  if (nextProps.isLogin) {
    this.handleClose();
  }
}

// after
static getDerivedStateFromProps(nextProps, prevState) {
  if (nextProps.isLogin !== prevState.isLogin) { // 被对比的props会被保存一份在state里
    return {
      isLogin: nextProps.isLogin, // getDerivedStateFromProps 的返回值会自动 setState
    };
  }
  return null;
}

componentDidUpdate(prevProps, prevState) {
  if (!prevState.isLogin && this.props.isLogin) {
    this.handleClose();
  }
}
```

## UNSAVE_componentWillMount

> UNSAFE_componentWillMount() 在挂载之前被调用。它在 render() 之前调用，因此在此方法中同步调用 setState() 不会触发额外渲染。

我们应该避免在此方法中引入任何副作用或事件订阅，而是选用`componentDidMount()`。

在 React 初学者刚接触的时候，可能有这样一个疑问：一般都是数据请求放在`componentDidMount`里面，但放在`componentWillMount`不是会更快获取数据吗？

因为理解是`componentWillMount`在 render 之前执行，早一点执行就早拿到请求结果；但是其实不管你请求多快，都赶不上首次 render，页面首次渲染依旧处于没有获取异步数据的状态。

还有一个原因，`componentWillMount`是服务端渲染唯一会调用的生命周期函数，如果你在此方法中请求数据，那么服务端渲染的时候，在服务端和客户端都会分别请求两次相同的数据，这显然也我们想看到的结果。

特别是有了 React Fiber，更有机会被调用多次，故请求不应该放在`componentWillMount`中。

还有一个错误的使用是在`componentWillMount`中订阅事件，并在`componentWillUnmount`中取消掉相应的事件订阅。事实上只有调用`componentDidMount`后，React 才能保证稍后调用`componentWillUnmount`进行清理。而且服务端渲染时不会调用`componentWillUnmount`，可能导致内存泄露。

还有人会将事件监听器（或订阅）添加到 `componentWillMount` 中，但这可能导致服务器渲染（永远不会调用 `componentWillUnmount`）和异步渲染（在渲染完成之前可能被中断，导致不调用 `componentWillUnmount`）的内存泄漏。

对于该函数，一般情况，如果项目有使用，则是通常把现有 `componentWillMount` 中的代码迁移至 `componentDidMount` 即可。

## UNSAFE_componentWillUpdate

> 当组件收到新的 props 或 state 时，会在渲染之前调用 `UNSAFE_componentWillUpdate()`。使用此作为在更新发生之前执行准备更新的机会。初始渲染不会调用此方法。

注意，不能在该方法中调用 this.setState()；在 `componentWillUpdate` 返回之前，你也不应该执行任何其他操作（例如，dispatch Redux 的 action）触发对 React 组件的更新。

首先跟上面两个函数一样，该函数也发生在 render 之前，也存在一次更新被调用多次的可能，从这一点上看就依然不可取了。

其次，该方法常见的用法是在组件更新前，读取当前某个 DOM 元素的状态，并在 `componentDidUpdate` 中进行相应的处理。但 React 16 版本后有 suspense、异步渲染机制等等，render 过程可以被分割成多次完成，还可以被暂停甚至回溯，这导致 `componentWillUpdate` 和 `componentDidUpdate` 执行前后可能会间隔很长时间，这导致 DOM 元素状态是不安全的，因为这时的值很有可能已经失效了。而且足够使用户进行交互操作更改当前组件的状态，这样可能会导致难以追踪的 BUG。

为了解决这个问题，于是就有了新的生命周期函数：

```
getSnapshotBeforeUpdate(prevProps, prevState)
```

> `getSnapshotBeforeUpdate` 在最近一次渲染输出（提交到 DOM 节点）之前调用。它使得组件能在发生更改之前从 DOM 中捕获一些信息（例如，滚动位置）。此生命周期的任何返回值将作为第三个参数传入`componentDidUpdate(prevProps, prevState, snapshot)`

与 `componentWillUpdate` 不同，`getSnapshotBeforeUpdate` 会在最终的 render 之前被调用，也就是说在 `getSnapshotBeforeUpdate` 中读取到的 DOM 元素状态是可以保证与 `componentDidUpdate` 中一致的。

虽然 `getSnapshotBeforeUpdate` 不是一个静态方法，但我们也应该尽量使用它去返回一个值。这个值会随后被传入到 `componentDidUpdate` 中，然后我们就可以在 `componentDidUpdate` 中去更新组件的状态，而不是在 `getSnapshotBeforeUpdate` 中直接更新组件状态。避免了 `componentWillUpdate` 和 `componentDidUpdate` 配合使用时将组件临时的状态数据存在组件实例上浪费内存，`getSnapshotBeforeUpdate` 返回的数据在 `componentDidUpdate` 中用完即被销毁，效率更高。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d8eb7f64f3f94a9f8038949001284385~tplv-k3u1fbpfcp-watermark.image)

来看官方的一个例子：

```
class ScrollingList extends React.Component {
  constructor(props) {
    super(props);
    this.listRef = React.createRef();
  }

  getSnapshotBeforeUpdate(prevProps, prevState) {
    // 我们是否在 list 中添加新的 items？
    // 捕获滚动位置以便我们稍后调整滚动位置。
    if (prevProps.list.length < this.props.list.length) {
      const list = this.listRef.current;
      return list.scrollHeight - list.scrollTop;
    }
    return null;
  }

  componentDidUpdate(prevProps, prevState, snapshot) {
    // 如果我们 snapshot 有值，说明我们刚刚添加了新的 items，
    // 调整滚动位置使得这些新 items 不会将旧的 items 推出视图。
    //（这里的 snapshot 是 getSnapshotBeforeUpdate 的返回值）
    if (snapshot !== null) {
      const list = this.listRef.current;
      list.scrollTop = list.scrollHeight - snapshot;
    }
  }

  render() {
    return (
      <div ref={this.listRef}>{/* ...contents... */}</div>
    );
  }
}
```

如果项目中有用到`componentWillUpdate`的话，升级方案就是将现有的 `componentWillUpdate` 中的回调函数迁移至 `componentDidUpdate`。如果触发某些回调函数时需要用到 DOM 元素的状态，则将对比或计算的过程迁移至 `getSnapshotBeforeUpdate`，然后在 `componentDidUpdate` 中统一触发回调或更新状态。

除了这些，React 16 版本的依然还有大改动，其中引人注目的就是 Fiber，之后我还会抽空写一篇关于 React Fiber 的文章，可以关注我的[个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~

参考：

- [为什么废弃 react 生命周期函数](https://www.html.cn/qa/react/14367.html)
- [React v16.3 版本新生命周期函数浅析及升级方案](https://juejin.cn/post/6844903600309665799)

# Webpack 动态导入 react-toastify

## 前言

如果你的项目正在使用 `react-toastify`，可以看看本文。我是最近通过`webpack-bundle-analyzer`发现的一个问题，其实我们只有某个页面会可能用到弹框，比如你点了收藏之后会有弹框出来。但是通过打包结果和运行得出，每个页面无论有没有使用，一开始都引入了 `react-toastify`。

## react-toastify 的使用

如果你打开它 [Github](https://github.com/fkhadra/react-toastify#readme)，它是这么使用的：

```javascript
import React from 'react'

import { ToastContainer, toast } from 'react-toastify'
import 'react-toastify/dist/ReactToastify.css'

function App() {
  const notify = () => toast('Wow so easy!')

  return (
    <div>
      <button onClick={notify}>Notify!</button>
      <ToastContainer />
    </div>
  )
}
```

而如果整个项目多个地方用或者需要控制好数据流的话，一个做法往往是把它拉到 Redux 管理，在我们项目就变成

`app.tsx`

```javascript
// 里面封装了逻辑和返回了 <ToastContainer />
import Toast from "src/containers/Toast";

const { noToast } = this.props

return (
    /* ... */
    {!noToast && <Toast />}
)
```

项目使用的是 **Next.js**

我们的每个`pages/`页面都引入了这个`app.tsx`公共文件，上面代码中，Toast 是经过 Redux 等封装好的 Toast 组件，由上面代码粗略看出，如果在每个页面的`app.tsx`组件传入`noToast`属性，那么`<Toast />`就不会被加载，调用 `toast`方法也不会有弹框出现。

这段代码已经是几年前的了，推测是这个意图，默认导入，假设确认你页面真的不需要弹框，就传`noToast`属性，然而我搜索了整个项目，用到`noToast`的地方只有一个，所以这个属性并不被后来接手的人所知，也就不会手动传入`noToast`，导致每个页面都引入了`react-toasify`

> 使用 toast 方法必须页面要挂载了 `<ToastContainer />`，才可以弹出弹框

## 现有的问题

1. 我们只有在页面详情页点击收藏，才会有弹框（使用`react-toastify`），其他页面都没有。但每个页面都加载了该 chunk 包
2. 项目中使用 toast 方法是通过 Redux 触发，通过`useUpdateEffect`监听了全局

`src/containers/Toast.tsx`

```javascript
useUpdateEffect(() => {
  // toastProp从redux获取而来的
  if (toastProp.message || toastProp.errors) notify(toastProp) // notify里面判断和调用toast
}, [notify, toastProp])
```

## 优化

1. 当然是不要每个页面都引入，而是用到的页面再引入
2. 最好用到的页面，没点击时也不加载，比如只有点击收藏，才让`react-toasify`加载。

这就让人想到动态导入了，做法即是不再需要在`app.tsx`判断引入`react-toastify`，如新封装了一个组件

```javascript
import { ToastContent, ToastOptions } from 'react-toastify'

// 动态导入	 ./Toastify是封装好的 Toastify
export const toastify = () => import('./Toastify')

export const toast = (message: ToastContent, toastOptions?: ToastOptions) => {
  return toastify().then(toast => {
    toast.showToast(message, toastOptions)
  })
}
```

上面做到的效果即是：

```javascript
// before
import { toast } from './toastify'
toast('Hello World')

// after
import('./toastify').then(module => {
  module.toast('Hello World')
})
```

现在看看这个文件 `./Toastify.tsx`（省略很多代码...）

```javascript
const setupToastContainer = () => {
  if (!hasToastContainer) {
    hasToastContainer = true
    // 需要去给根html添加一个div，id为 root-toastify
    ReactDOM.render(
      <>
        <ToastGlobalStyle />
        <ToastWrapper />
      </>,
      document.getElementById('root-toastify')
    )
  }
}

export const showToast = (message, toastOptions = {}) => {
  setupToastContainer()
  toast(message, toastOptions)
}
```

直接摆脱了 Redux 依赖，而且只有你点击的一瞬间才会加载`react-toastify`的 chunk 包，初始加载并不会，故最后优化结果是每个页面都打包压缩体积都减少 15-20K+。

这次就有点偷懒写文了，因为这只是项目优化的其中一个小点，以前也没有记录优化点，这次做了就马上回顾一下，上面代码可能看的有点懵，没有放出完整代码和做个 Demo。

不过没关系，因为我在 Github 找到了完整的 PR 例子供大家参考 [enhancement: move react-toastify to its own bundle](https://github.com/getfider/fider/pull/645/files)，同样是动态导入`react-toastify`，做法一样。

<br>

- ps：
  - [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)

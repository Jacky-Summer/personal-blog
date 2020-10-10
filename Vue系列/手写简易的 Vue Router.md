## 前言

还是那样，懂得如何使用一个常用库，还得了解其原理或者怎么模拟实现，今天实现一下 `vue-router`。

有一些知识我这篇文章提到了，这里就不详细一步步写，请看我 [手写一个简易的 Vuex](https://github.com/Jacky-Summer/mini-vuex)

## 基本骨架

- Vue 里面使用插件的方式是`Vue.use(plugin)`，这里贴出它的用法：

> 安装 Vue.js 插件。如果插件是一个对象，必须提供 install 方法。如果插件是一个函数，它会被作为 install 方法。install 方法调用时，会将 Vue 作为参数传入。这个方法的第一个参数是 Vue 构造器，第二个参数是一个可选的选项对象。

- 全局混入

使用 `Vue.mixin(mixin)`

> 全局注册一个混入，影响注册之后所有创建的每个 Vue 实例。可以使用混入向组件注入自定义的行为，它将影响每一个之后创建的 Vue 实例。

- 路由用法

比如简单的:

```javascript
// 路由数组
const routes = [
  {
    path: '/',
    name: 'Page1',
    component: Page1,
  },
  {
    path: '/page2',
    name: 'Page2',
    component: Page2,
  },
]

const router = new VueRouter({
  mode: 'history', // 模式
  routes,
})
```

它是传入了`mode`和`routes`，我们实现的时候需要在`VueRouter`构造函数中接收。

在使用路由标题的时候是这样：

```html
<p>
  <!-- 使用 router-link 组件来导航. -->
  <!-- 通过传入 `to` 属性指定链接. -->
  <!-- <router-link> 默认会被渲染成一个 `<a>` 标签 -->
  <router-link to="/page1">Go to Foo</router-link>
  <router-link to="/page2">Go to Bar</router-link>
</p>
<!-- 路由出口 -->
<!-- 路由匹配到的组件将渲染在这里 -->
<router-view></router-view>
```

故我们需要使用`Vue.component(id, [definition])`注册一个全局组件。

了解了大概，我们就可以写出一个基本骨架

```javascript
let Vue = null

class VueRouter {
  constructor(options) {
    this.mode = options.mode || 'hash'
    this.routes = options.routes || []
  }
}

VueRouter.install = function (_Vue) {
  Vue = _Vue

  Vue.mixin({
    beforeCreate() {
      // 根组件
      if (this.$options && this.$options.router) {
        this._root = this // 把当前vue实例保存到_root上
        this._router = this.$options.router // 把router的实例挂载在_router上
      } else if (this.$parent && this.$parent._root) {
        // 子组件的话就去继承父组件的实例，让所有组件共享一个router实例
        this._root = this.$parent && this.$parent._root
      }
    },
  })

  Vue.component('router-link', {
    props: {
      to: {
        type: [String, Object],
        required: true,
      },
      tag: {
        type: String,
        default: 'a', // router-link 默认渲染成 a 标签
      },
    },
    render(h) {
      let tag = this.tag || 'a'
      return <tag href={this.to}>{this.$slots.default}</tag>
    },
  })

  Vue.component('router-view', {
    render(h) {
      return h('h1', {}, '视图显示的地方') // 暂时置为h1标签，下面会改
    },
  })
}

export default VueRouter
```

## mode

`vue-router`有两种模式，默认为 hash 模式。

### history 模式

通过`window.history.pushState`API 来添加浏览器历史记录，然后通过监听`popState`事件，也就是监听历史记录的改变，来加载相应的内容。

- popstate 事件

> 当活动历史记录条目更改时，将触发 popstate 事件。如果被激活的历史记录条目是通过对 history.pushState()的调用创建的，或者受到对 history.replaceState()的调用的影响，popstate 事件的 state 属性包含历史条目的状态对象的副本。

- History.pushState()方法

```
window.history.pushState(state, title, url)
```

该方法用于在历史中添加一条记录，接收三个参数，依次为：

```
state：一个与添加的记录相关联的状态对象，主要用于popstate事件。该事件触发时，该对象会传入回调函数。也就是说，浏览器会将这个对象序列化以后保留在本地，重新载入这个页面的时候，可以拿到这个对象。如果不需要这个对象，此处可以填null。
title：新页面的标题。但是，现在所有浏览器都忽视这个参数，所以这里可以填空字符串。
url：新的网址，必须与当前页面处在同一个域。浏览器的地址栏将显示这个网址。
```

### hash 模式

使用 URL 的 hash 来模拟一个完整的 URL。，通过监听`hashchange`事件，然后根据`hash`值（可通过 window.location.hash 属性读取）去加载对应的内容的。

继续增加代码，

```javascript
let Vue = null

class HistoryRoute {
  constructor() {
    this.current = null // 当前路径
  }
}

class VueRouter {
  constructor(options) {
    this.mode = options.mode || 'hash'
    this.routes = options.routes || []
    this.routesMap = this.createMap(this.routes)
    this.history = new HistoryRoute() // 当前路由
    this.initRoute() // 初始化路由函数
  }

  createMap(routes) {
    return routes.reduce((pre, current) => {
      pre[current.path] = current.component
      return pre
    }, {})
  }

  initRoute() {
    if (this.mode === 'hash') {
      // 先判断用户打开时有没有hash值，没有的话跳转到 #/
      location.hash ? '' : (location.hash = '/')
      window.addEventListener('load', () => {
        this.history.current = location.hash.slice(1)
      })
      window.addEventListener('hashchange', () => {
        this.history.current = location.hash.slice(1)
      })
    } else {
      // history模式
      location.pathname ? '' : (location.pathname = '/')
      window.addEventListener('load', () => {
        this.history.current = location.pathname
      })
      window.addEventListener('popstate', () => {
        this.history.current = location.pathname
      })
    }
  }
}

VueRouter.install = function (_Vue) {
  Vue = _Vue

  Vue.mixin({
    beforeCreate() {
      if (this.$options && this.$options.router) {
        this._root = this
        this._router = this.$options.router
        Vue.util.defineReactive(this, '_route', this._router.history) // 监听history路径变化
      } else if (this.$parent && this.$parent._root) {
        this._root = this.$parent && this.$parent._root
      }
      // 当访问this.$router时即返回router实例
      Object.defineProperty(this, '$router', {
        get() {
          return this._root._router
        },
      })
      // 当访问this.$route时即返回当前页面路由信息
      Object.defineProperty(this, '$route', {
        get() {
          return this._root._router.history.current
        },
      })
    },
  })
}

export default VueRouter
```

## router-link 和 router-view 组件

```javascript
VueRouter.install = function (_Vue) {
  Vue = _Vue

  Vue.component('router-link', {
    props: {
      to: {
        type: [String, Object],
        required: true,
      },
      tag: {
        type: String,
        default: 'a',
      },
    },
    methods: {
      handleClick(event) {
        // 阻止a标签默认跳转
        event && event.preventDefault && event.preventDefault()
        let mode = this._self._root._router.mode
        let path = this.to
        this._self._root._router.history.current = path
        if (mode === 'hash') {
          window.history.pushState(null, '', '#/' + path.slice(1))
        } else {
          window.history.pushState(null, '', path.slice(1))
        }
      },
    },
    render(h) {
      let mode = this._self._root._router.mode
      let tag = this.tag || 'a'
      let to = mode === 'hash' ? '#' + this.to : this.to
      console.log('render', this.to)
      return (
        <tag on-click={this.handleClick} href={to}>
          {this.$slots.default}
        </tag>
      )
      // return h(tag, { attrs: { href: to }, on: { click: this.handleClick } }, this.$slots.default)
    },
  })

  Vue.component('router-view', {
    render(h) {
      let current = this._self._root._router.history.current // current已经是动态响应
      let routesMap = this._self._root._router.routesMap
      return h(routesMap[current]) // 动态渲染对应组件
    },
  })
}
```

至此，一个简易的`vue-router`就实现完了，案例完整代码附上：[mini-vue-router](https://github.com/Jacky-Summer/mini-vue-router/blob/master/src/myrouter.js)

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~

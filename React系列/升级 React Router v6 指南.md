# 升级 React Router v6 指南

## 前言

近期完成了公司新项目的开发，相关的技术栈都用到了最新版本，react router 也使用了 v6 的版本，所以借这个机会自己再梳理下 react router v5 与 v6 的区别，以及 v6 一些新特性。而在原有项目还是使用老版本 react router 的情况下，不太建议急着直接升级，可能存在较多的改动。

## v5 升级 v6 指南

### `<Switch>`全部换成`<Routes>`

v5

```tsx
<BrowserRouter>
  <Menu />

  <Switch>
    <Route component={Home} path="/home"></Route>
    <Route component={List} path="/list"></Route>
    <Route component={Detail} path="/detail"></Route>
    <Route component={Category} path="/category"></Route>
  </Switch>
</BrowserRouter>

// Category.tsx
<Switch>
  <Route component={CategoryA} path="/category/a"></Route>
  <Route component={CategoryB} path="/category/b"></Route>
</Switch>
```

`Switch` 组件作用：渲染第一个被 location 匹配到的并且作为子元素的 `<Route>` 或者 `<Redirect>`，它仅仅只会渲染一个路径

v6

```tsx
<BrowserRouter>
  <Menu />

  <Routes>
    <Route element={<Home />} path="/home"></Route>
    <Route element={<List />} path="/list"></Route>
    <Route element={<Detail />} path="/detail"></Route>
    <Route element={<Category />} path="/category">
      {/* children 写法嵌套子路由，path是相对路径 */}
      <Route element={<CategoryA />} path="a"></Route>
      <Route element={<CategoryB />} path="b"></Route>
    </Route>
  </Routes>
</BrowserRouter>
```

与  `Switch`  相比，`Routes`  的主要优点是:

- `Routes` 内的所有 `<Route>` 和 `<Link>`的 path 支持相对路由（如果以`/`开头的则是绝对路由）。这使得 `<Route path>` 和 `<Link to>` 中的代码更精简、更可预测
- 路由基于最佳 path 匹配的，而不是按顺序遍历选择的
- 路由可以嵌套在同一个地方而不必分散在不同的组件中

注意：

- 不能认为 `Routes` 作为 `Switch` 的替代品。`Switch` 功能是匹配唯一的 `Route` 组件但它本身是可选的，可使用`Route`组件而不使用`Switch`组件。但只要使用`Route`组件则 v6 的`Routes`组件是必选的， `Routes` 必须套在最外层才可以使用`Route`组件，否则会报错。

### Route 组件属性

**`Route` 的 `render` 或 `component` 改为  `element`**

```tsx
// v5
<Route component={Home} path="/home"></Route>
// v6
<Route element={<Home />} path="/home"></Route>
```

**简化`path`格式，只支持两种动态占位符**

- `:id`  动态参数
- `*`  通配符，只能在 `path` 的**末尾**使用，如 `users/*`

v6 `path`的正确写法：

```
/groups
/groups/admin
/users/:id
/users/:id/messages
/files/*
/files/:id/*
```

v6 `path`错误的写法

```
/users/:id? // ? 不满足上面两种格式
/tweets/:id(\d+) // 有正则表达式，不满足
/files/*/cat.jpg
/files-*
```

**路由匹配的区分大小写开启  `caseSensitive`**

`caseSensitive`，用于正则匹配 `path` 时是否开启 ignore 模式，即匹配时是否忽略大小写

```tsx
<Routes caseSensitive>
```

**所有路径匹配都会忽略 URL 上的尾部斜杠**

### 新增 Outlet 组件

作用：通常用于渲染子路由，类似插槽的作用，用于匹配子路由的 `element`

```tsx
export default function Category() {
  return (
    <div>
      <div>
        <Link to="a">跳转 CategoryA</Link>
      </div>
      <div>
        <Link to="b">跳转 CategoryB</Link>
      </div>

      {/* 自动匹配子路由的渲染 */}
      <Outlet />
    </div>
  )
}
```

### Link 组件属性

**to 属性有无 / 与当前 URL 的区别**

在 v5 中，如果 `to` 没有以 `/` 开头的话会充满不确定性，这取决于当前的 URL。

比如当前 URL 是`/category`, `<Link to="a">` 会渲染成 `<a href="/a">`； 而当前 URL 如果是 `/category/`，那么又会渲染成 `<a href="/category/a">`。

在 v6 中，无论当前 URL 是  `/category`  还是  `/category/`， `<Link to="a">`  都会渲染成  `<a href='/category/a'>`，即忽略 URL 上的尾部斜杠统一规则处理。

**to 属性支持相对位置与'..' 和'.'等写法**

```tsx
<ul>
  <li>当前页面：CategoryA</li>
  <li>当前url：/category/a</li>
  <li>
    {/* /list */}
    <Link to="../../list">跳转到list页面</Link>
  </li>
  <li>
    {/* /category/b */}
    <Link to="../b">跳转到category/b页面</Link>
  </li>
  <li>
    {/* /category/a */}
    <Link to=".">跳转到当前路由</Link>
  </li>
</ul>
```

**直接传 state 属性**

```
// v5:
<Link to={{ pathname: "/home", state: state }} />

// v6:
<Link to="/home" state={state} />
```

**新增 target 属性**

```tsx
type HTMLAttributeAnchorTarget = '_self' | '_blank' | '_parent' | '_top' | (string & {})
```

### NavLink

- `<NavLink exact>`  属性名改为了  `<NavLink end>`
- 移除`activeStyle`、`activeClassName`属性

```tsx
<NavLink
  to="/messages"
- style={{ color: 'blue' }}
- activeStyle={{ color: 'green' }}
+ style={({ isActive }) => ({ color: isActive ? 'green' : 'blue' })}
>
  Messages
</NavLink>
```

### 移除`Redirect`重定向组件

移除的主要原因是[不利于 SEO](https://gist.github.com/mjackson/b5748add2795ce7448a366ae8f8ae3bb)

```tsx
// v5
<Redirect from="/404" to="/home" />

// v6 使用 Navigate 组件替代
<Route path="/404" element={<Navigate to="/home" replace />} />
```

### 新增 `useNavigate` 代替 `useHistory`

函数组件可以使用*useHistory*获取`history`对象，用来做页面跳转导航

```tsx
// v5
import { useHistory } from 'react-router-dom'

export default function Menu() {
  const history = useHistory()

  return (
    <div>
      <div
        onClick={() => {
          history.push('/list')
        }}
      >
        编程式路由跳转list页面
      </div>
    </div>
  )
}

// v6
import { useNavigate } from 'react-router-dom'

export default function Menu() {
  const navigate = useNavigate()

  return (
    <div>
      <div
        onClick={() => {
          navigate('/list') // 等价于 history.push
        }}
      >
        编程式路由跳转list页面
      </div>
    </div>
  )
}
```

下面再列举其它其它的区别用法

```tsx
//v5
history.replace('/list')
// v6
navigate('/list', { replace: true })

// v5
history.go(1)
history.go(-1)

// v6
navigate(1)
navigate(-1)
```

### 新增 `useRoutes` 代替 `react-router-config`

`useRoutes` 根据路由表生成对应的路由规则

> `useRoutes`使用必须在`<Router>`里面

- [react-router-config](https://www.npmjs.com/package/react-router-config)：用于集中管理路由配置

```tsx
import { useRoutes } from 'react-router-dom'
import Home from './components/Home'
import List from './components/List'

function App() {
  const element = useRoutes([
    { path: '/home', element: <Home /> },
    { path: '/list', element: <List /> },
  ])

  return element
}

export default App
```

### 新增 `useSearchParams`

v6 提供  `useSearchParams`  返回一个数组来获取和设置 url 参数

```tsx
import { useSearchParams } from 'react-router-dom'

export default function Detail() {
  const [searchParams, setSearchParams] = useSearchParams()

  console.log('getParams', searchParams.get('name'))
  return (
    <div
      onClick={() => {
        setSearchParams({ name: 'jacky' })
      }}
    >
      当前页面：Detail 点我设置url查询参数为name=jacky
    </div>
  )
}
```

### 不支持 `<Prompt>`

在老版本中，`Prompt`组件可以实现页面关闭的拦截，但它在 v6 版本还暂不支持，如果想 v5 升级 v6 就要考虑清楚了。

```
// v5
<Prompt
  when={formIsHalfFilledOut}
  message="Are you sure you want to leave?"
/>
```

## 总结

v5 和 v6 在使用层面的区别总结：

- `<Switch>` 全部换成 `<Routes>`
- Route 新特性变更
  - `render` 和 `component` 改为 `element`，且支持嵌套路由
  - `path` 支持相对路径；简化`path`格式，只支持两种动态占位符
  - 路由匹配的区分大小写开启  `caseSensitive`
  - 所有路径匹配都会忽略 URL 上的尾部斜杠`/`
- 新增 `Outlet` 组件用于渲染匹配到的子路由
- 移除`Redirect`重定向组件，因为不利于 SEO
- 新增 `useNavigate` 替代 `useHistory`
- 新增 `useRoutes` 代替 `react-router-config`
- 新增 `useSearchParams` 来获取和设置 url 参数

## 参考文章

- [React Router](https://reactrouter.com/en/main/getting-started/overview)
- [upgrade-to-react-router-v6](https://github.com/remix-run/react-router/blob/main/docs/upgrading/v5.md#upgrade-to-react-router-v6)

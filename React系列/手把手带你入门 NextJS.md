# 手把手带你入门 Next.js (v9.5.2)

## 前言

Next.js 之前用过一次，这次是重新做个小回顾，现在最新版本已经到了 9.5.3，有些 API 也同以前有点不同了，网上大部分教程也都是旧版本 v7 的比较多，故打算写下简单的教程，绝对详细的带你入个小门。

## 库版本

本文案例用的关键库版本如下：

```
"next": "^9.5.2",
"react": "^16.13.1",
```

node 版本为 12.18.1（nodejs 版本 >= 10.13 即可）

## 初始化 Next.js 项目

1. 新建一个文件夹

如`learn-nextjs-example`，先进行初始化

```
npm i -y
```

2. 安装所需要的依赖包

```
npm i react react-dom next --save
```

3. 添加 script 命令

为了是开发输入 npm 命令更快捷，所以把常用的命令作为快捷命令设置

```
"scripts": {
  "dev": "next",
  "build": "next build",
  "start": "next start"
},
```

4. 创建 pages 文件夹，运行第一个页面

> pages 文件夹是放置页面的，这里边的文件会自动生成相应路由。

- 比如创建 `pages/about.js`，那么访问该页面地址就是 `http://localhost:3000/about`；
- 比如创建 `pages/about/about.js`，那么访问地址为`http://localhost:3000/about/about`。

现在我们创建 `pages/index.js`，index.js 就是默认代表根路径了

```javascript
const Home = () => {
  return <div>Hello Next.js!</div>
}

export default Home
```

此时运行

```
npm rum dev
```

打开`http://localhost:3000/`，可以看到页面成功运行了，也可以知道 Next.js 内置 React，因此不用我们再引入
`import React from 'react'`，就可以直接使用 React 的语法，当然你写了也不会报错，不过没必要。

## 路由跳转

页面跳转有两种形式，一种是利用标签`Link`，一种是编程式路由跳转`router.push('/')`。路由跳转方式还是跟 React 很像的，只不过用的是 `next` 的包

### Link

1. 先创建两个页面

`pages/about.js`

```javascript
const About = () => {
  return <div>About Page</div>
}

export default About
```

`pages/news.js`

```javascript
const News = () => {
  return <div>News Page</div>
}

export default News
```

2. 在首页`pages/index.js`编写路由跳转链接

```javascript
import Link from 'next/link'

const Home = () => {
  return (
    <div>
      <div>Hello Next.js!</div>
      <div>
        <Link href='/about'>
          <a>关于</a>
        </Link>
      </div>
      <div>
        <Link href='/news'>
          <a>新闻</a>
        </Link>
      </div>
    </div>
  )
}
export default Home
```

在这里要注意的是 a 标签不用添加 href 元素，添加了也不起作用。
如果`Link`里面换成其他标签如`span`，也依然能成功跳转到对应页面，可以知道`Link`是给里面的元素添加了点击绑定事件，而不是只会给 a 标签添加 href 而已。

此时 DOM 结构：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/de05d989ca674288856386d286ad3052~tplv-k3u1fbpfcp-zoom-1.image)

还有要注意的是：`Link` 里面只能有一个根元素，要不然它不知要绑定谁会报错。

### 编程式路由跳转

这里主要用`useRouter`钩子进行跳转

修改`pages/index.js`，修改其中一个路由

```javascript
import Link from 'next/link'
import { useRouter } from 'next/router'

const Home = () => {
  const router = useRouter()
  const gotoAbout = () => {
    router.push('/news')
  }
  return (
    <div>
      <div>Hello Next.js!</div>
      <div>
        <Link href='/about'>
          <a>关于</a>
        </Link>
      </div>
      <div>
        <button onClick={gotoAbout}>新闻</button>
      </div>
    </div>
  )
}
export default Home
```

## 路由传参

### query 形式传参

在 Next.js 中只能通过 query（?name=jackylin）来传递参数，不能通过(path:name)的形式传递参数。
修改`pages/index.js`

```javascript
import Link from 'next/link'
import { useRouter } from 'next/router'

const Home = () => {
  const router = useRouter()
  const gotoAbout = () => {
    router.push('/news')
  }
  return (
    <div>
      <div>Hello Next.js!</div>
      <div>
        <Link href='/about?name=jackylin'>
          <a>关于</a>
        </Link>
      </div>
      <div>
        <button onClick={gotoAbout}>新闻</button>
      </div>
    </div>
  )
}
export default Home
```

也可以变换成这种方式传

```
<Link href={{ pathname: '/about', query: { name: 'jackylin' } }}>
```

传递了参数`name`后，接下来是接收参数。

打开`pages/about.js`添加代码

```javascript
import { withRouter } from 'next/router'

const About = ({ router }) => {
  return (
    <div>
      <div>About Page</div>
      <div>接收到参数为：{router.query.name}</div>
    </div>
  )
}

export default withRouter(About)
```

点击`关于`，跳转到 about 页面，页面内容为：

```
About Page
接收到参数为：jackylin
```

同时也可看到地址栏带有参数：`http://localhost:3000/about?name=jackylin`

### 编程式路由跳转传递参数

修改`pages/index.js`中的 `gotoAbout`方法

```javascript
const gotoAbout = () => {
  router.push({
    pathname: '/news',
    query: {
      info: '学习Next.js',
    },
  })
}
```

`pages/news.js`

```javascript
import { withRouter } from 'next/router'

const News = ({ router }) => {
  return (
    <div>
      <div>News Page</div>
      <div>接收到参数为：{router.query.info}</div>
    </div>
  )
}

export default withRouter(News)
```

## 路由变化的钩子

这里就不一个个细细展开了，直接代码说话：

```javascript
import { useEffect } from 'react'
import Link from 'next/link'
import { useRouter } from 'next/router'

const Home = () => {
  const router = useRouter()
  const gotoAbout = () => {
    router.push({
      pathname: '/news',
      query: {
        info: '学习Next.js',
      },
    })
  }

  useEffect(() => {
    const handleRouteChangeStart = url => {
      console.log('routeChangeStart 路由开始变化，url:', url)
    }

    const handleRouteChangeComplete = url => {
      console.log('routeChangeComplete 路由结束变化，url:', url)
    }

    const handleBeforeHistoryChange = url => {
      console.log('beforeHistoryChange 在改变浏览器 history之前触发，url:', url)
    }

    const handleRouteChangeError = (err, url) => {
      console.log(`routeChangeError 跳转发生错误: ${err}, url:${url}`)
    }

    const handleHashChangeStart = url => {
      console.log('hashChangeStart hash路由模式跳转开始时执行，url:', url)
    }

    const handleHashChangeComplete = url => {
      console.log('hashChangeComplete hash路由模式跳转完成时，url:', url)
    }

    router.events.on('routeChangeStart', handleRouteChangeStart)
    router.events.on('routeChangeComplete', handleRouteChangeComplete)
    router.events.on('beforeHistoryChange', handleBeforeHistoryChange)
    router.events.on('routeChangeError', handleRouteChangeError)
    router.events.on('hashChangeStart', handleHashChangeStart)
    router.events.on('hashChangeComplete', handleHashChangeComplete)

    return () => {
      router.events.off('routeChangeStart', handleRouteChangeStart)
      router.events.off('routeChangeComplete', handleRouteChangeComplete)
      router.events.off('beforeHistoryChange', handleBeforeHistoryChange)
      router.events.off('routeChangeError', handleRouteChangeError)
      router.events.off('hashChangeStart', handleHashChangeStart)
      router.events.off('hashChangeComplete', handleHashChangeComplete)
    }
  }, [])

  return (
    <div>
      <div>Hello Next.js!</div>
      <div>
        <Link href='/about?name=jackylin'>
          <a>关于</a>
        </Link>
      </div>
      <div>
        <button onClick={gotoAbout}>新闻</button>
      </div>
    </div>
  )
}
export default Home
```

## style-jsx 编写页面 CSS 样式

我们建立一个头部组件再引入`pages/index.js`，在根目录下新建`components/header.js`

```javascript
const Header = () => {
  return (
    <div>
      <div className='header-bar'>Header</div>
    </div>
  )
}

export default Header
```

`pages/index.js`引入 Header 组件

```javascript
import Header from '../components/header'

const Home = () => {
  return (
    <div>
      <Header />
    </div>
  )
}
```

接着给 Header 写样式，Next.js 内置支持 style jsx 语法，这是其中一种 CSS-in-JS 的解决方案

```javascript
const Header = () => {
  return (
    <div>
      <div className='header-bar'>Header</div>
      <style jsx>
        {`
          .header-bar {
            width: 100%;
            height: 50px;
            line-height: 50px;
            background: lightblue;
            text-align: center;
          }
        `}
      </style>
    </div>
  )
}

export default Header
```

运行样式生效
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/27bcc345b1bf4fdcb50e582310abc0d7~tplv-k3u1fbpfcp-zoom-1.image)

查看 DOM 结构，会发现 Next.js 会自动加入一个随机类名（jsx-xxxxxxx），这样就能防止 CSS 的全局污染。
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2de14d7f46694782a151d440ecb2626f~tplv-k3u1fbpfcp-zoom-1.image)

## 添加全局 CSS 文件

新建文件夹 public，用于放置静态文件如图片和 css 文件等，注意只有名为`public`的目录能够存放静态资源并对外提供访问。新建文件`public/static/styles/common.css`

```css
body {
  margin: 0;
  padding: 0;
  font-size: 16px;
}
```

- 自定义 App

Next.js 使用`App`组件来初始化页面，我们可以覆盖`App`组件来控制页面的初始化。该`App`作用一般如下：

- 在页面切换之间保持布局的持久化
- 切换页面时保持状态
- 使用 componentDidCatch 自定义错误处理
- 向页面注入额外的数据
- 添加全局 CSS

我们要做的就是添加全局 CSS 样式，所以需要覆盖原来默认的`App`，首先要创建`pages/_app.js`

```javascript
import '../public/static/styles/common.css'

/**
 * Component 指当前页面，每次路由切换时，Component 都会更新
 * pageProps 是带有初始属性的对象，该初始属性由我们的某个数据获取方法预先加载到你的页面中，否则它将是一个空对象
 */
export default function MyApp({ Component, pageProps }) {
  return <Component {...pageProps} />
}
```

重启服务，运行，全局样式添加成功。

## 集成 style-components

如果想要项目支持 style-components 的话，需要我们自己去配置。我们通过自定义 Document 的方式来改写代码，即是指`_document.js`文件，它只有在服务器端渲染的时候才会被调用，主要用来修改服务器端渲染的文档内容，一般用来配合第三方 css-in-js 方案使用。

- 安装所需库

```
npm i styled-components --save
```

编译 styled-components

```
npm i babel-plugin-styled-components --save-dev
```

新建文件`.babelrc`

```
{
  "presets": ["next/babel"],
  "plugins": [["styled-components", { "ssr": true }]]
}
```

新建 `_document.js`，改写覆盖它原来的写法。这里要注意的是，要一定要继承 `Document`，才来改写

```javascript
import Document from 'next/document'
import { ServerStyleSheet } from 'styled-components'

export default class MyDocument extends Document {
  static async getInitialProps(ctx) {
    const sheet = new ServerStyleSheet()
    const originalRenderPage = ctx.renderPage
    try {
      ctx.renderPage = () =>
        originalRenderPage({
          enhanceApp: App => props => sheet.collectStyles(<App {...props} />),
        })
      const initialProps = await Document.getInitialProps(ctx)

      return {
        ...initialProps,
        styles: (
          <>
            {initialProps.styles}
            {sheet.getStyleElement()}
          </>
        ),
      }
    } finally {
      sheet.seal()
    }
  }
}
```

这其中详细配置意思可以去官方文档找点线索了解：[Next.js](https://nextjs.org/docs/advanced-features/custom-document)

接下来我们使用 styled-components 的方式添加样式，打开`pages/about.js`

```javascript
import { withRouter } from 'next/router'
import styled from 'styled-components'

const Title = styled.h1`
  color: green;
`

const About = ({ router }) => {
  return (
    <div>
      <Title>About Page</Title>
      <div>接收到参数为：{router.query.name}</div>
    </div>
  )
}

export default withRouter(About)
```

运行，如下图：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e914adfd47af4564a3c06b641f514b8e~tplv-k3u1fbpfcp-zoom-1.image)

## 获取数据的方式

四个 api 都只能在 pages 文件夹内的文件中使用

### getStaticProps

**当页面内容取决于外部数据**

Next.js 推荐我们尽可能使用静态生成的方式，因为所有页面都可以只构建一次并托管到 CDN 上，这比让服务器根据每个页面请求来渲染页面快得多。

比如下列一些类型的页面：

- 营销页面
- 个人博客
- 产品列表
- 静态文档

如果一个页面使用了 静态生成（SSG），在构建时将生成此页面对应的 HTML 文件 。HTML 文件将在每个页面请求时被重用，还可以被 CDN 缓存。这个跟 hexo，Gatsby 类似的，生成静态文件还有 json 文件。

关于 Gatsby，它是基于 React 的上层框架，具体可以看我这篇博客了解：[手把手带你入门 Gatsby](https://juejin.im/post/6850418110885789704)

Next.js 会尽可能地自动优化应用并输出静态 HTML，如果你在你的页面组件添加了`getInitialProps`方法才会禁用自动静态优化，`getInitialProps`方法下面会说到。

比如我们可以执行命令查看

```
npm run build
```

打包默认是 SSG 模式：

`● (SSG) automatically generated as static HTML + JSON (uses getStaticProps)`

会发现`.next/server/pages/`下多出了几个 HTML 文件，这些就是静态构建出的 HTML 文件。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/06f57ad8a7c046efbcbbbb90032f0396~tplv-k3u1fbpfcp-zoom-1.image)

新建文件`pages/blog.js`

```javascript
const Blog = ({ posts }) => {
  return <div>title: {posts.title}</div>
}

// 此函数在构建时被调用
export async function getStaticProps() {
  // 调用外部 API 获取内容
  const res = await fetch('https://jsonplaceholder.typicode.com/todos/1')
  const posts = await res.json()

  // 在构建时将接收到 `posts` 参数
  return {
    props: {
      posts,
    },
  }
}

export default Blog
```

进入`http://localhost:3000/blog`，发现页面能正常获取到请求结果：

```
title: delectus aut autem
```

### getStaticPaths

**当页面的路径取决于外部数据**

这种方式和上面就有些不同了，上面`http://localhost:3000/blog`访问路径时我们已经写了一个 js 页面文件，但如果我们想要根据获取的数据动态生成多篇文章路径的话，那就需要用到这个 API，通常用这个 API 会同时配合`getStaticProps`一起用。

新建文件`pages/posts/[id].js`，你没有看错，文件名就是`[id].js`

用`id`标识单篇博文，比如你访问`http://localhost:3000/posts/1`就展示`id`为 1 的文章。

代码如下：

```javascript
const Post = ({ post }) => {
  return (
    <div>
      <div>文章id: {post.id}</div>
      <div>博客标题: {post.title}</div>
    </div>
  )
}

// 构建路由
export async function getStaticPaths() {
  // 调用外部 API 获取博文列表
  const res = await fetch('https://jsonplaceholder.typicode.com/todos')
  const posts = await res.json()

  // 根据博文列表生成所有需要预渲染的路径
  const paths = posts.map(post => `/posts/${post.id}`)

  // fallback为false，表示任何不在 getStaticPaths 的路径的结果将是 404 页面。
  return { paths, fallback: false }
}

// 获取单个页面博文数据
export async function getStaticProps({ params }) {
  // 如果路由是 /posts/1，那么 params.id 就是 1
  const res = await fetch(`https://jsonplaceholder.typicode.com/todos/${params.id}`)
  const post = await res.json()

  // 通过 props 参数向页面传递博文的数据
  return { props: { post } }
}

export default Post
```

拉取下来的数据全部有 200 条，也就是说动态构建了 200 个路由，现在我们访问第 50 篇博文的话，直接输入`http://localhost:3000/posts/50`

页面显示：

```
文章id: 50
博文标题: cupiditate necessitatibus ullam aut quis dolor voluptate
```

接下来让我们再次打包，执行`npm run build`（如果打包发生错误很可能是网络差和请求链接有关，因为要请求很多页面，如果是网络问题多试几次就行），你会发现这次打包时间变长了，因为要构建的静态页面多了，我们请求返回来的数据有 200 条，所以应该生成 200 个静态 HTML 文件和相应的 json 数据文件。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/088770a941f1479fa15fc96d348f85a4~tplv-k3u1fbpfcp-zoom-1.image)

我们来看 `1.json` 文件，就是请求后的数据

```
{"pageProps":{"post":{"userId":1,"id":1,"title":"delectus aut autem","completed":false}},"__N_SSG":true}
```

### getServerSideProps

**每次页面请求时重新生成页面的 HTML**

如果无法在用户请求之前预渲染页面，那上面的"静态生成"就不太适用了，或者是页面数据需要频繁的更新，并且页面内容会随着每个请求而变化。这个时候，有两种方案：

- 将“静态生成”与 客户端渲染 一起使用：你可以跳过页面某些部分的预渲染，然后使用客户端 JavaScript 来填充它们
- 使用 服务器端渲染： Next.js 针对每个页面的请求进行预渲染。由于 CDN 无法缓存该页面，因此速度会较慢，但是预渲染的页面将始终是最新的。

由于服务器端渲染会导致性能比“静态生成”慢，因此仅在绝对必要时才使用此功能。

新建文件`pages/server-blog.js`

```javascript
const Blog = ({ posts }) => {
  return <div>title: {posts.title}</div>
}

// 在每次页面请求时都会运行，而在构建时不运行。
// 要设置某个页面使用服务器端渲染，就需要导出 getServerSideProps 函数
export async function getServerSideProps() {
  // 调用外部 API 获取内容
  const res = await fetch('https://jsonplaceholder.typicode.com/todos/1')
  const posts = await res.json()

  // 在构建时将接收到 `posts` 参数
  return {
    props: {
      posts,
    },
  }
}

export default Blog
```

运行`npm run build`，发现这次并没有生成对应的 html 文件——`server-blog.html`，因为我们指定了服务端渲染，请求构建时不会执行，不是静态生成。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2717dff8c6a0485d836091e87f9093c0~tplv-k3u1fbpfcp-zoom-1.image)

### getInitialProps

getInitialProps 是在渲染页面之前就会运行的 API。 如果该路径下包含该请求，则执行该请求，并将所需的数据作为 props 传递给页面。当第一次访问直接页面，getInitialProps 就在服务器端运行，加载完毕后页面给客户端托管，使用客户端的路由跳转，之后页面的 getInitialProps 就在客户端执行了。

> 推荐： getStaticProps 或 getServerSideProps。如果你使用的是 Next.js 9.3 或更高版本，我们建议你使用 getStaticProps 或 getServerSideProps 来替代 getInitialProps。这些新的获取数据的方法使你可以在静态生成（static generation）和服务器端渲染（server-side rendering）之间进行精细控制。

新建文件`pages/initial-blog.js`

```javascript
import Link from 'next/link'

const InitialBlog = ({ post }) => {
  return (
    <div>
      <h1>getInitialProps Demo Page</h1>
      <div>获取到的title: {post.title}</div>
      <Link href='/'>
        <a>返回首页</a>
      </Link>
    </div>
  )
}

InitialBlog.getInitialProps = async ctx => {
  const res = await fetch('https://jsonplaceholder.typicode.com/todos/1')
  const post = await res.json()
  return { post }
}

export default InitialBlog
```

再修改`pages/index.js`代码，增加一个`Link`标签可以跳转回该页面

```javascript
<div>
  <Link href='/initial-blog'>
    <a>进入Initial Blog</a>
  </Link>
</div>
```

接着直接地址栏输入`http://localhost:3000/initial-blog`，在渲染该页面内容前，先会执行`getInitialProps`方法，执行完的结果再传到页面组件，开始渲染页面。由于我们是第一次访问页面，首屏则为服务端渲染。

页面内容显示为：

```
getInitialProps Demo Page
获取到的title: delectus aut autem
```

打开浏览器的 Network 面板，找到 Name 值为`initial-blog`的请求，会看到请求响应内容的正是该页面的 HTML。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4171887a1ccc44e88da72bd43266b35b~tplv-k3u1fbpfcp-zoom-1.image)

点击`返回首页`，再从首页点击`进入Initial Blog`，重新进入该页面，注意的是此时已经不是我们浏览器通过地址栏访问该页面，而是通过前端路由进入该页面。

查看 Network 面板的请求，会看到此时返回的是`getInitialProps`请求回来的数据，而不是 HTML 页面内容，由此可知此时页面已经交由客户端渲染。这就是所谓 SSR 渲染，首屏交给服务端渲染返回，返回后页面跳转就是客户端渲染了。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2da317e74c0482ab58d19ee7bdc51a1~tplv-k3u1fbpfcp-zoom-1.image)

## 自定义 Head

之所以会选择 Next.js，相信一个非常重要的原因就是有利用 SEO，那么利于爬虫检索的方式，一种就是在页面写好 TDK（title、description、keyword），Next.js 可以自定义`Head`标签

修改`components/header.js`，在最外层 div 根元素里内添加代码

```javascript
<Head>
  <title>Next.js 教程 -- JackyLin</title>
  <meta name='viewport' content='initial-scale=1.0, width=device-width' />
</Head>
```

打开`http://localhost:3000/`，就会看到页面的头部标签 title 已经被我们改了。

## LazyLoding 实现模块/组件懒加载

懒加载是指需要用到或该加载到某个组件/模块的时候，才加载那个组件/模块的 js 文件，也叫异步加载。

如果页面文件内容多过大，那么可能出现首次打开速度慢，这时就需要进行优化了，懒加载就是其中一种方式。

### 懒加载模块

先安装一个处理日期时间的库

```
npm i moment --save
```

新建`pages/time.js`

```javascript
import { useState } from 'react'
import moment from 'moment'

const Time = () => {
  const getTime = () => {
    setNowTime(moment().format('YYYY-MM-DD HH:mm:ss'))
  }
  const [nowTime, setNowTime] = useState('')
  return (
    <div>
      <button onClick={getTime}>获取当前时间</button>
      <div>当前时间：{nowTime}</div>
    </div>
  )
}

export default Time
```

假设我们这个页面内容很多，然后你进入该页面可能不需要点击获取时间，但`moment`模块依然加载了，这就有点资源浪费。
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b24abdbe08d64eb7ae5585dcede06dee~tplv-k3u1fbpfcp-zoom-1.image)

于是，我们想当我们点击按钮的时候，再来加载`moment`模块，即是实现异步加载，而不是页面一进入就加载。

此时点击按钮获取时间，可看到 Network 面板看到`moment`的代码被打包成了一个`1.js`文件，这样就实现了懒加载了，这样可以减少了主要 JavaScript 包的大小,带来了更快的加载速度。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/94d858ae0d074bd4939864f6105d4f3b~tplv-k3u1fbpfcp-zoom-1.image)

```javascript
import { useState } from 'react'

const Time = () => {
  const getTime = async () => {
    const moment = await import('moment')
    setNowTime(moment.default().format('YYYY-MM-DD HH:mm:ss')) // 注意这里使用 default，不能直接用 moment()
  }
  const getTime = () => {
    setNowTime(moment().format('YYYY-MM-DD HH:mm:ss'))
  }
  const [nowTime, setNowTime] = useState('')
  return (
    <div>
      <button onClick={getTime}>获取当前时间</button>
      <div>当前时间：{nowTime}</div>
    </div>
  )
}

export default Time
```

### 懒加载组件

懒加载组件需要引入 `next/dynamic`模块

新建组件`components/content.js`

```javascript
const Content = () => {
  return <div>content</div>
}

export default Content
```

引入`pages/time.js`

```javascript
import { useState } from 'react'
import dynamic from 'next/dynamic'

// 懒加载自定义组件
const Content = dynamic(() => import('../components/content'))

const Time = () => {
  const getTime = async () => {
    const moment = await import('moment')
    setNowTime(moment.default().format('YYYY-MM-DD HH:mm:ss')) // 注意这里使用 default，不能直接用 moment()
  }
  const [nowTime, setNowTime] = useState('')
  return (
    <div>
      <button onClick={getTime}>获取当前时间</button>
      <div>当前时间：{nowTime}</div>
      <Content />
    </div>
  )
}

export default Time
```

观察 Network 可以看到 Content 组件被新单独打包成一个 js 文件

## 打包生产环境

执行

```
npm run build
```

打包完成后，运行服务器

```
npm run start
```

然后查看页面，打包成功！

参考：

- [Server side rendering Styled-Components with NextJS](https://medium.com/swlh/server-side-rendering-styled-components-with-nextjs-1db1353e915e)

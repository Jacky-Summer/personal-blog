# 手把手带你入门 Gatsby

## Gatsby 介绍

### 什么是 Gatsby

Gatsby 是一个基于 React ，用于搭建静态站点的开源框架，用于帮助 开发者构建运行速度极快的网站。可以说它是一个静态站点生成器，Gatsby 主要的应用的技术是 React 和 GraphQL。

官网：https://www.gatsbyjs.org/

### 为什么选 Gatsby

Gatsby 能快速使用 React 生态系统来生成静态网站，它具备更高的性能，而且 Gatsby 的生态也很强大。

当你想自己动手搭建个人博客时，考虑的是 SEO 要好，而且你不用理会数据库和服务器等复杂的部署设置，Gatsby 构建的网站为静态站点，你可以很轻松的将网站部署在很多服务上。Gatsby 不需要等待请求时生成页面，而是预先生成页面，因此网站的速度会很快。

Gatsby 中运用了 React, react-router, Webpack 以及 GraphQL 等新技术，也跟随了技术潮流。

## GraphQL 介绍

上面我们说到了 GraphQL，没了解的同学可能不太清楚。

> GraphQL 是一种用于 API 的查询语言，是 Restful API 的替代品。

至于替代 Restful API 一说，个人觉得现在 Restful API 占据绝对的主导地位，当前还是很难被撼动的。因为目前来看，GraphQL 也有它的缺点。

看到这里没学过 GraphQL 也不用怕，因为用到的不多而且 Gatsby 内置一个界面方便我们操作。GraphQL 目前国外比较火，它作为一门新技术，即使不深入我们也可以了解一下。

这里我们当然要吹捧它的优点，上面的定义你可能会疑惑，API 不就是后端写好的接口吗，怎么还能查询？

GraphQL 我们把这个单词拆开的话是： Graph(图形或图表) + QL（Query Language），它是一种图形化的查询语言，描述客户端如何像服务端请求数据的语法规范。听起来模模糊糊，那它相对 Restful API 又解决了什么痛点呢？

1. 请求你所要的数据不多不少

相信多数人肯定遇到过的场景，比如项目管理系统，进入某页面展示一个列表，列表上只需展示标题，但你请求返回的数据却多了其他信息，比如创建者、创建时间等。这些信息在这里是多余的，但它可能有它自己存在的理由，或许是其他场景需要返回这些信息，因此该接口就直接包含了所有数据。

这一点，GraphQL 能准确获取你想要的数据，不多不少，想要返回指定的什么数据，就返回什么数据。数据由应用来控制，而不是服务器。

2. 获取多个资源只需要一个请求

GraphQL 查询不仅能够获得资源的属性，还能沿着资源间引用进一步查询。典型的 REST API 请求多个资源时得载入多个 URL，而 GraphQL 可以通过一次请求就获取你应用所需的所有数据。这样一来，即使是比较慢的移动网络连接下，使用 GraphQL 的应用也能表现得足够迅速。

3. 描述所有的可能类型系统

GraphQL API 基于类型和字段的方式进行组织，而非入口端点。你可以通过一个单一入口端点得到你所有的数据能力。GraphQL 使用类型来保证应用只请求可能的数据，还提供了清晰的辅助性错误信息。应用可以使用类型，而避免编写手动解析代码。

Gatsby 使用了 GraphQL，作为在本地管理资源的一种方式。

## 搭建项目

### 开始

- 安装 Gatsby 脚手架

```
npm install -g gatsby-cli
```

- 新建项目

> gatsby new [projectName][starter]：根据 starter 创建一个新项目

```
gatsby new gatsby-blog-demo https://github.com/gatsbyjs/gatsby-starter-hello-world
```

我们这里使用官方最简单版本的 hello-world 模板进行开发，如果你直接使用

```
gatsby new gatsby-blog-demo
```

默认会使用 [gatsby-starter-default](https://github.com/gatsbyjs/gatsby-starter-default) 来新建项目

- 运行项目

```
cd gatsby-blog-demo
gatsby develop
```

打开 `localhost:8000`，就可以看到输出了一句`Hello world!`

### 常用命令

这里介绍 Gatsby 的几个常用命令：

- gastby develop：开启热加载开发环境
- gastby build：打包到 public 下，构建生产环境用的优化过的静态网站所需的一切静态资源、静态页面与 js 代码
- gatsby serve：在打包之后，启动一个本地的服务，供你测试刚才"gatsby build"生成的静态网页

### GraphiQL

打开`http://localhost:8000/__graphql`，就会看到 GraphQL 的调试界面，这里可以查看所有的数据、以及调试 query 语句是否返回相应的数据。可以直接在左侧点击选中，就可以自动生成 query 语句。

![](https://user-gold-cdn.xitu.io/2020/6/21/172d4b8f913506df?w=1922&h=538&f=png&s=78717)

### 可能出现的警告

如果运行后你的控制台出现：

```
React-Hot-Loader: react-🔥-dom patch is not detected. React 16.6+ features may not work.
```

可以这样解决:

```
npm install @hot-loader/react-dom
```

在根目录下新建`gatsby-node.js`

```
exports.onCreateWebpackConfig = ({ getConfig, stage }) => {
  const config = getConfig()
  if (stage.startsWith('develop') && config.resolve) {
    config.resolve.alias = {
      ...config.resolve.alias,
      'react-dom': '@hot-loader/react-dom'
    }
  }
}
```

再重新启动项目，就可以消除这个警告

### 项目结构

```
├── .cache
├── public
├── src
|  └── pages            // 该文件夹下的文件会被映射为路由
|     └── index.js
├── static              // 静态资源
├── .prettierignore
├── .prettierrc
├── gatsby-config.js    // 基本配置文件
├── LICENSE
├── package.json
├── README.md
└── yarn.lock
```

## Gatsby 配置文件

这里最初始的模板只有一个配置文件，一般项目中都会有 4 个配置文件

### gatsby-config.js

基本配置文件。整个 Gatsby 站点的配置文件。可以在里面配置网站的相关基本信息。

### gatsby-node.js

node 相关配置文件。这个文件只会在 build 期间运行一次，比如动态创建一些页面。

### gatsby-browser.js

客户端相关配置。一些浏览器相关的 API 通过在这个文件里去实现，比如一些监听路由变化，注册 serviceWorker 等等。

### gatsby-ssr.js

服务端渲染相关配置文件。使用 Gatsby 的服务端渲染 API 自定义影响服务端渲染的默认设置。

## 创建页面

### 路由页

我们在 `pages`目录下创建一个文件`about.js`，则对应的路由路径为`/about`

```javascript
import React from 'react'

const About = () => {
  return <div>about me</div>
}

export default About
```

打开`http://localhost:8000/about`，显示的是我们刚刚创建的页面

### 404 页面

`pages`目录下的文件名即是路由路径，但有一个比较特殊，当匹配不到路径时，Gatsby 会跳转到 404 页面，如果我们在 pages 下创建文件名`404.js`，会使用我们自定义的 404 页面。当前我们随便输入一个不存在的页面路径：`http://localhost:8000/test`，页面就会报错。

我们新建一个文件 404.js

```javascript
import React from 'react'

const About = () => {
  return <div>404 页面不存在</div>
}

export default About
```

再随便输入一个路径，显示出该页面：
![](https://user-gold-cdn.xitu.io/2020/6/21/172d52ba499ad501?w=1223&h=404&f=png&s=37041)

Gatsby 会确保将 404 页面构建为 404.html，默认情况下会为我们创建此页面，在该页面也列出了网站上的所有页面路由链接,我们可以通过单击`Preview custom 404 page`看我们刚才创建的 404 页面。

那我们想直接跳转到我们自定义的 404 页面啊，还要我们点击才跳转。原因是处于开发环境，即当使用 `gatsby develop`时，Gatsby 使用默认的 404 页面覆盖我们自定义的 404 页面，但实际上是已经生效了。

我们可以打包，启动一个本地的服务，模拟处于线上环境来看，这个时候才是直接显示我们自定义的 404 页面。

```
gatsby build
gatsby serve
```

然后打开`http://localhost:9000/test`（任意路径），就能跳转到我们自定义的 404 页面了

![](https://user-gold-cdn.xitu.io/2020/6/21/172d5e52ae8fcd52?w=334&h=102&f=png&s=3896)

### 创建布局组件

1. 创建一个新目录 `src/components`。

2. 在上面的目录中创建一个布局组件文件 src/components/layout.js：

```javascript
import React from 'react'

export default ({ children }) => (
  <div style={{ margin: `3rem auto`, maxWidth: 650, padding: `0 1rem` }}>{children}</div>
)
```

3. 在`src/pages/index.js`中，加入 Layout 组件，

```javascript
import React from 'react'
import Layout from '../components/layout'

export default function Home() {
  return <Layout>Hello world!</Layout>
}
```

这样就让整体页面居中了，这里都是 React 的基本操作

## 设置 CSS 样式

Gatsby 中样式可以使用多种形式，普通的 `import './xxx.css`我们就不说了。

### 使用全局样式

我们向网站添加全局样式的最直接的方法之一就是使用全局 .css 样式表。

在 src 目录下新建文件夹`styles`，添加一个 global.css 文件

```
html {
  background-color: #f5f5f5;
}
```

- 在目录下创建 gatsby-browser.js 中，将样式文件导入

```
import "./src/styles/global.css"
```

重启本地服务，就发现页面背景变为浅灰色了。

### 使用 CSS Module 为组件设置样式

CSS Module 可以减少全局污染、防止命名冲突和样式的覆盖。

> CSS Module 实现原理是：将所有模块化的类名，修改为成唯一的名称，即添加一些不重复的前缀或后缀。这样使得项目中，不同人员编写的样式不会出现覆盖和冲突。

1. 创建 CSS 文件 `src/styles/index.module.css`

```
.title {
  color: red;
}
.text {
  color: blue;
}
```

用法如下：`src/pages/index.js`

```
import React from "react"
import Layout from "../components/layout"
import styles from "../styles/index.module.css"

export default function Home() {
  console.log(styles)
  return (
    <Layout>
      <h1 className={styles.title}>Hello World</h1>
      <div className={styles.text}>Test</div>
    </Layout>
  )
}
```

运行结果：

![](https://user-gold-cdn.xitu.io/2020/6/21/172d61d3f95cadf0?w=718&h=174&f=png&s=5935)

输出 `styles`,可以看到被导入处理的结果:

![](https://user-gold-cdn.xitu.io/2020/6/21/172d61cc37846a59?w=514&h=68&f=png&s=6441)
如果将其与 CSS 文件进行比较，你会发现每个格式现在都是导入对象中指向长字符串的键（key），例如 `title` 指向 `index-module--title--1i-Wh`。 这些样式名称是 CSS 模块生成的。 保证它们在你的网站上是唯一的。 而且由于必须导入它们才能使用这些类，所以对于在任何地方使用这些 CSS 样式都没问题。

### 其他处理 CSS 的方式

使用 css 样式就点到为止了，其他的比如 Sass 等；还有 CSS in JS 的方式，比如 Emotion, Styled Component。可以去官网查看相应支持的 Gatsby 插件，就可以使用了。

## 获取 Gatsby 中的数据

建立网站时，您可能需要重用一些常用的数据——比如公用的网站标题、作者信息等，我们不可能在每个页面都加，所以我们把标题存储在一个位置，然后从其他文件引用该位置就行了，更改的时候也只能更改这一处地方。

这些常用数据的存放位置就是 `gatsby-config.js` 文件中的 `siteMetadata` 对象。打开这个文件，写入代码：

```
module.exports = {
  /* Your site config here */
  siteMetadata: {
    title: `Title from siteMetadata`,
    author: 'jacky'
  },
  plugins: [],
}
```

重启服务

### 使用页面查询

编辑 `src/pages/about.js`

```javascript
import React from 'react'
import { graphql } from 'gatsby'

// GraphQL 获取的数据，会当做参数传递到页面组件中
// 数据的形式是 { errors, data }，没有错误则不会有 errors
const About = ({ data }) => {
  console.log(data.site.siteMetadata.title)
  return (
    <div>
      <h1>Title: {data.site.siteMetadata.title}</h1>
      <p>author: {data.site.siteMetadata.author}</p>
      <div>about me</div>
    </div>
  )
}

export default About

export const query = graphql`
  query {
    site {
      siteMetadata {
        title
        author
      }
    }
  }
`
```

然后就拉出数据了

![](https://user-gold-cdn.xitu.io/2020/6/21/172d63bec7bccdeb?w=576&h=188&f=png&s=10256)

其中，GraphQL 查询语句是：

```
{
  site {
    siteMetadata {
      title,
      author
    }
  }
}
```

你可能会懵逼， site 是哪里来的，这数据格式怎么是这样，这就是 GraphQL 查询语句了，我们可以打开 Gatsby 内置的 GraphQL 调试界面 `http://localhost:8000/__graphql`。

这里的`site`即为我们`gatsby-config.js`写的站点配置。

依次点击左边的`site`、`siteMetadata`，然后有我们刚才写的`title`,`author`字段 供我们选择，在点击的同时，中间的框会生成查询语句，最后点击运行按钮，会输出查询结果。

![](https://user-gold-cdn.xitu.io/2020/6/21/172d660d32ae1825?w=1442&h=642&f=png&s=61514)

当你后面需要获取更多复杂嵌套数据的时候，可以直接在这个界面找和点击，自动生成查询语句就行了。

由我们自己写的查询语句也知道，上面我们介绍 GraphQL 的时候说的一个特点，获取数据不多不少，即我们可以选择想要的数据，比如我就想获取标题不想获取作者

```
export const query = graphql`
  query {
    site {
      siteMetadata {
        title
      }
    }
  }
`
```

### 使用非页面组件查询（静态查询）

这里注意，我们需要区分两种方式的查询，上面的例子的查询方式，只适用于页面查询，即是在`src/pages`下的路由界面，如果在非页面组件下进行 GraphQL 查询，则不能用上面的方式，应该使用 `StaticQuery` 组件或者 `useStaticQuery hook`。

```
// 页面查询
export const query = graphql`
    query{ ... }
`
```

比如我们创建一个组件`src/components/header.js`

- StaticQuery

```javascript
import React from 'react'
import { graphql, StaticQuery } from 'gatsby'

const Header = () => (
  <StaticQuery
    query={graphql`
      query {
        site {
          siteMetadata {
            author
          }
        }
      }
    `}
    render={data => {
      const {
        site: { siteMetadata },
      } = data
      return <div>这是Header组件，作者是：{siteMetadata.author}</div>
    }}
  />
)

export default Header
```

或者使用 useStaticQuery 方式

- useStaticQuery

```
import React from "react"
import { useStaticQuery, graphql } from "gatsby"

const Header = () => {
  const data = useStaticQuery(
    graphql`
      query {
        site {
          siteMetadata {
            author
          }
        }
      }
    `
  )
  return <div>这是Header组件，作者是：{data.site.siteMetadata.author}</div>
}

export default Header
```

然后在`src/pages/index.js`中引入 Header 组件

```javascript
import React from 'react'
import Layout from '../components/layout'
import Header from '../components/header'
import styles from '../styles/index.module.css'

export default function Home() {
  return (
    <Layout>
      <Header />
      <h1 className={styles.title}>Hello World</h1>
      <div className={styles.text}>Test</div>
    </Layout>
  )
}
```

运行结果如下：

![](https://user-gold-cdn.xitu.io/2020/6/21/172d6d9b2afff80c?w=705&h=203&f=png&s=8464)

## 插件

### gatsby-source-filesystem

这个是 Gatsby 的数据源插件，即通过该插件将各方面的数据转换为本地能够通过 GraphQL 提取的内容，用于设置项目的文件系统。

- 安装插件

```
npm install --save gatsby-source-filesystem
```

- 在 gatsby-config.js 文件配置

```javascript
module.exports = {
  siteMetadata: {
    title: `Title from siteMetadata`,
    author: 'jacky',
  },
  plugins: [
    {
      resolve: `gatsby-source-filesystem`,
      options: {
        name: `src`, // 名称，可以用来过滤
        path: `${__dirname}/src/`, // 文件路径
      },
    },
  ],
}
```

重启服务

打开`http://localhost:8000/__graphql`，关注`allFile`和`file`两个可选项。

我们点击 `allFile`这个可选项，依次选择如下图

![](https://user-gold-cdn.xitu.io/2020/6/21/172d74ca7b60229d?w=1542&h=928&f=png&s=113095)

这里我们要知道，我们要查询的数据基本在`edges.node`下，节点 `node`是图（graph）中一个对象的特殊称呼，每个 File 节点都包含了你要查询的字段。

我们选择的`id`,`relativePath`,`name`字段都对应我们 `src`目录下创建的文件, 还有很多其他字段可选择，由此可知我们创建的 7 个文件的各种信息。

既然这里能查询出来，说明我们组件里面也能查出来然后使用数据，比如你可以从组件中查询出来做个文件列表。

这个相比传统的 React 项目就挺厉害了，不用做什么复杂的工作，就能轻松拿下这些数据。Gatsby 比较强大的就是它的生态了，很多插件都配好了，更多插件可以看官网的插件库介绍：https://www.gatsbyjs.org/plugins/

### gatsby-transformer-remark

这个是数据转换插件，我们用 Gatsby 做个人博客的话，它可必不可少。现在我们程序员写博客基本是用 markdown 语法了，做一个博客的话，不可缺少的就是对 markdown 语法的解析。

- 安装插件

```
npm install --save gatsby-transformer-remark
```

- 在 gatsby-config.js 文件配置

```javascript
module.exports = {
  /* Your site config here */
  siteMetadata: {
    title: `Title from siteMetadata`,
    author: 'jacky',
  },
  plugins: [
    {
      resolve: `gatsby-source-filesystem`,
      options: {
        name: `src`, // 名称，可以用来过滤
        path: `${__dirname}/src/`, // 文件路径
      },
    },
    `gatsby-transformer-remark`,
  ],
}
```

重启服务

这个插件添加了 allMarkdownRemark 和 markdownRemark 两个数据字段

![](https://user-gold-cdn.xitu.io/2020/6/21/172d7761605cba2b?w=205&h=243&f=png&s=7538)

- 新建 md 文件

在`src`下创建`_posts`目录，添加一个`first-blog.md`文件

```
---
title: "我的第一篇博客"
date: "2020-06-21"
tags: "React"
categories: "React系列"
---

## 这是第一篇博客的标题

first blog 的内容 1
```

如果博客不是自己搭建的，而是直接上某平台写的话，对上面这段语法就可能不太了解，在 md 文件开头的 `---` 包裹的是 frontmatter（前言），然后在里面可以添加文章的各种关键词信息。

然后我们就可以在`http://localhost:8000/__graphql`查询到我们写的 md 文件的详情信息了

![](https://user-gold-cdn.xitu.io/2020/6/21/172d7823d4e7db7f?w=1703&h=499&f=png&s=61510)

我们稍微再改动，创建多两个 md 文件

`second-blog.md`

```
---
title: "我的第二篇博客"
date: "2020-06-22"
tags: "Vue"
categories: "Vue系列"
---

## 这是第二篇博客的标题

second blog 的内容 2
```

`third-blog.md`

```
---
title: "我的第三篇博客"
date: "2020-06-23"
tags: "React"
categories: "React系列"
---

## 这是第三篇博客的标题

third blog 的内容 3
```

再次查询数据，把我们刚刚添加的文章数据都拿下来了

![](https://user-gold-cdn.xitu.io/2020/6/21/172d78dc41c94349?w=599&h=687&f=png&s=41548)

既然 GraphQL 能拿到数据，说明我们也能把它放到页面展示出来。

新建文件`src/pages/blog.js`,我们做一个博客目录汇总：

```javascript
import React from 'react'
import Layout from '../components/layout'
import { graphql } from 'gatsby'

const Blog = ({ data }) => {
  return (
    <Layout>
      <h1>博客目录</h1>
      <div>
        {data.allMarkdownRemark.edges.map(({ node }) => {
          return (
            <div
              key={node.id}
              style={{
                border: '1px solid #000',
                margin: '10px',
                padding: '10px',
              }}
            >
              <h2>{node.frontmatter.title}</h2>
              <div>分类{node.frontmatter.categories}</div>
              <div>标签：{node.frontmatter.tags}</div>
              <div>发布时间：{node.frontmatter.date}</div>
            </div>
          )
        })}
      </div>
    </Layout>
  )
}

export default Blog

export const query = graphql`
  query {
    allMarkdownRemark {
      edges {
        node {
          id
          frontmatter {
            tags
            title
            date
            categories
          }
        }
      }
    }
  }
`
```

打开`http://localhost:8000/blog`，博客目录就展示出来了：

![](https://user-gold-cdn.xitu.io/2020/6/23/172e1b12d01e3ba3?w=722&h=601&f=png&s=36586)

但有个问题，每篇博客的信息确实能拿出来，但是我们要链接啊，即是点击博客标题，进入博客详情页面，即是文章路径，所以接下来我们将创建页面。

## 利用数据创建页面

### gatsby-node.js

这个时候我们就要建立`gatsby-node.js`文件了，这个配置文件里面的代码是 node 层相关的。

前面我们说过 `src/pages`目录下的文件会全部渲染成路由，但如果我们博客文章一篇篇的详情页要我们自己创建一个 js 文件，这肯定不合理了。

我们会用到两个 API：

#### onCreateNode

每当创建新节点（或更新）时，Gatsby 都会调用 onCreateNode 函数。

向`gatsby-node.js`写入代码：

```javascript
exports.onCreateNode = ({ node }) => {
  console.log(node.internal.type)
}
```

重启服务，查看终端，你会发现打印出很多创建的节点：`SitePage`, `SitePlugin`, `Site`, `SiteBuildMetadata`, `Directory`, `File`, `MarkdownRemark`,我们关注的只有`MarkdownRemark`，于是修改函数使其仅仅记录 MarkdownRemark 节点；我们使用每一个 md 文件的名称来作为路径，如要获取文件名称，你需要遍历一遍它的父节点 `File`，`File`节点包含了我们需要的文件数据。

```javascript
exports.onCreateNode = ({ node, getNode }) => {
  if (node.internal.type === `MarkdownRemark`) {
    const fileNode = getNode(node.parent)
    console.log(`\n`, fileNode.relativePath)
  }
}
```

重启服务，查看终端，打印如下：

```
 _posts/first-blog.md

 _posts/second-blog.md

 _posts/third-blog.md
```

相对路径出来，接下来我们要创建路径，可以使用`gatsby-source-filesystem`插件带的创建路径的函数

```javascript
const { createFilePath } = require(`gatsby-source-filesystem`)

exports.onCreateNode = ({ node, getNode }) => {
  if (node.internal.type === `MarkdownRemark`) {
    console.log(createFilePath({ node, getNode, basePath: `pages` }))
  }
}
```

重启服务，打印结果如下：

```
/_posts/first-blog/
/_posts/second-blog/
/_posts/third-blog/
```

接下来，向 API 传递一个函数`createNodeField`，该函数允许我们在其他插件创建的节点里创建其他字段。

```javascript
exports.onCreateNode = ({ node, getNode, actions }) => {
  const { createNodeField } = actions
  if (node.internal.type === `MarkdownRemark`) {
    const slug = createFilePath({ node, getNode, basePath: `pages` })
    createNodeField({
      node,
      name: `slug`,
      value: slug, // 通常用 slug 一词代表路径
    })
  }
}
```

重启服务，打开`http://localhost:8000/__graphql`，就能查询到路径了

![](https://user-gold-cdn.xitu.io/2020/6/23/172e1ce4c0a61cae?w=1097&h=448&f=png&s=28484)

#### createPages

createPages 这个 API 用于添加页面。在创建页面之前，首先要有页面模板，这样创建页面就能指定所使用的模板了。

新建文件`src/templates/blog-post.js`

```javascript
import React from 'react'
import Layout from '../components/layout'

export default () => {
  return (
    <Layout>
      <div>Hello blog post</div>
    </Layout>
  )
}
```

然后更改 `gatsby-node.js`的`createPages`函数：

```javascript
const path = require(`path`)

exports.createPages = async ({ graphql, actions }) => {
  const result = await graphql(`
    query {
      allMarkdownRemark {
        edges {
          node {
            fields {
              slug
            }
          }
        }
      }
    }
  `)
  result.data.allMarkdownRemark.edges.forEach(({ node }) => {
    createPage({
      path: node.fields.slug,
      component: path.resolve(`./src/templates/blog-post.js`),
      context: {
        slug: node.fields.slug,
      },
    })
  })
}
```

重启服务，随便输入一个路径，跳转到默认的 404 页面，就会看到自动生成三篇博客的路径了，点击任一篇，跳转的是我们刚创建的 `blog-post.js`组件。

![](https://user-gold-cdn.xitu.io/2020/6/24/172e3bb13a244d4c?w=1230&h=548&f=png&s=51476)

我们要的是显示博客内容，所以我们需要对模板文件再进行改造：

```javascript
import React from 'react'
import { graphql } from 'gatsby'
import Layout from '../components/layout'

export default ({ data }) => {
  const post = data.markdownRemark
  return (
    <Layout>
      <div>
        <h1>{post.frontmatter.title}</h1>
        <div dangerouslySetInnerHTML={{ __html: post.html }} />
      </div>
    </Layout>
  )
}

export const query = graphql`
  query($slug: String!) {
    markdownRemark(fields: { slug: { eq: $slug } }) {
      html
      frontmatter {
        title
      }
    }
  }
`
```

上面是动态变量查询的写法，GraphQL 的语法，可以看看 (https://graphql.org/learn/queries/#variables)

完成之后，比如我点击第二篇博客，就可以正确进入页面并显示内容了

![](https://user-gold-cdn.xitu.io/2020/6/24/172e3c8f759609dd?w=757&h=255&f=png&s=18207)

为了更完善，我们给 blog 目录页面添加可跳转的链接，加入 `slug`查询字段, 增加 `Link`跳转，这个用法与 React 路由的 `Link`差不多一致，涉及不深的情况下暂且当成相同用法。

`src/pages/blog.js`

```javascript
import React from 'react'
import Layout from '../components/layout'
import { graphql, Link } from 'gatsby'

const Blog = ({ data }) => {
  return (
    <Layout>
      <h1>博客目录</h1>
      <div>
        {data.allMarkdownRemark.edges.map(({ node }) => {
          return (
            <Link to={node.fields.slug} key={node.id}>
              <div
                style={{
                  border: '1px solid #000',
                  margin: '10px',
                  padding: '10px',
                }}
              >
                <h2>{node.frontmatter.title}</h2>
                <div>分类{node.frontmatter.categories}</div>
                <div>标签：{node.frontmatter.tags}</div>
                <div>发布时间：{node.frontmatter.date}</div>
              </div>
            </Link>
          )
        })}
      </div>
    </Layout>
  )
}

export default Blog

export const query = graphql`
  query {
    allMarkdownRemark {
      edges {
        node {
          id
          frontmatter {
            tags
            title
            date
            categories
          }
          fields {
            slug
          }
        }
      }
    }
  }
`
```

![](https://user-gold-cdn.xitu.io/2020/6/24/172e3ce7be50852c?w=722&h=596&f=png&s=36793)

现在点击博客，就能跳转到对应页面了，虽然没做样式页面比较丑，但只要在`src/_posts`目录新建 md 文件，然后这里就能跟着自动刷新，算是一个超简易的博客了。

## 打包上线

停止开发服务，构建生产版本，把静态文件输出到 public 目录中

```
gatsby build
```

在本地查看生产环境版本，运行：

```
gatsby serve
```

接下来，就可以在`localhost:9000`访问我们刚才做的网站了

本项目的 [github 地址](https://github.com/Jacky-Summer/gatsby-blog-demo)

至此，我们教程就介绍到这里了。

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~

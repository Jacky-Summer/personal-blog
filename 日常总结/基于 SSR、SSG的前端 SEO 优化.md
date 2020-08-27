# 基于 SSR/SSG + Gatsby 的前端 SEO 优化

## 前言

前段时间对项目做了 SEO 优化，到现在才来写总结。我们知道，常规用 Vue/React 开发的是 SPA 应用，但是天然的单页面应用 SEO 就是不好，虽然说现在也有各种技术可以改善了，比如使用预渲染，但也都存在各种缺点。但是即使这样，也抵不住 Vue/React 这类框架的潮流，很多产品也可以通过其他亮点而不依赖 SEO 普及开，也有需要登录才能用的使用 SEO 也没有什么意义。

如果项目中真的**对 SEO 和首屏加载速度有刚性需求**，又使用 Vue/React 这类技术，且想尽量减少代码开发附加的难度，有一种比较直接的方式，就是直接使用服务端渲染的框架，Vue 的 Nuxt.js，React 的 Next.js/Gatsby。

不过，其实学习一门新框架也是一项附加成本啊哈哈，但是 SSR 渲染不过实际开发用不用，起码都要了解一下。我当前以 React 技术栈为主，所以目前只了解的是关于 React 的 SSR 渲染框架，有兴趣的可以看下我这两篇文章：

- [手把手带你入门 NextJs（v9.5）](https://juejin.im/post/6863336367309455373)
- [手把手带你入门 Gatsby](https://juejin.im/post/6850418110885789704)

所以本文不讨论单页面应用的 SEO 优化，讲的是基于服务端渲染（SSR）/静态生成（SSG）网站 SEO 的优化。

本文会回顾传统的 SEO 的优化方式，以及基于 Gatsby SEO 的优化。

## 服务端渲染 SSR 与静态网站渲染 SSG

服务端是指客户端向服务器发出请求，然后运行时动态生成 html 内容并返回给客户端。
静态站点的解析是在构建时执行的，当发出请求时，html 将静态存储，直接发送回客户端。

通常来说，静态站点在运行时会更快，因为不需要服务端做处理，但缺点是对数据的任何更改都需要在服务端进行完全重建；而服务端渲染则会动态处理数据，不需要进行完全重建。

对于 Vue/React 来说，对于它们的 SSR/SSG 框架出现的原因就是主要就是 SEO 和首屏加载速度。

## 搜索引擎的工作原理

> 在搜索引擎网站的后台会有一个非常庞大的数据库，里面存储了海量的关键词，而每个关键词又对应着很多网址，这些网址是被称之为“搜索引擎蜘蛛”或“网络爬虫”程序从互联网上收集而来的。

这些"蜘蛛"在互联网上爬行，从一个链接到另一个链接，对内容进行分析，提炼关键词加入数据库中；如果蜘蛛认为是垃圾或重复信息，就舍弃继续爬行。当用户搜索时，就能检索出与关键字相关的网址显示给用户。

当用户在搜索引擎搜索时，比如搜索"前端"，则跳出来所有含有"前端"二字关键字的网页，然后根据特定算法给每个含有"前端"二字的网页一个评分排名返回搜索结果。而这些包含"前端"的内容，可以是文章标题、描述、关键字、内容甚至可以是链接。当然，也有可能是广告优先置顶，你懂的。

> 一个关键词对用多个网址，因此就出现了排序的问题，相应的当与关键词最吻合的网址就会排在前面了。在“蜘蛛”抓取网页内容，提炼关键词的这个过程中，就存在一个问题：“蜘蛛”能否看懂。如果网站内容是 flash 和 js 等，那么它是看不懂的，会犯迷糊，即使关键字再贴切也没用。相应的，如果网站内容可以被搜索引擎能识别，那么搜索引擎就会提高该网站的权重，增加对该网站的友好度。这样一个过程我们称之为 SEO(Search Engine Optimization)，即搜索引擎优化。

## SEO 目的

让网站更利于各大搜索引擎抓取和收录，增加对搜索引擎的友好度，使得用户在搜索对应关键词时网站时能排在前面，增加产品的曝光率和流量。

## SEO 优化方式

我们这里主要讲前端能参与和做的优化方式。比如很多 SEO 优化方式都有介绍：控制首页链接数量，扁平化目录层次，优化网站结构布局，分页导航写法这些等，但实际上，日常前端开发也充当不了网站整体设计的角色，只能是协调，这些大部分都是一开始就定好的东西。

比如新闻媒体类等网站比较重视 SEO 的，通常公司还会设有 SEO 部门或者是 SEO 优化工程师岗位，像上面说的，还有网页关键词、描述的就交给他们参与和提供，有些优化方式我们难以触及的就不细谈了，有兴趣的可以去了解。

### 网页 TDK 标签

- title：当前页面的标题（强调重点即可，每个页面的 title 尽量不要相同）
- description：当前页面的描述（列举几个关键词即可，不要过分堆积）
- keywords：当前页面的关键词（高度概括网页内容）

每个页面的 TDK 都不一样，这个需要根据产品业务提炼出核心关键词。

那么页面的 TDK 都不一样，我们就需要对它进行动态设置,react 的话有[react-helmet](https://github.com/nfl/react-helmet)
插件，用于设置头部标签。

```javascript
import React from 'react'
import { Helmet } from 'react-helmet'

const GoodsDetail = ({ title, description, keywords }) => {
  return (
    <div className='application'>
      <Helmet>
        <title>{title}</title>
        <meta name='description' content={`${description}`} />
        <meta name='keywords' content={`${keywords}`} />
      </Helmet>
      <div>content...</div>
    </div>
  )
}
```

上面是演示，实际项目做法还是会把 Helmet 里的内容单独抽离出来做组件。

在 Next.js 里面，是自带 Head 组件的：`import Head from 'next/head'`

### 语义化标签

根据内容的结构化，选择合适的 HTML5 标签尽量让代码语义化，如使用 header，footer，section，aside，article，nav 等等语义化标签可以让爬虫更好的解析。

### 合理使用 h1~h6 标签

一个页面中只能最多出现一次`h1`标签，`h2`标签通常作为二级标题或文章的小标题。其余`h3`-`h6`标签如要使用应按顺序层层嵌套下去，不可以断层或反序。

比如通常在首页的 logo 上加`h1`标签，但网站设计只展示 logo 图无文字的情况下，h1 的文字就可以设置`font-size`为零来隐藏

```html
<h1>
  <img src="logo.png" alt="jacky" />
  <span>jacky的个人博客</span>
</h1>
```

### 图片的 alt 属性

一般来说，除非是图片仅仅是纯展示类没有任何实际信息的话，`alt`属性可以为空。否则使用`img`标签都要添加`alt`属性，使"蜘蛛"可以抓取到图片的信息。
当网络加载不出来或者图片地址失效时，`alt`属性的内容才会代替图片呈现出来，

```html
<img src="dog.jpg" width="300" height="200" alt="哈士奇" />
```

### a 标签的 title

同理，a 标签的 title 属性其实就是提示文字作用，当鼠标移动到该超链接上时，就会有提示文字的出现。通过添加该属性也有微小的作用利于 SEO。

```html
<a
  href="https://github.com/Jacky-Summer/personal-blog"
  title="了解更多关于Jacky的个人博客"
  >了解更多</a
>
```

### 404 页面

404 页面首先是用户体验良好，不会莫名报一些其他提示。其次对蜘蛛也友好，不会因为页面错误而停止抓取，可以返回抓取网站其他页面。

### nofollow 忽略跟踪

- nofollow 有两种用法：

1. 用于 meta 元标签，告诉爬虫该页面上所有链接都无需追踪。

```
<meta name="robots" content="nofollow" />
```

2. 用于 a 标签，告诉爬虫该页面无需追踪。

```
<a href="https://www.xxxx?login" rel="nofollow">登录/注册</a>
```

通常用在 a 标签比较多，它主要有三个作用：

1. "蜘蛛"分配到每个页面的权重是一定的，为了集中网页权重并将权重分给其他必要的链接，就设置`rel='nofollow'`告诉"蜘蛛"不要爬，来避免爬虫抓取一些无意义的页面，影响爬虫抓取的效率；而且一旦"蜘蛛"爬了外部链接，就不会再回来了。
2. 付费链接：为了防止付费链接影响 Google 的搜索结果排名，Google 建议使用 nofollow 属性。
3. 防止不可信的内容，最常见的是博客上的垃圾留言与评论中为了获取外链的垃圾链接，为了防止页面指向一些拉圾页面和站点。

### 建立 robots.txt 文件

> robots.txt 文件由一条或多条规则组成。每条规则可禁止（或允许）特定抓取工具抓取相应网站中的指定文件路径。

```
User-agent: *
Disallow:/admin/
SiteMap: http://www.xxxx.com/sitemap.xml
```

关键词：

1. User-agent 表示网页抓取工具的名称
2. Disallow 表示不应抓取的目录或网页
3. Allow 应抓取的目录或网页
4. Sitemap 网站的站点地图的位置

- `User-agent: *`表示对所有的搜索引擎有效
- `User-agent: Baiduspider` 表示百度搜索引擎，还有谷歌 Googlebot 等等搜索引擎名称，通过这些可以设置不同搜索引擎访问的内容

参考例子的话比如百度的 [robots.txt](https://www.baidu.com/robots.txt)，京东的 [robots.txt](https://www.jd.com/robots.txt)

robots 文件是搜索引擎访问网站时第一个访问的，然后根据文件里面设置的规则，进行网站内容的爬取。通过设置`Allow`和`Disallow`访问目录和文件，引导爬虫抓取网站的信息。

它主要用于使你的网站避免收到过多请求，告诉搜索引擎应该与不应抓取哪些页面。如果你不希望网站的某些页面被抓取，这些页面可能对用户无用，就通过`Disallow`设置。实现定向 SEO 优化，曝光有用的链接给爬虫，将敏感无用的文件保护起来。

即使网站上面所有内容都希望被搜索引擎抓取到，也要设置一个空的 robot 文件。因为当蜘蛛抓取网站内容时，第一个抓取的文件 robot 文件，如果该文件不存在，那么蜘蛛访问时，服务器上就会有一条 404 的错误日志，多个搜索引擎抓取页面信息时，就会产生多个的 404 错误，故一般都要创建一个 robots.txt 文件到网站根目录下。

空 robots.txt 文件

```
User-agent: *
Disallow:
```

如果想要更详细的了解 robots.txt 文件，可以看下：

- [关于 robots.txt](https://support.google.com/webmasters/answer/6062608?hl=en&ref_topic=6061961&visit_id=637337520313091593-2508183586&rd=1)
- [关于 Robots.txt 和 SEO: 你所需要知道的一切](https://ahrefs.com/blog/zh/robots-txt/)

一般涉及目录比较多的话都会找网站工具动态生成 robots.txt，比如 [生成 robots.txt](https://tool.chinaz.com/robots/)

### 建立网站地图 sitemap

当网站刚刚上线的时候，连往该网站的外部链接并不多，爬虫可能找不到这些网页；或者该网站的网页之间没有较好的衔接关系，爬虫容易漏掉部分网页。这个时候，sitemap 就派上用场了。

sitemap 是一个将网站栏目和连接归类的一个文件，让搜索引擎全面收录站点网页地址，了解站点网页地址的权重分布以及站点内容更新情况，提高爬虫的爬取效率。Sitemap 文件包含的网址不可以超过 5 万个，且文件大小不得超过 10MB。

sitemap 地图文件包含 html(针对用户)和 xml(针对搜索引擎)两种，最常见的就是 xml 文件，XML 格式的 Sitemap 一共用到 6 个标签，其中关键标签包括链接地址(loc)、更新时间(lastmod)、更新频率(changefreq)和索引优先权(priority)。

爬虫怎么知道网站有没有提供 sitemap 文件呢，也就是上面说的路径放在了 robots.txt 里面。

先找网站的根目录里找 robots.txt，比如腾讯网下的 robots.txt 如下：

```
User-agent: *
Disallow:
Sitemap: http://www.qq.com/sitemap_index.xml
```

就找到了 sitemap 路径（只列出一部分）

```
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap>
    <loc>http://news.qq.com/news_sitemap.xml.gz</loc>
    <lastmod>2011-11-15</lastmod>
  </sitemap>
  <sitemap>
    <loc>http://finance.qq.com/news_sitemap.xml.gz</loc>
    <lastmod>2011-11-15</lastmod>
  </sitemap>
  <sitemap>
    <loc>http://sports.qq.com/news_sitemap.xml.gz</loc>
    <lastmod>2011-11-15</lastmod>
  </sitemap>
  <sitemap>
</sitemapindex>
```

- loc：页面永久链接地址，可以是静态页面，也可是动态页面
- lastmod：页面的最后修改时间，非必填项。搜索引擎根据此项与 changefreq 相结合，判断是否要重新抓取 loc 指向的内容

一般网站开发完后，这个 sitemap 一般都是靠自动生成，比如 [sitemap 生成工具](https://www.xml-sitemaps.com/)

### 结构化数据

结构化数据(Structured data)是一种标准化格式，使用它向 Google 提供有关该网页含义的明确线索，从而帮助理解该网页。一般都 JSON-LD 格式，这格式长什么样呢，看谷歌官方的示例代码：

```html
<html>
  <head>
    <title>Party Coffee Cake</title>
    <script type="application/ld+json">
      {
        "@context": "https://schema.org/",
        "@type": "Recipe",
        "name": "Party Coffee Cake",
        "author": {
          "@type": "Person",
          "name": "Mary Stone"
        },
        "nutrition": {
          "@type": "NutritionInformation",
          "calories": "512 calories"
        },
        "datePublished": "2018-03-10",
        "description": "This coffee cake is awesome and perfect for parties.",
        "prepTime": "PT20M"
      }
    </script>
  </head>
  <body>
    <h2>Party coffee cake recipe</h2>
    <p>This coffee cake is awesome and perfect for parties.</p>
  </body>
</html>
```

列明了网页页面种类属于"食谱"，作者和发布时间，描述和烹饪时间等等。这样谷歌搜索出来就有机会含有这些提示或者你带着关键信息去搜索更有利于找到结果。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ee7e3ae735049b1ba53e28600ed31fa~tplv-k3u1fbpfcp-zoom-1.image)

官方提供了各种字段用来描述"食谱"，你只要去查阅相关字段，就可以直接按格式来使用了。

因为该 SEO 优化针对谷歌搜索引擎特有的，所以有设置该方式的网站通常是用户是不限于国内的，不仅是结构化数据特有，还有一种 SEO 优化方式是 **AMP 网页**，感兴趣的可以了解看看 —— [AMP](https://developers.google.com/search/docs/guides/about-amp)

谷歌还提供了测试工具 [Structured Data Testing Tool](https://search.google.com/structured-data/testing-tool)，可以输入测试网站网址来查看该网站有没有结构化数据设置。

### 性能优化

比如减少 http 请求，控制页面大小，懒加载，利用缓存等等，这方式就很多了，都是为了提高网站的加载速度和良好用户体验，这个也不是专指 SEO 的问题，是开发中都要做的事情。

因为当网站速度很慢时，一旦超时，"蜘蛛"也会离开。

## Gatsby 下的 SEO 优化

本身 Gatsby 就采取静态生成的方式，SEO 已是可以，但依然还是要做 SEO 优化。

知道了上述的 SEO 优化方式后，Gatsby 该如何实战优化呢？这个，由于 Gatsby 社区比较强大，插件很多，所以上面几个依靠插件就可以快速配置生成。

### gatsby-plugin-robots-txt

在 gatsby-config.js 里面配置

```
module.exports = {
  siteMetadata: {
    siteUrl: 'https://www.xxxxx.com'
  },
  plugins: ['gatsby-plugin-robots-txt']
};
```

### gatsby-plugin-sitemap

在 gatsby-config.js 里面配置

```
{
  resolve: `gatsby-plugin-sitemap`,
  options: {
    sitemapSize: 5000,
  },
},
```

### 网页 TDK

Gatsby 标准脚手架和有官方文档都有一个 SEO.js 文件，里面就是给我们设置 TDK 提供了方法

```javascript
import React from 'react'
import PropTypes from 'prop-types'
import { Helmet } from 'react-helmet'
import { useStaticQuery, graphql } from 'gatsby'

function SEO({ description, lang, meta, title }) {
  const { site } = useStaticQuery(
    graphql`
      query {
        site {
          siteMetadata {
            title
            description
            author
          }
        }
      }
    `
  )

  const metaDescription = description || site.siteMetadata.description

  return (
    <Helmet
      htmlAttributes={{
        lang,
      }}
      title={title}
      meta={[
        {
          name: `description`,
          content: metaDescription,
        },
        {
          property: `og:title`,
          content: title,
        },
        {
          property: `og:description`,
          content: metaDescription,
        },
        {
          property: `og:type`,
          content: `website`,
        },
        {
          name: `twitter:card`,
          content: `summary`,
        },
        {
          name: `twitter:creator`,
          content: site.siteMetadata.author,
        },
        {
          name: `twitter:title`,
          content: title,
        },
        {
          name: `twitter:description`,
          content: metaDescription,
        },
      ].concat(meta)}
    />
  )
}

SEO.defaultProps = {
  lang: `en`,
  meta: [],
  description: ``,
}

SEO.propTypes = {
  description: PropTypes.string,
  lang: PropTypes.string,
  meta: PropTypes.arrayOf(PropTypes.object),
  title: PropTypes.string.isRequired,
}

export default SEO
```

然后在页面模板文件中引入 SEO.js，并传入页面的变量参数，即可设置 TDK 等等头部信息。

### structured data

比如说项目是新闻文章展示的，可以设置三种结构化数据（数据类型和字段不是凭空捏造的，这个需要去 Google 查符合对应匹配的）——文章详情页，文章列表页，再加个公司的介绍。

在项目根目录下新建

- `./src/components/Jsonld.js`
  封装外部包住的 script 标签作为组件

```
import React from 'react'
import { Helmet } from 'react-helmet'

function JsonLd({ children }) {
  return (
    <Helmet>
      <script type="application/ld+json">{JSON.stringify(children)}</script>
    </Helmet>
  )
}

export default JsonLd
```

- `./src/utils/json-ld/article.js` - 文章详情结构化数据描述

```javascript
const articleSchema = ({
  url,
  headline,
  image,
  datePublished,
  dateModified,
  author,
  publisher,
}) => ({
  '@context': 'http://schema.org',
  '@type': 'Article',
  mainEntityOfPage: {
    '@type': 'WebPage',
    '@id': url,
  },
  headline,
  image,
  datePublished,
  dateModified,
  author: {
    '@type': 'Person',
    name: author,
  },
  publisher: {
    '@type': 'Organization',
    name: publisher.name,
    logo: {
      '@type': 'ImageObject',
      url: publisher.logo,
    },
  },
})

export default articleSchema
```

- `./src/utils/json-ld/item-list.js` - 文章列表结构化数据描述

```javascript
const itemListSchema = ({ itemListElement }) => ({
  '@context': 'http://schema.org',
  '@type': 'ItemList',
  itemListElement: itemListElement.map((item, index) => ({
    '@type': 'ListItem',
    position: index + 1,
    ...item,
  })),
})

export default itemListSchema
```

- `./src/utils/json-ld/organization.js` - 公司组织结构化数据描述

```javascript
const organizationSchema = ({ name, url }) => ({
  '@context': 'http://schema.org',
  '@type': 'Organization',
  name,
  url,
})

export default organizationSchema
```

然后再分别引入页面，比如我们在文章详情页面，引入对应类型文件，大概就是这么个用法：

```javascript
// ...
import JsonLd from '@components/JsonLd'
import SEO from '@components/SEO'
import articleSchema from '@utils/json-ld/article'

const DetailPage = ({ data }) => {
  // 处理 data，拆开相关字段
  return (
    <Layout>
      <SEO
        title={meta_title || title}
        description={meta_description}
        keywords={meta_keywords}
      />
      <JsonLd>
        {articleSchema({
          url,
          headline: title,
          datePublished: first_publication_date,
          dateModified: last_publication_date,
          author: siteMetadata.title,
          publisher: {
            name: siteMetadata.title,
            logo: 'xxx',
          },
        })}
      </JsonLd>
      <Container>
        <div>content...</div>
      </Container>
    </Layout>
  )
}
```

上面的代码要是迷迷糊糊倒是正常，因为没有了解过结构化数据的内容，但看文档就大概可以了解清楚了。

### Lighthouse 性能优化工具

可以去谷歌商店安装`LightHouse`，打开 F12，进入你的网站，点击`Generate report`，就会生成网站对应的报告

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f758e6c497674256a516d3a357774826~tplv-k3u1fbpfcp-zoom-1.image)

- 生成的 report：
  ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d84656f6cce4800ab793feb73141e30~tplv-k3u1fbpfcp-zoom-1.image)

在它下面有一些提示，针对性能和 SEO 等，你可以根据提示去改善你的代码。

文章的介绍到这里就结束了，希望对大家了解 SEO 有一点帮助。SEO 的摸索并不是以上举例完就差不多没了，其实有各种各样的方式可以优化。上面列举的是比较常见的，事实上，我觉得 SEO 优化无非是想吸引更多的用户点击和使用网站，但如果网站的内容优质用户体验良好，加性能好，那么有用户使用后就自带推广性，那么无疑比 SEO 简单的优化强多了。

## 参考文章

- [前端 SEO 优化](https://juejin.im/post/6844903824428105735)

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~

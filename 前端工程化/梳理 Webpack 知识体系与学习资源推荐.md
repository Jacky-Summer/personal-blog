# 梳理 Webpack 知识体系与学习资源推荐

## 前言

关于 Webpack 之前已经有写过几篇零散的文章:

- [由零开始使用 Webpack 来搭建 React 项目](https://juejin.cn/post/6868917159901593614)
- [Webpack Sourcemap 回顾](https://juejin.cn/post/6941924227444375589)
- [升级 Webpack5 实践](https://juejin.cn/post/7021540509277487111)
- [如何编写一个 Webpack Loader](https://juejin.cn/post/6882895689773383694)
- [如何编写一个 Webpack Plugin](https://juejin.cn/post/6884866016565084173)

而本文的内容则不会不再赘述各种 Webpack 细节，而是梳理一遍 Webpack 的相关知识（内容可能不算全，但也可当回顾 Webpack 了），并推荐看过的一些大佬的文章供学习阅读参考，希望有助于大家深入 Webpack 的学习。

本文从 Webpack 基础、优化、原理三个角度出发来进行梳理 Webpack 知识体系，以下是我整理的一份 Webpack 知识体系思维导图：

![Webpack 知识体系.jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/10df44fb991142128d097df4f253df93~tplv-k3u1fbpfcp-watermark.image?)

## Webpack 核心基础

Webpack 基础的就不说了

### 项目配置

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/16155698a12245a480d51be3e05efe3e~tplv-k3u1fbpfcp-watermark.image?)

- [由零开始使用 Webpack 来搭建 React 项目](https://juejin.cn/post/6868917159901593614)
- [带你深度解锁 Webpack 系列(基础篇)](https://juejin.cn/post/6844904079219490830)

### Loader

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1ad76fabe144ba59d75a19599ed5183~tplv-k3u1fbpfcp-watermark.image?)

**什么是 Loader**

Loader 通常指**打包的方案**，作用即是**让 webpack 拥有了加载和解析⾮ JavaScript ⽂件的能⼒**

- [Loader 知识分享](https://juejin.cn/post/6950092728919130126)：关于 Loader 比较全的介绍

**常用的 Loader**

- babel-loader： 将 ES6+ 语法转换为 ES5 语法
- file-loader：把⽂件输出到⼀个指定目录中，在代码中通过相对 URL 去引⽤输出的⽂件。
- url-loader：可以在小于限制文件大小的情况下以 base64 的⽅式把⽂件内容注⼊到代码中去
- css-loader：加载 CSS，⽀持模块化、压缩、⽂件导⼊等特性
- style-loader：将处理好的 css 通过 style 标签的形式添加到页面上
- less-loader：解析 Less 文件
- sass-loader：将 SCSS/SASS 代码转换成 CSS
- postcss-loader：扩展 CSS 语法，如可以配合 autoprefixer 插件自动补齐 CSS3 前缀

**如何编写 Loader**

Loader 其实就是一个函数，通过获得处理前的源内容，对原内容执行处理后，返回处理后的内容。

- [如何编写一个 Webpack Loader](https://juejin.cn/post/6882895689773383694)：可以先看下我这篇，比较简单的入门
- [Webpack 原理系列七：如何开发 Loader](https://juejin.cn/post/6966785086473633806)：较为详细的编写

**关于 pitch loader**

- [你不知道的「pitch loader」应用场景](https://juejin.cn/post/7037696103973650463)

### Plugin

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29b86280ee9e4c46b1c0d90c9f250709~tplv-k3u1fbpfcp-watermark.image?)

**什么是 Plugin**

Plugin 是一个功能扩展器，webpack 的插件体系是一种基于 Tapable 实现的强耦合架构，可以在 webpack 运行到某个时刻的时候，帮我们做一些事情

**常用的 Plugin**

- html-webpack-plugin： 依据一个 HTML 模板，生成 HTML 文件，并把打包生成的 js 自动引入到这个 HTML 文件中
- terser-webpack-plugin: 支持压缩 ES6，同时可以开启  parallel  参数，使用多进程压缩，加快压缩
- mini-css-extract-plugin: CSS 提取到单独的⽂件中, ⽀持按需加载
- webpack-bundle-analyzer: 可视化 Webpack 输出文件的体积 (业务组件、依赖第三方模块)
- speed-measure-webpack-plugin: 可以看到整个打包耗时、每个 Plugin 和 Loader 耗时

**如何编写 Plugin**

- [如何编写一个 Webpack Plugin](https://juejin.cn/post/6884866016565084173)
- [Webpack 原理系列二：插件架构原理与应用](https://juejin.cn/post/6955421936373465118)：介绍了 Plugin 的核心库——Tapable

### SourceMap

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6242f69a2e234fc88b11d3fb8a1d17c2~tplv-k3u1fbpfcp-watermark.image?)

这个只是因为之前做过总结，所以也被我单独拎出来做一大部分了。

Sourcemap 是为了解决开发代码与实际运行代码不一致时造成无法 debug 的问题，它是**源代码和目标代码出错位置的映射**，Webpack4 的 Webpack5 的 devtool 配置项的关键词顺序有点区别，注意下这个。

- [Webpack Sourcemap 回顾](https://juejin.cn/post/6941924227444375589)
- [深入浅出之 Source Map](https://juejin.cn/post/7023537118454480904)

## Webpack 优化

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9394272335f8489f91f5dc2a4f1af22a~tplv-k3u1fbpfcp-watermark.image?)

这个实用性比较高，可以对项目进行各种优化。上面思维导图就列得不全，这个具体可以去看各种优化文章吧。

需要注意辨别的是，有些优化方式在升级 Webpack4 或 Webpack5 的时候就过时了；其次，本身升级 Webpack5 就是个大优化，胜过你在低版本上各种优化搞法，愿没有 Webpack 配置工程师。

- [升级 Webpack5 实践](https://juejin.cn/post/7021540509277487111)
- [带你深度解锁 Webpack 系列(优化篇)](https://juejin.cn/post/6844904093463347208)
- [Webpack 配置全解析（优化篇）](https://juejin.cn/post/6858905382861946894)

## Webpack 原理

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d943f7b65dee4d269bd1cdd52655c900~tplv-k3u1fbpfcp-watermark.image?)

### Webpack 打包构建流程

- [[万字总结] 一文吃透 Webpack 核心原理](https://juejin.cn/post/6949040393165996040)：Webpack 大佬的经典原理文章
- [「搞点硬货」从源码窥探 Webpack4.x 原理](https://juejin.cn/post/6844904046294204429)
- [Webapck5 核心打包原理全流程解析](https://juejin.cn/post/7031546400034947108)

### Webpack 懒加载

webpack 的懒加载实际借助了 ES6 的 import()语法和 jsonp。可以看下：

- [一文搞懂 webpack 懒加载机制 —— webpack 系列](https://juejin.cn/post/6924484965073862664)
- [webpack 的异步加载原理及分包策略](https://segmentfault.com/a/1190000038180453)
- [懒加载构建原理详解（模块如何被组建&如何加载）&源码解读](https://www.cnblogs.com/zhaoweikai/p/10945780.html)

### 热更新原理

热更新即是当对代码进行修改并保存后，webpack 将对代码重新打包，在不刷新浏览器的前提下就能够对页面进行更新。这部分原理下面三篇读完也差不多了：

- [Webpack HMR 原理解析](https://zhuanlan.zhihu.com/p/30669007)
- [搞懂 webpack 热更新原理](https://github.com/careteenL/webpack-hmr)
- [轻松理解 webpack 热更新原理](https://juejin.cn/post/6844904008432222215)

### Tree Shaking

Tree Shaking 本质上为了消除无用的 JS 代码，减少加载文件体积。

实现前提：

- 代码遵循**ES6 Module**规范
- 将`mode` 选项为`production`
  - optimization.usedExports: true
  - optimization.minimize: true

推荐文章：

- [Webpack | TreeShaking 工作原理](https://zhuanlan.zhihu.com/p/472733451)
- [Webpack 原理系列九：Tree-Shaking 实现原理](https://juejin.cn/post/7019104818568364069)

## Webpack5

Webpack5 的主要变更：

- 用持久性缓存来提高构建性能。
- 用更好的算法和默认值来改进长期缓存。
- 更好的 Tree Shaking 和代码生成来改善包大小。
- 改善与网络平台的兼容性。
- 在不引入任何破坏性变化的情况下，清理那些在实现 v4 功能时处于奇怪状态的内部结构。
- 试图通过现在引入突破性的变化来为未来的功能做准备，尽可能长时间地保持在 v5 版本上。

Webpack5 还有个重大的特性就是模块联邦，它主要是为了更好的共享代码，让代码跨应用间利用 CDN 直接共享，不再需要本地安装 NPM 包、构建再发布，可以看下这篇： [模块联邦浅析](https://juejin.cn/post/7101457212085633054)

推荐文章：

- [阔别两年，webpack 5 正式发布了！](https://juejin.cn/post/6882663278712094727)
- [webpack5 上手指南](https://juejin.cn/post/6955266854839386119)

## Webpack 系统教程推荐

- [Hello Webpack](https://webpack-doc-20200329.vercel.app/)
- [史上最走心 webpack4.0 中级教程——配置之外你应该知道的事  ](https://www.cnblogs.com/dashnowords/p/9572755.html)
- [精通 Webpack 核心原理](https://juejin.cn/column/6978684601921175583)

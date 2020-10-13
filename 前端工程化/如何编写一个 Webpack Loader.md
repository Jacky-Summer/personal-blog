# 如何编写一个 Webpack Loader

## 前言

在平时自己由零搭建项目时，虽然基础配置都比较熟悉，比如配置 file-loader, url-loader, css-loader 等，配置不难，但究竟是怎么起作用的呢，今天就来说说如何编写一个 Webpack Loader。

## Loader 作用

按我自己的简单理解，loader 通常指打包的方案，即按什么方式来处理打包，打包的时候它可以拿到模块源代码，经过特定 loader 的转换后返回新的结果。

比如 sass-loader 可以把 SCSS 代码转换成 CSS 代码

## 编写 Loader

### 保持功能单一

我们项目中可能会配置很多，但要记住，要保持一个 Loader 的功能单一，避免做多种功能，只需完成一种功能转换即可。

所以如 less 文件转换成 css 文件，也不是一步到位，而是 less-loader, css-loader, style-loader 几个 loader 的链式调用才能完成转换。

### 模块

因为 Webpack 本身是运行在 Node.js 之上的，一个 loader 其实就是一个 node 模块，这个模块导出的是一个函数，即：

```javascript
module.exports = function (source) {
  // source 为 compiler 传递给 Loader 的一个文件的原内容
  // 处理...
  return source // 需要返回处理后的内容
}
```

这个导出的函数的工作就是获得处理前的原内容，对原内容执行处理后，返回处理后的内容。

### 替换字符串的 loader

比如我们打包时，想要替换源文件的字符串，这时可以考虑使用 Loader，因为 loader 就是获得源文件内容然后对其进行处理，再返回。

比如 src 目录下有三个文件：

src/msg1.js

```javascript
export const msg1 = '学习框架'
```

src/msg2.js

```javascript
export const msg2 = '深入理解JS'
```

src/index.js

```javascript
import { msg1 } from './msg1'
import { msg2 } from './msg2'

function print() {
  console.log(`输出：${msg1}, ${msg2}`)
}

print()
```

做的事情则是把 msg1 和 msg2 两个文件导入，然后输出两个字符串。

我们要做的事也很简单，把"框架"转为"React 框架"， "JS"转为"JavaScript"。

新建 `src/loaders/replaceLoader.js`文件，

```javascript
module.exports = function (source) {
  const handleContent = source.replace('框架', 'React框架').replace('JS', 'JavaScript')
  return handleContent
}
```

就这样，loader 写完了！！！

上面我们讲到，source 是源文件内容，如果打印的话，则是：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ad0aa5b46ede404eb4d1c2602e609ccb~tplv-k3u1fbpfcp-watermark.webp)

### 使用 Loader

接下来，我们要来使用它，在根目录下新建文件 webpack.config.js

```javascript
const path = require('path')

module.exports = {
  mode: 'production',
  entry: './src/index.js',
  module: {
    rules: [
      {
        test: /\.js$/,
        use: './src/loaders/replaceLoader.js',
      },
    ],
  },
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].js',
  },
}
```

执行`npx webpack`,
查看打包结果`dist/main.js`

```javascript
;(() => {
  'use strict'
  console.log('输出：学习React框架, 深入理解JavaScript')
})()
```

替换成功！

需要注意的是，`use`里面填写的 loader 是去`node_modules`目录里面找的，由于我们是自定义的 loader，所以不能直接写`use: 'replaceLoader'`，但直接写路径的方式未免难看点，我们可以通过 webpack 来配置：

```javascript
module.exports = {
  resolveLoader: {
    modules: ['node_modules', './src/loaders'], // node_modules找不到，就去./src/loaders找
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        use: 'replaceLoader',
      },
    ],
  },
}
```

### 获取 loader 的 options

写完之后，让我们来想想，其实就是写一个功能函数嘛。

当然，这只是最简单的例子，如果 loader 可以传入参数呢，比如：

```javascript
module: {
  rules: [
    {
      test: /\.js$/,
      use: {
        loader: 'replaceLoader',
        options: {
          params: 'replaceString',
        },
      },
    },
  ],
},
```

这个时候可以使用`this.query`来获取，通过`this.query.params`就能拿到，这里需要注意的是，this 上下文是有用的，所以这个 loader 导出函数不能是箭头函数。

但 webpack 更推荐`loader-utils`模块来获取，它提供了许多有用的工具，最常用的一种工具是获取传递给 loader 的选项。

首先要安装

```
npm i -D loader-utils
```

修改`src/loaders/replaceLoader.js`

```javascript
const { getOptions } = require('loader-utils')

module.exports = function (source) {
  console.log(getOptions(this)) // { params: 'replaceString' }
  console.log(this.query.params) // replaceString
  const handleContent = source.replace('框架', 'React框架').replace('JS', 'JavaScript')
  return handleContent
}
```

这里需要注意的是，`getOptions(this)`参数传入的是 this，也就是说

打印结果：

```
{ params: 'replaceString' }
{ params: 'replaceString' }
{ params: 'replaceString' }
```

### this.callback()

上面都是返回原来内容转换后的内容，但有些场景下还需要返回其他东西比如 sourceMap

```javascript
module.exports = function (source) {
  // 告诉 Webpack 返回的结果
  this.callback(null, source, sourceMaps)
}
```

另外也不需要 return 了，所以也可使用此 API 替代 return

```javascript
const { getOptions } = require('loader-utils')

module.exports = function (source) {
  const handleContent = source.replace('框架', 'React框架').replace('JS', 'JavaScript')
  this.callback(null, handleContent)
}
```

### 自定义 loader 应用场景

1. 在所有 function 外面加一层 try catch 代码块捕获错误，避免手动繁琐添加。
2. 实现中英文替换：可以将文字用占位符如`{{ title }}`包裹，检测到占位符则根据环境变量替换为中英文。

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~

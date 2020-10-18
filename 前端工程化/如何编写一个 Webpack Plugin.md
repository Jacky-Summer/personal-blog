# 如何编写一个 Webpack Plugin

## 前言

上次写了 [如何编写一个 Webpack Loader](https://juejin.im/post/6882895689773383694)，今天来说说如何编写一个 Webpack Plugin。

## webpack 内部执行流程

一次完整的 webpack 打包大致是这样的过程：

- 将命令行参数与 webpack 配置文件 合并、解析得到参数对象。
- 参数对象传给 webpack 执行得到 Compiler 对象。
- 执行 Compiler 的 run 方法开始编译。每次执行 run 编译都会生成一个 Compilation 对象。
- 触发 Compiler 的 make 方法分析入口文件，调用 compilation 的 buildModule 方法创建主模块对象。
- 生成入口文件 AST(抽象语法树)，通过 AST 分析和递归加载依赖模块。
- 所有模块分析完成后，执行 compilation 的 seal 方法对每个 chunk 进行整理、优化、封装。
- 最后执行 Compiler 的 emitAssets 方法把生成的文件输出到 output 的目录中。

## Plugin 作用

按我的理解，Webpack 插件的作用就是在 webpack 运行到某个时刻的时候，帮我们做一些事情。

在 Webpack 运行的生命周期中会广播出许多事件，Plugin 可以监听这些事件，在合适的时机通过 Webpack 提供的 API 改变输出结果。

官方解释是：

> 插件向第三方开发者提供了 webpack 引擎中完整的能力。使用阶段式的构建回调，开发者可以引入它们自己的行为到 webpack 构建流程中。

## 编写 Plugin

webpack 插件的组成：

- 一个 JS 命名函数或一个类（可以想下我们平时使用插件就是 `new XXXPlugin()`的方式）
- 在插件类/函数的 (prototype) 上定义一个 apply 方法。
- 通过 apply 函数中传入 compiler 并插入指定的事件钩子，在钩子回调中取到 compilation 对象
- 通过 compilation 处理 webpack 内部特定的实例数据
- 如果是插件是异步的，在插件的逻辑编写完后调用 webpack 提供的 callback

比如我们写一个插件，生成一个版权的文件。

### 基本雏形

```javascript
function CopyrightWebpackPlugin() {}

CopyrightWebpackPlugin.prototype.apply = function (compiler) {}

module.exports = CopyrightWebpackPlugin
```

也可以写成类的形式：

```javascript
class CopyrightWebpackPlugin {
  apply(compiler) {
    console.log(compiler)
  }
}

module.exports = CopyrightWebpackPlugin
```

webpack 在启动之后，在读取配置的过程中会先执行`new CopyrightWebpackPlugin(options)`操作，初始化一个`CopyrightWebpackPlugin`实例对象。在初始化 compiler 对象之后，会调用上述实例对象的`apply`方法并将`compiler`对象传入。

在`apply`方法中，通过`compiler`对象来监听 webpack 生命周期中广播出来的事件，我们也可以通过 compiler 对象来操作 webpack 的输出。

### Compiler 和 Compilation

在插件开发中最重要的两个对象是 `compiler` 和 `compilation` 对象。

> compiler 对象代表了完整的 webpack 环境配置，在初始化 compiler 对象之后，通过调用插件实例的 apply 方法，作为其参数传入。这个对象在启动 webpack 时被一次性建立，并包含了 webpack 环境的所有的配置信息，包括 options，loader 和 plugin。当在 webpack 环境中应用一个插件时，插件将收到此 compiler 对象的引用。可以使用它来访问 webpack 的主环境。

> compilation 对象会作为 plugin 内置事件回调函数的参数，一个 compilation 对象包含了当前的模块资源、编译生成资源、变化的文件以及被跟踪依赖的状态信息。当 Webpack 以开发模式运行时，每当检测到一个文件变化，一次新的 compilation 将被创建。compilation 对象也提供了很多事件回调供插件做扩展。通过 compilation 也能读取到 compiler 对象。

### 编码

下面代码为生成一个版权 txt 文件，新建文件`src/plugins/copyright-webpack-plugin.js`：

```javascript
class CopyrightWebpackPlugin {
  apply(compiler) {
    // emit 钩子是生成资源到 output 目录之前执行，emit 是一个异步串行钩子，需要用 tapAsync 来注册
    compiler.hooks.emit.tapAsync('CopyrightWebpackPlugin', (compilation, callback) => {
      // 回调方式注册异步钩子
      const copyrightText = '版权归 JackySummer 所有'
      // compilation存放了这次打包的所有内容
      // 所有待生成的文件都在它的 assets 属性上
      compilation.assets['copyright.txt'] = {
        // 添加copyright.txt
        source: function () {
          return copyrightText
        },
        size: function () {
          // 文件大小
          return copyrightText.length
        },
      }
      callback() // 必须调用
    })
  }
}

module.exports = CopyrightWebpackPlugin
```

webpack 中许多对象扩展自 Tapable 类。这个类暴露 tap, tapAsync 和 tapPromise 方法，可以使用这些方法，注入自定义的构建步骤，这些步骤将在整个编译过程中不同时机触发。

使用 tapAsync 方法来访问插件时，需要调用作为最后一个参数提供的回调函数。

在 webpack.config.js

```javascript
const path = require('path')
const CopyrightWebpackPlugin = require('./src/plugins/copyright-webpack-plugin')

module.exports = {
  mode: 'production',
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].js',
  },
  plugins: [new CopyrightWebpackPlugin()],
}
```

执行 webpack 命令，就会看到 dist 目录下生成`copyright.txt`文件

如果在配置文件使用 plugin 时传入参数该怎么获得呢，可以在插件类添加构造函数拿到：

```javascript
 plugins: [
  new CopyrightWebpackPlugin({
    name: 'jacky',
  }),
],
```

在`copyright-webpack-plugin.js`中

```javascript
class CopyrightWebpackPlugin {
  constructor(options = {}) {
    console.log('options', options) // options { name: 'jacky' }
  }
}
```

参考文章： [揭秘 webpack plugin](https://www.jianshu.com/p/8e92f36e52da)

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~

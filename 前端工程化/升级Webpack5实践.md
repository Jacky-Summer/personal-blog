# 升级Webpack5实践

最近将公司项目由 webpack4 升级到了 webpack5，配置了 webpack5 的长效缓存后，二次构建速度直接提速了 80%以上，同时打包体积也减少了，当然前提是要调研清楚坑多不多。

网上有一些做法是直接全部升级相关的包，而我是一个个包升级过来，如果有必要再升级，有的包没有升级（如 babel-loader 等等），项目本身用的大多也是较新的版本。

## Webpack5 新特性

首先先要了解下升级 Webpack5 能给我们带来什么好处，先看看 Webpack5 的新特性

- **通过持久化硬盘缓存能力来提升构建性能**
- 通过更好的算法来改进长期缓存（降低产物资源的缓存失效率）
- **通过更好的 Tree Shaking 能力和代码的生成逻辑来优化产物的大小**
- 改善 web 平台的兼容性能力
- 清除了内部结构中，在 Webpack4 没有重大更新而引入一些新特性时所遗留下来的一些奇怪的 state
- 通过引入一些重大的变更为未来的一些特性做准备，使得能够长期的稳定在 Webpack5 版本上

对我来说最吸引人的还是构建性能的提升和更好的 tree shaking 能力来优化产物的大小。

## Webpack5 版本上为什么构建速度有了质的飞跃?

主要是因为：

webpack4 是根据代码的结构生成 chunkhash，添加了空白行或注释，会引起 chunkhash 的变化，webpack5 是根据内容生成 chunkhash，改了注释或者变量不会引起 chunkhash 的变化，浏览器可以继续使用缓存。

- 优化了对缓存的使用效率。在 webpack4 中，chunkId 与 moduleId 都是自增 id。只要我们新增一个模块，那么代码中 module 的数量就会发生变化，从而导致 moduleId 发生变化，于是文件内容就发生了变化。chunkId 也是如此，新增一个入口的时候，chunk 数量的变化造成了 chunkId 的变化，导致了文件内容变化。所以对实际未改变的 chunk 文件不能有效利用。webpack5 采用新的算法来计算确定性的 chunkId 和 moduleId。可以有效利用缓存。在 production 模式下，optimization.chunkIds 和 optimization.moduleIds 默认会设为’deterministic’。

- 新增了可以将缓存写入磁盘的配置项, 在命令行终止当前构建任务，再次启动构建时，可以复用上一次写入硬盘的缓存，加快构建过程。

> 以上三点解释引用自[Webpack5 构建速度提升令人惊叹，早升级早受益](https://juejin.cn/post/6961800266484023327)

简单了解上述后，简单记录下我在升级过程做的一些事情，要看完整的建议去官方升级文档查阅

## 直接升级依赖

以下一些库在我们项目直接升级到最新版本后是不需要做兼容的，本身配置不多，但是如果你项目插件写的配置项多就可能有需要做兼容的地方。

- webpack webpack-cli
- html-webpack-plugin
- terser-webpack-plugin
- mini-css-extract-plugin
- style-loader、css-loader、postcss、postcss-loader（升级）

## 兼容

### 资源模块类型

在 webpack4 及之前我们会用各种 loader 来处理一些资源，比如`file-loader`,`url-loader`，`raw-loader`等等，但 webpack5 内置了静态资源构建能力，不再需要安装这些额外的 loader，通过简单的配置就能实现静态资源的打包。

资源模块类型(asset module type)，通过添加 4 种新的模块类型，来替换所有这些 loader：

- `asset/resource`  发送一个单独的文件并导出 URL。之前通过使用  `file-loader`  实现。
- `asset/inline`  导出一个资源的 data URI。之前通过使用  `url-loader`  实现。
- `asset/source`  导出资源的源代码。之前通过使用  `raw-loader`  实现。
- `asset`  在导出一个 data URI 和发送一个单独的文件之间自动选择。之前通过使用  `url-loader`，并且配置资源体积限制实现。

例子：

```js
module: {
    rules: [
        {
            test: /\.(png|jpg|jpeg|gif)$/,
            type: `asset/resource`
        }
    ]
},
```

### webpack-dev-server

需要注意的是：webpack-dev-server v4.0.0+ requires node >= v12.13.0

升级 webpack-dev-server 至 ^4(next) 版本，否则 HMR 会有异常

- wbepack4 启动方式为：`webpack-dev-server`。
- webpack5 修改为：`webpack server`

```js
// v4
devServer: {
    contentBase: path.resolve(__dirname, '../dist'),
    compress: true,
    inline: true, // 在构建变化后的代码会通过代理客户端来控制网页刷新
    host: '0.0.0.0',
    port: PORT,
    clientLogLevel: 'info',
    historyApiFallback: true,
    stats: 'errors-only',
    disableHostCheck: true,
}
```

```js
// v5
devServer: {
    // contentBase 变为 static 对象里面来配置
    static: {
      directory: path.resolve(__dirname, '../dist'),
    },
    client: {
      logging: 'info',
    },
    compress: true,
    // inline: true, // 直接移除，没有替代项
    host: '0.0.0.0',
    port: PORT,
    historyApiFallback: true,
    allowedHosts: 'all', // 代替 disableHostCheck: true
    // 新增中间件配置
    devMiddleware: {
      stats: 'errors-only',
    },
 },
```

相关文章：
[webpack-dev-server v3 迁移 v4 指南](https://github.com/webpack/webpack-dev-server/blob/master/migration-v4.md)

### Webpack5 移除了 Node.js Polyfill（兼容）

Webpack5 移除了 Node.js Polyfill，将会导致一些包变得不可用（会在控制台输出 'XXX' is not defined），如果需要兼容 process/buffer 等 Nodejs Polyfill，则要安装相关的 Polyfill：process，并在 Plugin 中显式声明注入。业务代码中是有使用的 process 变量的，故需要兼容，同时要安装 process/buffer 库。

```js
{
  plugins: [
    new webpack.ProvidePlugin({
      process: "process/browser",
    }),
  ];
}
```

### 升级废弃的配置项

未改动前的项目配置项含义（config/webpack_prod.ts）：

```js
splitChunks: {
  chunks: 'all',
  minSize: 30000, // 模块要大于30kb才会进行提取
  minChunks: 1, // 最小引用次数，默认为1
  maxAsyncRequests: 5, // 异步加载代码时同时进行的最大请求数不得超过5个
  maxInitialRequests: 3, // 入口文件加载时最大同时请求数不得超过3个
  automaticNameDelimiter: '_', // 模块文件名称前缀
  name: true, // 自动生成文件名
  cacheGroups: {
    // 将来自node_modules的模块提取到一个公共文件中
    vendors: {
      test: /[\\/]node_modules[\\/]/,
      priority: -10, // 执行优先级，默认为0
    },
    // 其他不是node_modules中的模块，如果有被引用不少于2次，那么也提取出来
    default: {
      minChunks: 2,
      priority: -20,
      reuseExistingChunk: true, // 如果当前代码块包含的模块已经存在，那么不在生成重复的块
    },
  },
},
```

```js
// v5
splitChunks: {
  cacheGroups: {
    // vendors ——> defaultVendors
    defaultVendors: {
      test: /[\\/]node_modules[\\/]/,
      priority: -10, // 执行优先级，默认为0
    },
  },
},
```

这个在（config/webpack_prod.ts）里面改个名字即可

**splitChunks.name（移除）**

splitChunks.name 表示抽取出来文件的名字

- 在 v4 中该配置默认为 true 表示自动生成文件名
- v5 移除了 optimization.splitChunks 的 name: true：不再支持自动命名

迁移：使用默认值。chunkIds: "named" 会为你的文件取一个有用的名字，以便于调试

### output 配置（兼容）

`(node:58337) [DEP_WEBPACK_TEMPLATE_PATH_PLUGIN_REPLACE_PATH_VARIABLES_HASH] DeprecationWarning: [hash] is now [fullhash] (also consider using [chunkhash] or [contenthash], see documentation for details)`

hash 已经不推荐使用了，改为了 fullhash，这个名字比起原来的 hash 更清楚
config/webpack_dev.ts

```js
// v4
output: {
    filename: 'js/[name].[hash].js',
    chunkFilename: 'js/[name].[hash].js',
},
// v5
output: {
    filename: 'js/[name].[fullhash].js',
    chunkFilename: 'js/[name].[fullhash].js',
},
```

### 优化配置项（废弃移除）

（config/webpack_prod.ts）

// webpack v4，在 v5 已被废弃，故需移除

```js
new webpack.HashedModuleIdsPlugin();
```

HashedModuleIdsPlugin 作用：实现持久化缓存。模块 ID 通过 HashedModuleIdsPlugin 来进行计算，它会把基于数字增量的 ID 替换成模块自身的 hash。这样的话，一个模块的 ID 只会在重命名或者移除的时候才会改变，新模块不会影响到它的 ID 变化。

webpack5 增加了确定的 moduleId，chunkId 的支持，如下配置：

```js
optimization.moduleIds = "deterministic";
optimization.chunkIds = "deterministic";
```

此配置在生产模式下是默认开启的，它的作用是以确定的方式为 module 和 chunk 分配 3-5 位数字 id，替代 v4 版本的 HashedModuleIdsPlugin。

### 全局变量写法（兼容）

```js
module.exports = () => {
  return {
    // ...
    plugins: [
      // webpack5 定义环境变量的写法变了
      new webpack.DefinePlugin({
        "process.env.NODE_ENV": JSON.stringify("production"),
      }),

      // webpack4的写法
      //   new webpack.DefinePlugin({
      //    'process.env': {
      //       NODE_ENV: JSON.stringify('production'),
      //     },
      //   }),
    ],
  };
};
```

### 使用 cache 属性，缓存进行优化（新增）

默认情况下 webpack5 不会启用持久化缓存，不自己手动加 cache 配置，webpack5 等于没优化，直接升级二次构建反而更慢；并不是升级就自动提速，还是要手动配置
当设置 cache.type: "filesystem" 时，webpack 会在内部以分层方式启用文件系统缓存和内存缓存。 从缓存读取时，会先查看内存缓存，如果内存缓存未找到，则降级到文件系统缓存。 写入缓存将同时写入内存缓存和文件系统缓存。
文件系统缓存不会直接将对磁盘写入的请求进行序列化。它将等到编译过程完成且编译器处于空闲状态才会执行。 如此处理的原因是序列化和磁盘写入会占用资源，并且我们不想额外延迟编译过程。

```js
cache: {
    // 将缓存类型设置为文件系统,默认为memory
    type: 'filesystem',

    buildDependencies: {
      // 当构建依赖的config文件（通过 require 依赖）内容发生变化时，缓存失效
      config: [__filename],
      // 如果你有其他的东西被构建依赖，你可以在这里添加它们
    },
},
```

为了防止缓存过于固定，导致更改构建配置无感知，依然使用旧的缓存，默认情况下，每次修改构建配置文件都会导致重新开始缓存。当然也可以自己主动设置 version 来控制缓存的更新。

## 最终优化成果

第一次编译：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e72577f178a44b3faf5d793bbc50a5e3~tplv-k3u1fbpfcp-watermark.image?)

第二次编译：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c81461a61371420da0e69b784a536606~tplv-k3u1fbpfcp-watermark.image?)

二次构建提速 85%以上

## 参考文章

- [Webpack5 新特性业务落地实战](https://zhuanlan.zhihu.com/p/348612482)
- [Webpack5 官网](https://webpack.js.org/concepts/)
- [webpack4 优化性能](https://www.jianshu.com/p/ac7e26187beb)
- [构建效率大幅提升，webpack5 在企鹅辅导的升级实践](https://juejin.cn/post/6937609106022727717)
- [Webpack5 构建速度提升令人惊叹，早升级早受益](https://juejin.cn/post/6961800266484023327)

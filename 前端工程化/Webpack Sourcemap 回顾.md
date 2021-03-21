# Webpack Sourcemap 回顾

## 前言

前几天在优化项目时，发现`next.config.js`的配置里，development 模式下的 sourcemap 为`cheap-module-sourcemap`，仔细想了想，记忆中好像有个更推荐常用的开发模式 sourcemap 配置：`cheap-module-eval-source-map`。

看了一下是三年前添加的代码，于是又回顾研究了下 webpack 的 sourcemap 配置，开个 PR。

## 什么是 Sourcemap

我们的项目在打包后，将开发环境中源代码经过压缩，去空格，babel 编译等工程化转换，最终的项目代码和源代码之间差异性很大，会造成无法 debug 的问题，在线上环境定位到的代码是压缩处理后的代码。

而 Sourcemap 就是是为了解决开发代码与实际运行代码不一致时帮助我们 debug 到原始开发代码的技术，解决上述**代码定位**的问题，是**源代码和目标代码出错位置的映射**。

## Sourcemap 关键词

Sourcemap 的关键词组合眼花缭乱，我们不可能一个一个去记每种 sourcemap 用途，我们只需知道几个关键词的意思，便可推测它们组合的 sourcemap 类型。

### eval

**每一个模块都执行 eval() 过程，执行后不会生成`.map`文件，而是在每一个模块后追加`//@ sourceURL`来关联代码处理前后的对应关系**。使用 eval 包裹模块代码，可以提高 **rebuild** 的速度。故一般 sourcemap 带有`eval`的选项，`rebuild` 速度都快一些。

```
webpackJsonp([1],[
function(module,exports,__webpack_require__){
eval(
      ...
//# sourceURL=webpack:///./src/js/index.js?'
    )
  },
function(module,exports,__webpack_require__){
eval(
      ...
//# sourceURL=webpack:///./src/static/css/app.less?./~/.npminstall/css-loader/0.23.1/css-loader!./~/.npminstall/postcss-loader/1.1.1/postcss-loader!./~/.npminstall/less-loader/2.2.3/less-loader'
    )
  },
function(module,exports,__webpack_require__){
 eval(
      ...
 //# sourceURL=webpack:///./src/tmpl/appTemplate.tpl?"
    )
  },
...])
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42794b4bbbc3449599a92327f6c6d45a~tplv-k3u1fbpfcp-watermark.image)

为什么 eval 模式 rebuild 的速度会快?

> 因为带 eval 的构建模式能够 cache Sourcemap，猜测可能是带 eval 的模式能把每个模块的生成代码单独打在一行里，并且都以 DataURL 的形式添加 sourcemap，这样它在 rebuild 时会对每一个模块进行检查，如果没有发生改动，就直接使用之前的 sourcemap，故只需要为改动的模块重新生成 sourcemap；而非 eval 的构建模式下生成代码不管有任何模块作出修改，都会影响到最后 bundle 文件整体的行列格式，所以它每次都要重新生成整体的 sourcemap，rebuild 的速度会很慢。

### source-map

会为每一个打包后的模块生成独立的`.map`文件，会在 bundle 文件末尾追加 `sourceURI=指定.map文件路径`，会在浏览器开发者工具中看到`webpack://`的文件夹

```
webpackJsonp([1],[
function(e,t,i){...},
function(e,t,i){...},
function(e,t,i){...},
function(e,t,i){...},
  ...
])//# sourceMappingURL=index.js.map
```

打包后的模块在模块后面会对应引用一个`.map`文件，同时在打包好的目录下会针对每一个模块生成相应的`.map` 文件，在上例中会生成一个 index.js.map 文件，这个文件是一个典型的 sourcemap 文件，形式如下：

```
{
"version":3,
"sources":[
    "webpack:///js/index.js","webpack:///./src/js/index.js",
    "webpack:///./~/.npminstall/css-loader/0.23.1/css-loader/lib/css-base.js",
    ...
],
"names":["webpackJsonp","module","exports"...],
"mappings":"AAAAA,cAAc,IAER,SAASC...",
"file":"js/index.js",
"sourcesContent":[...],
"sourceRoot":""
}
```

### cheap

不包含列信息也不包含 loader 的 sourcemap，会为每一个模块生成`.map`文件，与`source-map`的区别在于 cheap 生成的`.map`文件会忽略原始代码中的列信息。

由于生成的 sourcemap 不会有列信息而只有行信息，编译计算量少，所以速度较快。

### module

包含 **loader** 模块之间的 sourcemap（比如 jsx 语法代码经 loader 编译为原生 js 代码）,这样可以看到 loader 处理前的原始代码

### inline

正常的 sourcemap 的生成是在 dist 目录下创建一个`.map` 文件， inline 的含义就是不产生独立的 `.map` 文件，把 sourcemap 的内容以 **DataURI** 的方式追加到 bundle 件末尾，即将`.map` 作为 DataURI 嵌入。（一般使用该类型如造成体积过大，该类型比较少用）

```
webpackJsonp([1],[
function(e,t,i){...},
function(e,t,i){...},
function(e,t,i){...},
function(e,t,i){...},
  ...
])
//# sourceMappingURL=data:application/json;charset=utf-8;base64,eyJ2ZXJzaW9...
```

> DataURL 最早是出现在 HTML 文件 img 标签中的关于图片的引用，如 base64 图片

DataURL 使用于如下的场景

- 访问外部资源受限
- 图片体积小，占用一个 HTTP 会话资源浪费

## hidden

bundle 里不包含 sourcemap 的引用地址，这样浏览器开发者工具里看不到原始代码

## Sourcemap 类型

上述的关键词混一起搭配，又增加了好几种 sourcemap 类型

- **eval-source-map**：每一个模块在执行 eval()过程之后，并且会为每一个模块生成 `.map` 文件，生成的 sourcemap 文件通过 DataURL 的方式添加（把 eval 的 sourceURL 换成了完整 sourcemap 信息的 DataURL）

- **cheap-eval-source-map**：跟 eval-source-map 相同，唯一不同的就是增加了"cheap"，"cheap"是指忽略了行信息。这个属性同时也不会生成不同 loader 模块之间的 sourcemap。

- **cheap-module-eval-source-map**：与 cheap-eval-source-map 相同，但是包含了不同 loader 模块之间的 sourcemap

- **cheap-source-map**： 不包含列信息，不包含 loader 的 sourcemap

- **inline-source-map**： 为每一个文件添加 sourcemap 的 DataURL，注意这里的文件是打包前的每一个文件而不是最后打包出来的，同时这个 DataURL 是包含一个文件完整 sourcemap 信息的 base64 格式化后的字符串

-**hidden-source-map**：不在 bundle 文件结尾处追加 sourceURL 指定其 sourcemap 文件的位置，但是仍然会生成 sourcemap 文件。这样，浏览器开发者工具就无法应用 sourcemap, 目的是避免把 sourcemap 文件发布到生产环境，造成源码泄露。

不仅还有这几种，还可以继续组合，这里就列举这么多了

## 如何选择 Sourcemap

- 首先在源代码的列信息意义不大，因为只要有行信息就能完整的建立打包前后代码之间的依赖关系，够我们定位了。
- 其次，不管在生产环境还是开发环境，我们都需要定位 debug 到最最原始的资源，比如定位错误到 jsx 的原始代码处，而不是编译成 js 的代码处，因此，不能忽略 module 属性。
- `eval-source-map` 使用 DataURL 本身包含完整 sourcemap 信息，并不需要像 sourceURL 那样，浏览器需要发送一个完整请求去获取 sourcemap 文件，这会略微提高点效率

开发环境中使用：**cheap-module-eval-source-map**（该配置值能保留 loader 处理前的原始代码信息，而打包速度也较快，是一个较佳的选择。）

生产环境中使用 sourcemap 会有泄露源代码的风险，但如果要保留定位线上的错误，应该禁止浏览器开发者工具看到源代码，而是用一些错误收集系统，将 sourcemap 文件传到系统上，通过系统 source map 分析出原始代码的错误堆栈，如使用`hidden-source-map`。

当然，也有使用`nosources-source-map`, `source-map`的，依据你们需求场景选择。总之，最终结果是不能被人通过开发者工具看到源代码的。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a674f3f10dc0482da017738a79772d50~tplv-k3u1fbpfcp-watermark.image)

## 结语

我们项目目前还是 Webpack4.x，如果你们项目已经用 Webpack5，sourcemap 名称有些不
同，具体请查阅 Webpack5 文档。

## 参考

[Webpack 中的 sourcemap 以及如何在生产和开发环境中合理的设置 sourcemap 的类型](https://github.com/forthealllight/blog/issues/6)

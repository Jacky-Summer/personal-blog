## 前言

今天动动手回顾下 Webpack 基本配置，搭了个 react 项目脚手架，做下笔记。

本文适合有了解过 webpack 基础配置的人阅读，因为有些基础配置和流程我就不做详细解释了。

## 初始化

新建文件夹`react-cli`，初始化

```
npm init -y
```

- 安装 webpack

```
yarn add webpack webpack-cli -D
```

我们知道 webpack 主要是帮我们打包构建，我们新建个文件`src/index.js`

```javascript
console.log('hello webpack')
```

- 新建 webpack.config.js

```javascript
const path = require('path')

module.exports = {
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'app.js',
  },
}
```

再到`package.json`设置打包命令

```
"scripts": {
  "build": "npx webpack --config webpack.config.js"
},
```

因为我没有全局安装 webpack，这里使用 npx（npm5.2 开始自带 npx 命令）
关于 npx 的使用可以看阮一峰大神这篇博客：[npx 使用教程](https://www.ruanyifeng.com/blog/2019/02/npx.html)

- 执行打包命令

```
npm run build
```

就会看到生成了`dist`文件夹和打包生成的文件`app.js`

## 引入 Html 模板

> `html-webpack-plugin`简化了 HTML 文件的创建，该插件将为你生成一个 HTML5 文件， 其中包括使用 script 标签的 body 中的所有 webpack 包。

```
yarn add html-webpack-plugin -D
```

新建模板 HTML 文件`public/index.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <div id="root">Hello</div>
  </body>
</html>
```

在`webpack.config.js`配置

```javascript
module.exports = {
  plugins: [
    new HtmlWebpackPlugin({
      template: './public/index.html',
    }),
  ],
}
```

再执行`npm run build`，会看到生成了一个`index.html`文件，打开它会发现`app.js`也被引入到文件且能正常运行

## 编译 JS 文件

ES6 或以上版本的语法有些浏览器还是不识别语法的，所以我们需要用 babel 对它进行编译，编译成 ES5 代码
在`src/index.js`写一个 ES6 的箭头函数

```
const func = () => {
  return 100
}
```

安装 babel 插件

```
yarn add babel-loader @babel/core @babel/preset-env -D
```

- **babel-loader**：使用 Babel 和 webpack 来转译 JavaScript 文件。
- **@babel/preset-env**：将语法翻译成 es5 语法
- **@babel/core**：babel 的核心模块

在根目录下新建`.babelrc`

```
{
  "presets": ["@babel/preset-env"]
}
```

在`webpack.config.js`中配置解析 js 的规则

```javascript
module.exports = {
  module: {
    rules: [
      {
        test: /\.(js|jsx)$/,
        loader: 'babel-loader',
        exclude: /node_modules/,
      },
    ],
  },
}
```

执行`npm run build`，查看`app.js`已经把箭头函数编译成 ES5 语法了

```javascript
/***/ ;(function (module, exports) {
  eval(
    "console.log('hello webpack');\n\nvar func = function func() {\n  return 100;\n};\n\n//# sourceURL=webpack:///./src/index.js?"
  )

  /***/
})
```

## 处理浏览器没有的新特性

有些浏览器没有 Promise、Symbol、Map 等一些新特性，就需要使用 polyfill 来实现；

首先来理解下 babel 和 babel polyfill 的关系：

1. babel 解决**语法**层面的问题，比如将 ES6+语法转为 ES5 语法。
2. babel polyfill 解决**API**层面的问题，需要使用垫片，比如下面三种：

- **@babel/polyfill**：通过向全局对象和内置对象的 prototype 上添加方法来实现的，从而解决了低版本浏览器无法实现的一些 es6+语法；但污染了全局空间和内置对象也是个缺点。
- **@babel/runtime**：不会污染全局空间和内置对象原型，但如果使用的特性非常多，就会导致这种按需引入非常繁琐。由于它是 polyfill 的包，适合在组件，类库中使用，所以需要添加到生产环境中。
- **@babel/plugin-transform-runtime**：将 helper 和 polyfill 都改为从一个统一的地方引入，解决了全局空间和内置对象的问题；需要用到两个依赖包 `@babel/runtime`和`@babel/runtime-corejs3`——加载 ES6+ 新的 API（corejs 根据你的版本）

- babel-runtime 适合在组件，类库项目中使用，而 babel-polyfill 适合在业务项目中使用。

polyfill 用到的 core-js 是可以指定版本的，比如使用 `core-js@3` `core-js@2`

```
yarn add @babel/polyfill core-js@3
yarn add @babel/plugin-transform-runtime -D
yarn add @babel/runtime @babel/runtime-corejs3
```

安装完毕后，在 `.babelrc`进行配置：

```
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "useBuiltIns": "usage", // 按需引入，它能自动给每个文件添加其需要的polyfill
        "corejs": 3 // 根据你的版本来写
      }
    ]
  ],
  "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        "corejs": 3,
        "helpers": true, // 开启内联的babel helpers(即babel或者环境本来的存在的垫片或者某些对象方法函数)
        "regenerator": true, // 开启generator函数转换成使用regenerator runtime来避免污染全局域
        "useESModules": false
      }
    ]
  ]
}
```

## 编译 React

安装 react 有关包和编译 React 的 babel 插件

```
yarn add react react-dom
yarn add @babel/preset-react -D
```

重写`src/index.js`

```javascript
import React from 'react'
import ReactDOM from 'react-dom'
import App from './App.jsx'

ReactDOM.render(<App />, document.getElementById('root'))
```

新建文件`src/App.jsx`

```javascript
import React from 'react'

const App = () => <div>App</div>

export default App
```

修改`.babelrc`文件（presets 的顺序从右向左）

```
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "useBuiltIns": "usage", // 按需引入，它能自动给每个文件添加其需要的polyfill
        "corejs": 3 // 根据你的版本来写
      }
    ],
    "@babel/preset-react"
  ],
}
```

执行打包命令，成功编译

## devServer 配置

安装 devServer，在开发中实时检测文件的变化

```
yarn add webpack-dev-server -D
```

在``package.json`中设置启动命令

```
"dev": "webpack-dev-server",
```

webpack.config.js

```javascript
module.exports = {
  devServer: {
    publicPath: '/',
    host: '0.0.0.0',
    compress: true, // 使用gzip压缩
    port: 8080, // 监听的端口号
    historyApiFallback: true,
    overlay: true, // 代码出错弹出浮动层
    hot: true, // 开启热更新
  },
}
```

开启服务器`yarn dev`，打开`http://localhost:8080`，就可以 React 组件正常编译运行了。

## 开发环境热更新

```
yarn add -D react-hot-loader
```

在`webpack.config.js`配置，devServer 上面已经开启`hot:true`

```javascript
plugins: [
  new webpack.HotModuleReplacementPlugin(),
],
```

在`src/index.js`中添加

```javascript
import React from 'react'
import ReactDOM from 'react-dom'
import App from './App'

if (module && module.hot) {
  module.hot.accept()
}

ReactDOM.render(<App />, document.getElementById('root'))
```

重新运行`yarn dev`，热更新生效

## 省略文件后缀和设置别名

上面引入 App 组件我们是`import App from './App.jsx'`加了后缀，如果省略的话会报错；我们可以在 webpack 配置文件的`resolve`设置可以省略后缀；

还有一个问题，如果项目较大，组件层级关系多的时候，我们可以给路径设置别名

新建`src/components/Header/index.js`

```javascript
import React from 'react'

const Header = () => <div>Header</div>

export default Header
```

在 webpack.config.js 进行配置

```javascript
module.exports = {
  mode: 'development',
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'app.js',
  },
  resolve: {
    extensions: ['.jsx', '.js', '.json'],
    alias: {
      '@components': path.resolve(__dirname, 'src/components'),
    },
  },
```

修改`App.jsx`

```javascript
import React from 'react'
import Header from '@components/Header'

const App = () => (
  <div>
    <Header />
  </div>
)

export default App
```

## 加载 Css

```
yarn add style-loader css-loader -D
```

给 Header 组件添加样式，新建文件`src/components/Header/index.css`，再到 Header 组件中引入`import './index.css'`

```css
div {
  background-color: lightgreen;
}
```

- css-loader 的作用是处理 css 文件中 @import，url 之类的语句（将 CSS 转化成 CommonJS 模块）
- style-loader 则是将 css 文件内容放在 style 标签内并插入 head 中

`webpack.config.js`配置（loader 执行顺序从右到左）

```javascript
 module: {
    rules: [
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'],
      },
    ],
  },
}
```

## 支持 Sass

```
yarn add sass sass-loader -D
```

```javascript
module: {
  rules: [
    {
      test: /\.(sass|scss)$/,
      use: [
        'style-loader',
        'css-loader',
        {
          loader: 'sass-loader',
          options: {
            implementation: require('sass'), // 使用dart-sass代替node-sass
          },
        },
      ],
    },
  ],
},
```

把`src/components/Header/index.css`改为`.scss`文件，Header 组件`index.js`的引入也修改一下引入 css 文件名，然后执行打包命令，OK。

less 就不写了，毕竟一个项目选一种就行了。

## 集成 Postcss

postcss-loader 作用：

- 把 css 解析为一个抽象语法树
- 调用插件处理抽象语法树并添加功能，如处理 CSS 兼容添加一些新特性的厂商前缀

```
yarn add postcss-loader autoprefixer -D
```

在根目录下新建`postcss.config.js`

```javascript
module.exports = {
  plugins: [
    require('autoprefixer')({
      // 自动在样式中添加浏览器厂商前缀
      overrideBrowserslist: ['last 2 versions', '>1%'],
    }),
  ],
}
```

`webpack.config.js`中配置

```javascript
module: {
  rules: [
    {
      test: /\.css$/,
      use: ['style-loader', 'css-loader', 'postcss-loader'],
    },
    {
      test: /\.(sass|scss)$/,
      use: [
        'style-loader',
        'css-loader',
        'postcss-loader',
        {
          loader: 'sass-loader',
          options: {
            implementation: require('sass'), // 使用dart-sass代替node-sass
          },
        },
      ],
    },
  ]
}
```

在样式文件添加 CSS3 一些新特性，就能看到被自动添加了兼容处理了。

## 加载图片

```
yarn add file-loader url-loader -D
```

加入一张图片`src/components/Header/img/fe-house-qrcode.jpg`，在`src/components/Header/index.js`中引入

```
import './img/fe-house-qrcode.jpg'
```

在`webpack.config.js`中进行配置：

```javascript
module: {
  rules: [
    {
      test: /\.(png|jpe?g|gif)$/,
      use: {
        loader: 'url-loader',
        options: {
          name: '[name]_[hash].[ext]',
          outputPath: 'images/', // 打包图片到dist目录下的images文件夹
          limit: 102400, // 图片超过100KB就使用file-loader打包，否则图片以base64格式存在
        },
      },
    },
  ]
}
```

## 解析字体图标

这里就不演示了，直接上配置

```javascript
module: {
  rules: [
    {
      test: /\.(eot|ttf|svg|woff|woff2)$/,
      use: {
        loader: 'file-loader',
        options: {
          name: '[name]_[hash].[ext]',
          outputPath: 'fonts/',
        },
      },
    },
  ]
}
```

## 区分生产环境和开发环境

到这里，最基本配置差不多可以用了，接下来需要做一些细节或优化的配置。

安装`webpack-merge`整合

```
yarn add webpack-merge -D
```

提炼出三个文件：

- webpack.common.js 公共配置
- webpack.dev.js 开发环境配置
- webpack.prod.js 生产环境配置

将`webpack.config.js`的东西提炼出来分三个文件

- webpack.common.js

```javascript
const path = require('path')
const HtmlWebpackPlugin = require('html-webpack-plugin')

module.exports = {
  mode: 'development',
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'app.js',
  },
  resolve: {
    extensions: ['.jsx', '.js', '.json'],
    alias: {
      '@components': path.resolve(__dirname, 'src/components'),
    },
  },
  module: {
    rules: [
      {
        test: /\.(js|jsx)$/,
        loader: 'babel-loader',
        exclude: /node_modules/,
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader', 'postcss-loader'],
      },
      {
        test: /\.(sass|scss)$/,
        use: [
          'style-loader',
          'css-loader',
          'postcss-loader',
          {
            loader: 'sass-loader',
            options: {
              implementation: require('sass'), // 使用dart-sass代替node-sass
            },
          },
        ],
      },
      {
        test: /\.(png|jpe?g|gif)$/,
        use: {
          loader: 'url-loader',
          options: {
            name: '[name]_[hash].[ext]',
            outputPath: 'images/', // 打包图片到dist目录下的images文件夹
            limit: 102400, // 图片超过100KB就使用file-loader打包，否则图片以base64格式存在
          },
        },
      },
      {
        test: /\.(eot|ttf|svg|woff|woff2)$/,
        use: {
          loader: 'file-loader',
          options: {
            name: '[name]_[hash].[ext]',
            outputPath: 'fonts/',
          },
        },
      },
    ],
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: './public/index.html',
    }),
  ],
  devServer: {
    publicPath: '/',
    host: '0.0.0.0',
    compress: true, // 使用gzip压缩
    port: 8080, // 监听的端口号
    historyApiFallback: true,
    overlay: true, // 代码出错弹出浮动层
    hot: true, // 开启热更新
  },
}
```

- webpack.dev.js

```javascript
const webpack = require('webpack')
const { merge } = require('webpack-merge')
const commonConfig = require('./webpack.common')
const devConfig = {
  mode: 'development',
  devtool: 'cheap-module-eval-source-map', // 帮助我们定位到错误信息位置的源代码文件
  plugins: [new webpack.HotModuleReplacementPlugin()],
  devServer: {
    publicPath: '/',
    host: '0.0.0.0',
    compress: true, // 使用gzip压缩
    port: 8080, // 监听的端口号
    historyApiFallback: true,
    overlay: true, // 代码出错弹出浮动层
    hot: true, // 开启热更新
  },
}

module.exports = merge(commonConfig, devConfig)
```

- webpack.prod.js

```javascript
const { merge } = require('webpack-merge')
const commonConfig = require('./webpack.common')
const devConfig = {
  mode: 'production',
  devtool: 'none',
}

module.exports = merge(commonConfig, devConfig)
```

修改`package.json`

```
"scripts": {
  "dev": "webpack-dev-server --config webpack.dev.js",
  "build": "npx webpack --config webpack.prod.js"
},
```

大功告成。

## 打包前清理 dist 目录

在你调试打包的过程可能会多出一些遗留文件，我们想要最新打包编译的文件，就需要先清除 dist 目录

```
yarn add clean-webpack-plugin -D
```

加到`webpack.common.js`文件

```javascript
const { CleanWebpackPlugin } = require('clean-webpack-plugin')
module.exports = {
  plugins: [new CleanWebpackPlugin()],
}
```

## 项目规范

作为项目代码风格的统一规范，我们需要借助第三方工具来强制，让团队成员的代码风格更趋于一致。

### EditorConfig

`.editorconfig` 是跨编辑器维护一致编码风格的配置文件，在 VSCode 中需要安装相应插件`EditorConfig for VS Code`，安装完毕之后，
使用快捷键`ctrl+shift+p`打开命令台，输入 `Generate .editorcofig` 即可快速生成 `.editorconfig` 文件。

在`.editorcofig`文件，就可以大家根据不同来设置文件了，比如我的是这样：

```
root = true

[*]
indent_style = space
indent_size = 2
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true
end_of_line = lf
```

### Prettier

```
yarn add prettier -D
```

同样也需要安装 VSCode 插件`Prettier - Code formatter`

安装完成后在根目录下新建文件夹`.vscode`，在该文件夹下新建文件`settings.json`,该文件的配置优先于你自己 VSCode 全局的配置，不会因为团队不同成员的 VSCode 全局配置不同而导致格式化不同。

settings.json

```
{
  // 指定哪些文件不参与搜索
  "search.exclude": {
    "**/node_modules": true,
    "dist": true,
    "yarn.lock": true
  },
  "editor.formatOnSave": true,
  "eslint.validate": ["javascript", "javascriptreact"],
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true,
  },
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[javascriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[json]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[html]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[markdown]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[css]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[less]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[scss]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  }
}
```

安装完后新建文件`.prettierrc`

```
{
  "trailingComma": "all",
  "tabWidth": 2,
  "semi": false,
  "singleQuote": true,
  "endOfLine": "lf",
  "printWidth": 120,
  "bracketSpacing": true,
  "arrowParens": "avoid"
}
```

### ESLint

```
yarn add eslint -D
```

安装完后运行` npx eslint --init`，运行后有选项，选择如下（自行根据需要）：

>

- To check syntax, find problems, and enforce code style
- JavaScript modules (import/export)
- React
- No（这里我们没使用 TS 就不必了）
- Browser Node
- Use a popular style guide
- Airbnb: https://github.com/airbnb/javascript
- JavaScript
- Would you like to install them now with npm（Yes）直接安装

完毕后会发现已经自动安装了 ESLint 相关的包，也自动生成了`.eslintrc.js`文件，里面的`rules`默认是空，这个就需要团队自己的额外配置了。注：VSCode 需要安装`ESLint`插件

新建文件`.eslintignore`，忽略一些文件的检查

```
/node_modules
/public
/dist
```

### StyleLint

上面是针对规范 JS 代码的，接下来规范 CSS 代码，同样需要安装 VSCode 插件`stylelint`

```
yarn add stylelint stylelint-config-standard -D
```

新建文件`.stylelintrc.js`

```javascript
module.exports = {
  extends: ['stylelint-config-standard'],
  rules: {
    'comment-empty-line-before': null,
    'declaration-empty-line-before': null,
    'function-name-case': 'lower',
    'no-descending-specificity': null,
    'no-invalid-double-slash-comments': null,
    'rule-empty-line-before': 'always',
  },
  ignoreFiles: ['node_modules/**/*', 'dist/**/*'],
}
```

### Prettier、ESLint、Stylelint 冲突解决

上面一顿操作后，可能发现 ESLint 给文件飘红，但保存后格式变了又好像恢复原样，这个就是冲突问题，一般解决就是利用插件禁用与 Prettier 起冲突的规则。

```
yarn add eslint-config-prettier -D
```

修改`.eslintrc.js`

```javascript
module.exports = {
  extends: [
    //...
    'prettier',
    'prettier/react',
  ],
}
```

如果代码还因为 ESLint 飘红，可以看情况添加忽略的规则。

```
yarn add stylelint-config-prettier -D
```

修改`.stylelintrc.js`文件中增加一个 extend

```javascript
extends: ['stylelint-config-prettier'],
```

## 结语

写到这里就停了，当然要继续往下做的话还有很多，比如用 husky 代码提交，还有很多像 webpack 压缩分离 JS 和 CSS 等一些，这些留到以后写一篇 webpack 性能优化再来一起总结。

## 前言

平时大家在公司接手一个已有项目的时候，首先会看的是什么呢？我的习惯是先看 `README.md` 和 `package.json`。

通过 README 了解项目是做什么和注意点，通过`package.json`了解项目涉及的技术栈和 npm 库等等。

今天就来深入了解下`package.json`这个文件，不仅是解释详细字段含义与运用（忽略部分第三方字段本文就不介绍了），更重要的是想借此扩展总结下涉及工作中与社区知名库的一些实践，对我们自己做开源项目也有一定帮助。

## 常见配置字段

### name

**项目的名称**，该字段决定了你发布的包在 npm 的名字，也就是平时安装库的包名了 `npm i 包名`。

该字段也是有**命名规范**的，如下：

- 名称的长度必须小于或等于 214 个字符，不能以`.`和`_`开头。不要使用`空格`、`<>`、`[]`、`{}`、`|`、`\`、`^`、`%` 等。
- 不能使用大写字母命名
- 如果要发布到 npm 上，名称不能和社区已有的重复，可以使用`npm view`命令查询，或直接上[npm](https://www.npmjs.com/)查。比如随机想一个包名，`jacky-summer-utils`

```
npm view jacky-summer-utils

# 报 404，证明该名称可用
npm ERR! code E404
```

除了常规命名，我们还会见到社区有`@`开头的命名，格式如：`@[scope]/[name]`

例子：`@ant-design/icons`、`@babel/preset-env`，这代表一个组织下的库。

如果你也想使用这样的结构，比如用自己名字`@`开头的话，需要在自己 npm 上建立组织（`Add Organization`），不这么做的话包名带`@`则发布时会不通过。

比如我创建了个组织 `@summer-toolkit`，包名是`config`，则我的这个包和`name`值就命名为 [@summer-toolkit/config](https://github.com/Jacky-Summer/toolkit/blob/master/packages/config/package.json)

```
"name": "@summer-toolkit/config"
```

### version

**项目版本号**，当发布 npm 包时`name`和`version`这两个字段是必须的，两个共同构成唯一的项目标识，通过更改 `version` 来对你的 npm 包进行发布更新。

```
"version": "1.1.0"
```

通常推荐使用 [semver 版本控制策略](https://semver.org/)，开源项目也通常遵循这个语义化规范

> 版本格式：主版本号.次版本号.修订号，版本号递增规则如下：

1. 主版本号 Major，当你做了不兼容的 API 修改，通常在涉及重大功能更新，产生了破坏性变更时会更新此版本号
2. 次版本号 Minor，引入了新功能，但未产生破坏性变更，依然可以向下兼容时会更新此版本号，可理解为 Feature 版本
3. 修订号 Patch，修复了一些 bug，也未产生破坏性变更时会更新此版本号，可理解为 Bug Fix

当要发布大版本或者核心的功能时，无法确定该版本功能稳定，可能无法满足预期的兼容性需求，这个时候就需要通过发布**先行版本**。先行版本通过会加在版本号的后面，通过`-`号连接以`.`分隔的标识符和版本编译信息：

- 内部版本（alpha）：通常该版本软件的 Bug 较多，需要持续修改
- 公测版本（beta）：相对于 alpha 版已有很大改进，会继续加新特性，修复了较严重的错误，但依然存在一些缺陷
- 候选版本（rc，即 release candiate）：正式版本的候选版本，该版本一般可以说错误较少了，基本是修复而不是再加新特性了

```
react：16.3.0-alpha.0
vue：3.2.34-beta.1、3.0.0-rc.13
```

比如想查看 vue 的历史 Tag 版本，可看到 vue 是比较严格遵循 semver 版本规范的：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d4f88d8b7be4fffba4e9f784aacd1d6~tplv-k3u1fbpfcp-watermark.image?)

```
# 查看 vue 历史版本
npm view vue versions
# 查看 vue 最新版本
npm view vue version
```

一些问题解答：

> 在库初始开发阶段，该如何进行版本控制？

最简单的做法是以 0.1.0 作为你的初始化开发版本，并在后续的每次发行时递增次版本号。当最终完成用于正式线上环境了，这时就可以改为 1.0.0，之后就都按照 Semver 规范走了

> 公司一些较小的公共库版本，怎么维护？

这个可视乎库规模和使用人数多少决定，像一般公司大部分私库测试环境都可直接发布 beta 版本，等 UAT/线上就是直接走的正式版本

### description

**项目的描述**，信息会直接展示在 npm 官网，通过它能让别人能快速你的项目。

如`dayjs`库的：

```
"description": "2KB immutable date time library alternative to Moment.js with the same modern API "
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1dba19150fa94480b9dec1f546a4da86~tplv-k3u1fbpfcp-watermark.image?)

Webpack 的描述就相对长一些了：

```
"description": "Packs CommonJs/AMD modules for the browser. Allows to split your codebase into multiple bundles, which can be loaded on demand. Support loaders to preprocess files, i.e. json, jsx, es7, css, less, ... and your custom stuff.",
```

### keywords

**项目关键词**。关键词填得准，可方便在 npm 上更好地检索

如 Ant Design 的 `keywords`：

```
"keywords": [
    "ant",
    "component",
    "components",
    "design",
    "framework",
    "frontend",
    "react",
    "react-component",
    "ui"
],
```

Redux 的：

```
"keywords": [
    "redux",
    "reducer",
    "state",
    "predictable",
    "functional",
    "immutable",
    "hot",
    "live",
    "replay",
    "flux",
    "elm"
],
```

### repository

**项目的仓库地址以及版本控制信息**，可通过该字段找到代码托管地址

```json
// Ant Design
"repository": {
    "type": "git",
    "url": "https://github.com/ant-design/ant-design"
}
```

### license

**项目的开源许可证**，说明你的库使用哪个许可证，用户保护你自己和贡献者。

没有 `License` 的内容是默认会被版权保护，如果你允许社区内开发者可基于你的项目二次改造等，需要选择合适的许可证才能赋予任何人放心使用（改造、分享等）。

目前 Github 我们熟知的大部分项目都是 [MIT](https://choosealicense.com/licenses/mit/) 许可证

```
 "license": "MIT"
```

> MIT 基本就是没有任何限制，表示任何人都可以售卖我的软件，甚至可以用作者的名字促销。
> 项目的版权拥有人可以使用开源许可证来限制源码的使用、复制、修改和再发布等行为，

通常开源项目都会在根目录下新建 `LICENSE` 文件，并将许可证的文本复制到这里。如看下 [Vue](https://github.com/vuejs/vue/blob/main/LICENSE)的

对于公司不开源的项目，这个配置项一般可忽略。

了解详细可阅读：

- [七种开源许可证](https://zhuanlan.zhihu.com/p/62578705)
- [如何为自己的 Github 项目选择开源许可证？](https://zhuanlan.zhihu.com/p/51331026)

### private

**设置为私有防止意外发布**。如果你不希望把项目发布到 npm 仓库上，可以将 `private` 设为 true。

```
"private": true
```

当设置为 true 时，执行`npm publish`会报错，npm 会拒绝发布这个项目。

像公司的非开源项目就可以设置为 true，防止被无意间发布出去

### publishConfig

**npm 包发布使用的配置**，通常`publishConfig`会配合`private`使用；如果只想让模块发布到特定 npm 仓库，就可以这样来配置

```
"publishConfig": {
    "registry": "https://registry.npmjs.org", // 私源地址配置
    "tag": "beta", // 如果没有指定tag，默认是 latest
    "access": "public"
}
```

- registry：npm 私源地址
- tag：指定当前版本对应的 tag
- access：包的访问级别。如果是 scoped 包（如 `@babel/xxx，@ant-design/xxx`），一定需要设置为 public（除非加入付费计划）

### bugs

**bug 反馈地址**，通过该链接向你的仓库反馈 bug，github 上的一般都是 github issue 页的链接

```
"bugs": "https://github.com/webpack/webpack/issues",
```

### homepage

**项目主页地址**，如：仓库 Github 链接、官网地址、文档地址等

```json
// Ant Design：
"homepage": "https://ant.design"

// Redux：
"homepage": "http://redux.js.org"
```

`homepage`字段会展示在 npm 的这个地方：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a4b9172192a46c4be47f8d5621c0ddb~tplv-k3u1fbpfcp-watermark.image?)

`homepage` 还可设置应用的根路径，详细可见[【小技巧】package.json 中 homepage 属性的作用](https://juejin.cn/post/6844904077474676743)

### author

**项目的作者（信息）**。

- 字符串形式

```
"author": "JackySummer"
```

- 对象形式：包含必选的 name 属性和可选的 url、email 属性

```json
"author": {
    "name": "JackySummer",
    "email": "xxxxx@qq.com",
    "url": "https://jackylin.vercel.app"
}
```

也可以用下面的字符串简写，npm 会帮我们解析：

```json
"author": "JackySummer <xxxxx@qq.com> (https://jackylin.vercel.app)"
```

### contributors

**贡献者信息**。对象格式数组，对象的内容和`author`一样

```json
"contributors": [{
    "name": "JackySummer",
    "email": "xxxxx@qq.com",
    "url": "https://jackylin.vercel.app"
}]
```

通常对应的，Github 开源项目通常会有 `/.github/CONTRIBUTING.md`贡献指南这个文件，里面也可能有列出一些贡献者简要信息等，如 Vue 的 [CONTRIBUTING.md](https://github.com/vuejs/vue/blob/main/.github/CONTRIBUTING.md)

### scripts

**项目内置的脚本命令**，这一字段下的东西是我比较感兴趣的，通常可以看到项目的启动方式、prettier/eslint 运行脚本，是否用了单元测试等等...

找到项目的`/node_modules/.bin`目录，项目 `npm run xx`的底层调用命令在这里都能找到

> npm 脚本的原理：每当执行 npm run，就会自动新建一个 Shell，在这个 Shell 里面执行指定的脚本命令。因此，只要是 Shell（一般是 Bash）可以运行的命令，就可以写在 npm 脚本里面

比如我们安装了 eslint，`scripts`里面配置：

```
"eslint": "eslint --fix --ext .ts,.tsx ."
```

而不必写成路径的方式

```
"eslint": "./node_modules/.bin/eslint --fix --ext .ts,.tsx .",
```

---

**npm scripts 钩子**

npm 脚本有 pre 和 post 两个钩子，如 运行 npm run install 的时候，分 3 个阶段：

1. 检查 scripts 对象中是否存在 preinstall 命令，如果有，先执行该命令；
2. 检查是否有 install 命令，有的话运行 install 命令，没有的话报错；
3. 检查是否存在 postinstall 命令，如果有，执行 postinstall 命令

npm 默认提供下面这些钩子。

```
prepublish，postpublish
preinstall，postinstall
preuninstall，postuninstall
preversion，postversion
pretest，posttest
prestop，poststop
prestart，poststart
prerestart，postrestart
```

另外，我们自定义的脚本命令也可以加上 pre 和 post 钩子。

下面举例我工作中使用到 npm 钩子的场景：

> 1. 所有项目都由 npm 迁移 pnpm，但有的人没注意或忘了，拿到项目还是使用 npm 安装，也产生了 package-lock.json 文件

解决方案：在 npm install 之前限制住只能使用 pnpm 安装。`.gitignore` 添加 npm 和 yarn 的 lock 文件

```
"scripts": {
    "preinstall": "npx only-allow pnpm"
}
```

`.gitignore`

```
yarn.lock
package-lock.json
```

> 2. 第三方库有 Bug，如何临时紧急修复

这个以前遇过，年代久远了举不出例子来了，找了别人的文章可以了解下：

patch-package 和 postinstall ：[【 非常实用】如何优雅地解决 npm 依赖 bug](https://juejin.cn/post/7102695629641482270)

了解更多 npm scripts 可参考学习：[npm scripts 使用指南](https://www.ruanyifeng.com/blog/2016/10/npm_scripts.html)

---

**建议 script 有约定俗成的规范脚本命令，提高可读性与降低维护成本**

来看下一些开源项目的 `scripts` 配置：

[Vue](https://github.com/vuejs/vue/blob/main/package.json)：

```json
 "scripts": {
    "dev": "rollup -w -c scripts/config.js --environment TARGET:full-dev",
    "dev:cjs": "rollup -w -c scripts/config.js --environment TARGET:runtime-cjs-dev",
    "dev:esm": "rollup -w -c scripts/config.js --environment TARGET:runtime-esm",
    "dev:ssr": "rollup -w -c scripts/config.js --environment TARGET:server-renderer",
    "dev:compiler": "rollup -w -c scripts/config.js --environment TARGET:compiler ",
    "build": "node scripts/build.js",
    "build:ssr": "npm run build -- runtime-cjs,server-renderer",
    "build:types": "rimraf temp && tsc --declaration --emitDeclarationOnly --outDir temp && api-extractor run && api-extractor run -c packages/compiler-sfc/api-extractor.json",
    "test": "npm run ts-check && npm run test:types && npm run test:unit && npm run test:e2e && npm run test:ssr && npm run test:sfc",
    "test:unit": "vitest run test/unit",
    "test:ssr": "npm run build:ssr && vitest run server-renderer",
    "test:sfc": "vitest run compiler-sfc",
    "test:e2e": "npm run build -- full-prod,server-renderer-basic && vitest run test/e2e",
    "test:transition": "karma start test/transition/karma.conf.js",
    "test:types": "npm run build:types && tsc -p ./types/tsconfig.json",
    "format": "prettier --write --parser typescript \"(src|test|packages|types)/**/*.ts\"",
    "ts-check": "tsc -p tsconfig.json --noEmit",
    "ts-check:test": "tsc -p test/tsconfig.json --noEmit",
    "bench:ssr": "npm run build:ssr && node benchmarks/ssr/renderToString.js && node benchmarks/ssr/renderToStream.js",
    "release": "node scripts/release.js",
    "changelog": "conventional-changelog -p angular -i CHANGELOG.md -s"
  },
```

[dayjs](https://github.com/iamkun/dayjs/blob/dev/package.json)：

```json
 "scripts": {
    "test": "TZ=Pacific/Auckland npm run test-tz && TZ=Europe/London npm run test-tz && TZ=America/Whitehorse npm run test-tz && npm run test-tz && jest",
    "test-tz": "date && jest test/timezone.test --coverage=false",
    "lint": "./node_modules/.bin/eslint src/* test/* build/*",
    "prettier": "prettier --write \"docs/**/*.md\"",
    "babel": "cross-env BABEL_ENV=build babel src --out-dir esm --copy-files && node build/esm",
    "build": "cross-env BABEL_ENV=build node build && npm run size",
    "sauce": "npx karma start karma.sauce.conf.js",
    "test:sauce": "npm run sauce -- 0 && npm run sauce -- 1 && npm run sauce -- 2  && npm run sauce -- 3",
    "size": "size-limit && gzip-size dayjs.min.js"
  },
```

### config

**用于添加命令行的环境变量**，即我们脚本在运行时的参数。

```json
"config" : { "port" : "8080" },
"scripts" : { "start" : "node server.js" }
```

在 server.js 脚本就可以引用 config 字段的值

```
http
  .createServer(...)
  .listen(process.env.npm_package_config_port) // 8080
```

执行 npm run start 命令时，这个脚本就可以得到值。

用户可以改变这个值

```
npm config set foo:port 80
```

这个字段没用过，实战中这些配置一般不会直接写在 package.json 文件里面

例子引用自 [config 字段](https://javascript.ruanyifeng.com/nodejs/packagejson.html#toc6)

### engines

**声明项目需要在怎样的 node 环境下运行**，曾遇过比较老旧的项目，甚至可能要降级 node 版本才能跑起来。这时可以给项目添加字段，说明哪个版本下运行能跑起来项目，当然你现在正维护开发的项目也推荐补充下。

```json
"engines": {
    "node": ">=14.16.0",
    "pnpm": ">=6 <7"
},
```

`engines`只是起一个说明的作用，即使版本不符合要求也丝毫不影响安装使用。

但如果真的遇上对 node 版本要求比较严格的项目，就可以在 `.npmrc`文件设置

```
engine-strict = true
```

这时本地 node 版本不匹配的话就会报错 `Unsupported engine`，要求你必须切换`engines`配置的对应版本才能正常安装

### workspaces

**monorepo 项目的工作区配置**，用于在本地的根目录下管理多个子项目，可以自动地在 `npm install` 时将 workspaces 下面的包，软链到根目录的 `node_modules` 中，而不用手动执行 `npm link` 操作

workspaces 字段接收一个数组，数组里可以是文件夹名称或者通配符，通常子项目都会管理在 packages 目录下。比如：

```json
  "workspaces": [
    "packages/*"
  ]
```

[Babel](https://github.com/babel/babel/blob/main/package.json#L78) 的配置：

```json
  "workspaces": [
    "codemods/*",
    "eslint/*",
    "packages/*",
    "test/esm",
    "test/runtime-integration/*",
    "benchmark"
  ],
```

### files

**项目发布时包含的文件**，配置格式为数组，可以指定单独的文件或整个文件夹。

如 Ant Design：

```json
"files": [
    "dist",
    "lib",
    "es"
]
```

声明`files`字段后，当该 npm 包被安装时，安装的是`files`字段指定的内容，从而做到精准控制发布内容，也控制了 npm 包大小。

如果有不想提交的文件，可以在项目根目录中新建一个`.npmignore`文件进行配置，声明不需要提交的文件；写在这里的文件即使 `files` 声明了也不会被提交，这是为了防止垃圾无用文件被推到 npm 上

```json
.vscode
```

### sideEffects

**声明设置哪些模块具有副作用**，让打包工具知道你的模块是否是“纯”的以此更好的 Tree Shaking。

> 首先要知道什么会让一个模块有副作用？

例如修改一个全局变量，发送 API 请求，或导出 CSS，是否重写了原生对象方法，总结起来就是函数可能对外部产生影响的行为

```js
window.foo = 'global foo'
```

---

> Tree Shaking 怎么优化的

Webpack 的 Tree Shaking 机制由 `optimization.usedExports` 和 `sideEffects` 共同承担

1. 通过设置 `usedExports` 为 true，表示模块只导出被使用的成员，配合 terser 删除项目所有模块中未被引用的导出变量

```js
module.exports = {
  optimization: {
    usedExports: true, // 作用于代码语句层面，只导出（export）有使用的变量/方法
  },
}
```

当然这个 webpack 打包生产环境默认开启的

2. 通过 `package.json` 配置的 `sideEffects`，用于标记整个模块的副作用。

`sideEffects` 设为 `false`，表示没有任何模块具有副作用，所有模块都是"纯"的，即打包的时候不管它是否有没有副作用，只要它没有被引用，整个模块/包都会被完整的移除

```
"sideEffects": false
```

也可以使用字符串数组来列出哪些文件具有副作用

如 Ant Design 的`sideEffects`声明如下 ，告知这些文件有副作用，引入后不能被删除

```json
"sideEffects": [
    "dist/*",
    "es/**/style/*",
    "lib/**/style/*",
    "*.less"
],
```

### type

**定义 package.json 文件和及其所在目录根目录中`.js`文件和没有拓展名文件的处理方式**。

默认值为 `commonjs`

```
"type": "commonjs"
```

比如我们在项目根目录下新建两个文件：

```
// test.js
export const foo = 10

// index.js
import { foo } from './test.js'

console.log(foo)
```

然后在控制台运行`node index.js`，这时会报错

```
SyntaxError: Cannot use import statement outside a module
```

因为 Node 的模块化方案采用的是 CommonJS，而代码是 ES 语法，所以报错；

而 Node 在 v13.2.0 已开始正式支持 ES Modules 特性，我们可在 package.json 设置：

```
"type": "module"
```

这样子 Node 就会用 ES 规范进行解析，重新运行就不会报错了

需要注意的是，无论`package.json`中的`type`字段设置为何值，只要文件后缀是 `.mjs`的文件都依然按照 ES 模块规范来处理， `.cjs`的文件都按照 CommonJS 模块规范来处理，不会受 `type`影响

### types/typings

**对外暴露相关的类型**，指定 TypeScript 的类型定义的入口文件，前提是项目是用 TypeScript 写的，用于方便 IDE 识别与智能提示。

```json
"typings": "types/index.d.ts"
```

使用 `types` 或`typings`都可以，作用相同

### main

`main`、`browser`、`module`三个字段都是用于 npm 包的，如果项目不是作为 npm 包发布，这三个字段不需要写。

**main 字段指定加载的入口文件**，指向一个兼容 CommonJS 格式的产出，这个文件名是项目作为 npm 包被打包时配置的，在 browser 和 node 环境中都可以使用。如使用`require`的方式导入该 npm 包时，返回的就是`main`字段指定文件的`module.exports`属性。

当不设置该字段时，默认值是根目录下的`index.js` 文件

```json
"main": "./index.js"
```

```json
// Vue
"main": "dist/vue.runtime.common.js"
```

### browser

**指向支持 UMD 模块的入口文件**，这个字段也会被一些公共 CDN 使用，比如 `unpkg` 及 `jsdelivr`。
也可以直接通过声明 `unpkg` 及 `jsdelivr` 字段来配置入口文件，下面再细说这两个字段。

> UMD：兼容 CommonJS 和 AMD 的模块，既可以在 node 环境中被 `require` 引用，也可以在浏览器中直接用 CDN 被`script`标签 引入

> unpkg 和 jsDelivr 是开源 CDN 服务

main 字段里指定的入口文件在 browser 和 node 环境中都可以使用。如果只想在 web 端使用，禁止在 server 端使用，可以通过 browser 字段指定入口。

```json
"browser": "./browser/index.js"
```

### module

- **module 指向支持 ESM 模块的入口文件**，browser 环境和 node 环境均可使用

```json
"module": "./index.mjs"
```

如果一个项目同时定义了 `main`，`browser` 和 `module`，Webpack 等构建工具打包的时候会根据环境以及不同的模块规范来进行不同的入口文件查找。

```json
// Vue：
"main": "dist/vue.runtime.common.js",
"module": "dist/vue.runtime.esm.js",
```

总结 npm 包 `main`、`module`、`browser`：

- 导出包只在 web 端使用，并且禁止在 server 端使用，使用 `browser`
- 导出包只在 server 端使用，使用 `main`
- 导出 ESM 规范的包，使用 `module`
- 导出包在 web 端和 server 端都允许使用，使用 `module` 和 `main`

还有其他具体情况这里就不展开了

### exports

如果打包工具支持 `exports` 字段，则该字段优先级最高，会忽略 `main/browser/module`的配置。

比如同时使用 `require` 和 `import` 字段定义模块规范入口：

```json
"exports": {
  "require": "./index.js",
  "import": "./index.mjs"
 }
}

// 上面写法还等同于
"exports": {
  // 这里路径声明在根目录下，因为还支持配置包的子路径
  ".": {
    "require": "./index.js",
    "import": "./index.mjs"
  }
 }
}
```

更多 `exports` 文档可看 [NodeJS 文档说明](https://nodejs.org/api/packages.html#exports)

```json
// Vue 的配置：
"exports": {
    ".": {
      "import": {
        "node": "./dist/vue.runtime.mjs",
        "default": "./dist/vue.runtime.esm.js"
      },
      "require": "./dist/vue.runtime.common.js",
      "types": "./types/index.d.ts"
    },
    "./compiler-sfc": {
      "import": "./compiler-sfc/index.mjs",
      "require": "./compiler-sfc/index.js"
    },
    "./dist/*": "./dist/*",
    "./types/*": "./types/*",
    "./package.json": "./package.json"
},
```

### unpkg/jsdelivr

**让 npm 上所有的文件都开启 cdn 服务**。注意文件需要是 UMD 模块规范格式

```
"unpkg": "dist/redux.js",
```

使用`unpkg/jsdelivr`字段可以让 npm 上所有的文件都开启 cdn 服务，比如当我们从 CDN 访问使用 `Redux`时（ [https://unpkg.com/redux](https://unpkg.com/redux)），

会重定向到 [https://unpkg.com/redux@4.2.0/dist/redux.js](https://unpkg.com/redux@4.2.0/dist/redux.js) （取最新`Redux`版本）获取文件

### dependencies

**运行依赖**，项目线上环境下需要用到的依赖。
使用 `npm install xxx` 或 `npm install xxx --save` 时，npm 包就会自动插入到该字段下。

```json
"dependencies": {
  "react": "18.2.0",
  "react-dom": "18.2.0",
  "@ant-design/icons": "4.7.0",
}
```

在安装之前就可以判断 npm 包是否需要在线上运行，如不需要则不要放到该字段下。

在我经历过的来看，建议一些重要且稳定的库锁死版本，防止意外升级导致生产 bug，因为升级库的次版本，也有概率出现较严重 Bug。

```
"antd": "4.21.0",
```

找了篇文章可以看下：[如何管理 npm 版本号：语义化版本策略 SemVer](https://eminoda.github.io/2021/01/29/npm-semver-strategy/)

### devDependencies

**开发依赖**，项目开发环境需要用到而线上运行时不需要的依赖，用于辅助开发。

比如 `webpack`、`eslint`、`jest`、TS `@types/xxx`类型文件等等。

当打包上线时并不需要这些包，所以可以把这些依赖添加到 devDependencies 中，这些依赖依然会在本地指定 npm install 时被安装和管理，但是不会被安装到生产环境中。

使用 `npm install xxx -D` 或者 `npm install xxx --save-dev` 时，npm 包就会自动插入到该字段下。

### peerDependencies

**同伴依赖**，防止包避免重复安装，一般组件库比较常见。

如 Ant Design 的配置：表示我们项目使用 `antd` npm 包还需安装 `react` 和 `react-dom`，且版本都要`>=16.9.0`

```json
"peerDependencies": {
    "react": ">=16.9.0",
    "react-dom": ">=16.9.0"
},
```

### peerDependenciesMeta

**可选的同伴依赖**。

举例：[react-redux package.json](https://github.com/reduxjs/react-redux/blob/master/package.json)

```json
 "peerDependencies": {
    "@types/react": "^16.8 || ^17.0 || ^18.0",
    "@types/react-dom": "^16.8 || ^17.0 || ^18.0",
    "react": "^16.8 || ^17.0 || ^18.0",
    "react-dom": "^16.8 || ^17.0 || ^18.0",
    "react-native": ">=0.59",
    "redux": "^4"
  },
  "peerDependenciesMeta": {
    "@types/react": {
      "optional": true
    },
    "@types/react-dom": {
      "optional": true
    },
    "react-dom": {
      "optional": true
    },
    "react-native": {
      "optional": true
    },
    "redux": {
      "optional": true
    }
  },
```

上述指定了 5 个 npm 包在`peerDependenciesMeta`中，表示都为可选项，所以只安装任意几个可能不会报错，当然也不需要全部安装，因为这里明显看出这是区分 native 和 web 两个环境的，使用的话就任选只在一种环境下。

### bin

**指定各个内部命令对应的可执行文件的位置**，指定了 bin 字段的 npm 包，如果被全局安装，就会被加载到全局环境中，可以通过别名来执行该文件。如带有工具性质的 npm 包。

```
// webpack
"bin": {
    "webpack": "bin/webpack.js"
},

// eslint
"bin": {
    "eslint": "./bin/eslint.js"
},
```

> `node_modules/.bin/`目录下的命令，都可以用 npm run [命令] 的格式运行

拿 eslint 举例，`eslint` 命令对应的可执行文件为 bin 子目录下的 `eslint.js`。npm 会在`node_modules/.bin/`目录下建立符号链接，由于`node_modules/.bin/`目录会在运行时加入系统的 PATH 变量，所以在运行 npm 时，就可以不带路径，直接通过命令来调用这些脚本。

```
"eslint": "./node_modules/bin/eslint.js --fix --ext .ts,.tsx"

// 简写为
"eslint": "eslint --fix --ext .ts,.tsx"
```

### overrides

**重写项目依赖的依赖，及其依赖树下某个依赖的版本号**，进行包的替换，支持任意深度的嵌套。。

这个拿我遇过的问题来解释，比如`ts-ebml`库，我们在项目中已经锁死了版本，但是有次发现 lock 文件更新了，当然它还是保持原来版本不变，但是它的子依赖`matroska`被升级了小版本导致 bug。

由于`matroska`这个库的场景在我们项目只有一处地方，为了解决这个问题，就需要把`matroska`版本锁死，恢复到之前没有 bug 的版本，所以可以用该字段

```json
"overrides": {
  "matroska": "2.2.3"
}
```

此时查找 lock 文件依赖树，会发现 lock 文件全部被安装为我们指定的版本`/matroska/2.2.3`，当然你也可以针对单独的库进行版本重写，如下：

```json
"overrides": {
  "ts-ebml": {
    "matroska": "2.2.3"
  }
}
```

- `pnpm` 同样也是使用 `overrides` 字段
- `yarn` 需要使用 `resolution` 字段

更多可了解： [前端依赖版本重写指南](https://developer.aliyun.com/article/1050105)

## 参考

- [前端工程化基建探索：从内部机制和核心原理了解 npm](https://mp.weixin.qq.com/s/8WKqxJ_CSvwEtKHPcWHmdA)
- [package.json 配置完全解读](https://juejin.cn/post/7161392772665540644)
- [关于前端大管家 package.json，你知道多少？](https://juejin.cn/post/7023539063424548872)

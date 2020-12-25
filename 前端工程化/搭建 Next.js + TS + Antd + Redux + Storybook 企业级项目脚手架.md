# 搭建 Next.js + TS + Antd + Redux + Storybook 企业级项目脚手架

## 前言

⭐ [Nextjs-TS-Antd-Redux-Storybook-Jest-Starter](https://github.com/Jacky-Summer/nextjs-ts-antd-redux-storybook-starter)

之所以有该项目呢，是因为日常可能自己需要练手其他 Next.js 项目，又不想每次都重新配置一遍，但基于强迫症正常企业级项目该有的配置觉得不能少了，于是就想开搞一个通用脚手架模板。

说起 Next.js，8 月份写了一篇文章[手把手带你入门 NextJs（v9.5）](https://juejin.cn/post/6863336367309455373)，主要是因为网上大部分 Next.js 是旧版本 v7.x 的教程，于是写个较新的 9.5 版，没想到 10 月就出了 Next.js 10，措手不及，不过更新部分主要是图片优化等，可以照样看。

该项目也是想把日常工作中我觉得实践比较好的点加进来，也打算根据该项目持续跟进良好规范和最新库版本。当然，到具体业务场景的话脚手架肯定多少需要改，但目标希望能降低修改的成本，起码基本配置得搞好。

该脚手架主要库和版本：

```
Next.js 10.x
React 17.x
TypeScript 4.x
Ant Design 4.x
Styled-components 5.x
Storybook 6.x
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/facf99da9e1645db8b9b33e5cebff0af~tplv-k3u1fbpfcp-watermark.image)

## 初始化 Next.js 模板

```
npx create-next-app nextjs-ts-redux-antd-starter
```

## 添加 TypeScript 支持

根目录下新建`tsconfig.json`文件，此时运行`yarn dev`，会看到它提示我们安装类型库

```
yarn add --dev typescript @types/react @types/node
```

顺便把`@types/react-dom`也装上

安装之后，再运行`yarn dev`, 会在根目录自动生成`next-env.d.ts`文件，且`tsconfig.json`有了默认配置，这里我再对配置稍加改动。

具体可以参考 TS 官网看[配置项](https://www.typescriptlang.org/docs/handbook/compiler-options.html)：

```
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "src/*": [
        "src/*"
      ]
    },
    "target": "es5",
    "module": "esnext",
    "strict": true,
    "allowJs": true, // 允许编译js文件
    "jsx": "preserve", // 在 .tsx文件里支持JSX
    "noEmit": true, // 不输出文件
    "lib": [
      "dom",
      "dom.iterable",
      "esnext"
    ], // TS需要引用的库，即声明文件
    "esModuleInterop": true, // 允许export=导出，由import from导入
    "moduleResolution": "node", // 模块解析策略，ts默认用node的解析策略，即相对的方式导入
    "allowSyntheticDefaultImports": true, // 允许从没有设置默认导出的模块中默认导入
    "isolatedModules": true, // 将每个文件作为单独的模块
    "resolveJsonModule": true, // 允许把json文件当做模块进行解析
    "skipLibCheck": true, // 跳过所有声明文件的类型检查
    "forceConsistentCasingInFileNames": true // 不允许对同一文件使用不一致大小写的引用
  },
  "include": [
    "next-env.d.ts",
    "**/*.ts",
    "**/*.tsx"
  ],
  "exclude": [
    "node_modules",
    ".next",
    "dist"
  ]
}
```

然后清除干净目录，把`styles`, `pages`只留下一个`index.js`即可， 并将`index.js`重命名为`index.tsx`

```javascript
import { NextPage } from 'next'

const Home: NextPage = () => {
  return <div>Hello nextjs-ts-redux-antd-starter</div>
}

export default Home
```

## EditorConfig

作为项目代码风格的统一规范，我们需要借助第三方工具来强制

`.editorconfig` 是跨编辑器维护一致编码风格的配置文件，在 VSCode 中需要安装相应插件 EditorConfig for VS Code，安装完毕之后， 可以通过输入 `Generate .editorcofig` 即可快速生成 `.editorconfig` 文件，也可以自己新建文件。

在`.editorcofig`文件，就可以大家根据不同来设置文件了，比如我的是这样：

```
# http://editorconfig.org
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.md]
trim_trailing_whitespace = false

[Makefile]
indent_style = tab
```

## Prettier

```
yarn add prettier -D
```

同样也需要安装 VSCode 插件`Prettier - Code formatter`

新建文件`.prettierrc`

```
{
  "singleQuote": true,
  "tabWidth": 2,
  "endOfLine": "lf",
  "trailingComma": "all",
  "printWidth": 100,
  "arrowParens": "avoid",
  "semi": false,
  "bracketSpacing": true,
  "overrides": [
    {
      "files": ".prettierrc",
      "options": { "parser": "json" }
    }
  ]
}
```

再添加忽略文件`.prettierignore`

```
**/*.png
**/*.svg
**/*.ico
package.json
lib/
es/
dist/
.next/
coverage/
LICENSE
yarn.lock
yarn-error.log
*.sh
.gitignore
.npmignore
.prettierignore
.DS_Store
.editorconfig
.eslintignore
**/*.yml
```

## ESLint

```
yarn add eslint -D
```

安装完后运行 `npx eslint --init`，运行后有选项，选择如下（自行根据需要）：

- To check syntax, find problems, and enforce code style
- JavaScript modules (import/export)
- React
- TypeScript Yes
- Browser Node
- Use a popular style guide
- Airbnb: https://github.com/airbnb/javascript
- JavaScript
- Would you like to install them now with npm (Yes)

npm 安装后会出现`package-lock.json`，如果你默认想用`yarn.lock`，为了避免冲突就删掉它。

安装自动生成`.eslintrc`文件，还没完，为了写出来的代码更好更符合社区规范，我们再加一些不错的 eslint 插件

- `eslint-plugin-unicorn`：提供了循环依赖检测，文件名大小写风格约束等非常实用的规则集合。
- `eslint-config-prettier`：eslint 和 prettier 混合使用时候，需要修改规则，以防止重复或冲突；该插件即为解决此问题的存在，可以使用它关闭所有可能引起冲突的规则。
- `eslint-plugin-import`：能够正确解析 `.tsx, .ts, .js, .json` 后缀名（还需指定允许的后缀名，添加到 setttings 字段）
- `eslint-import-resolver-alias`: eslint 能识别 alias 别名自定义路径
- `eslint-import-resolver-typescript`：让 `eslint-plugin-import` 能够正确解析 `tsconfig.json` 中的 `paths` 映射，需要安装它。

我的配置如下，`rules`忽略规则自己添加因人而异

```javascript
module.exports = {
  env: {
    browser: true,
    es2021: true,
    node: true,
  },
  extends: [
    'airbnb',
    'plugin:react/recommended',
    'plugin:import/typescript',
    'plugin:@typescript-eslint/recommended',
    'prettier/react',
  ],
  settings: {
    'import/resolver': {
      node: {
        extensions: ['.tsx', '.ts', '.js', '.json'],
      },
      alias: [['src', './src']],
    },
  },
  parser: '@typescript-eslint/parser',
  parserOptions: {
    ecmaFeatures: {
      jsx: true,
    },
    ecmaVersion: 12,
    sourceType: 'module',
  },
  plugins: ['react', '@typescript-eslint', 'react-hooks', 'unicorn'],
  rules: {
    semi: 0,
    indent: 0,
    'react/jsx-filename-extension': 0,
    'react/prop-types': 0,
    'react/jsx-props-no-spreading': 0,

    'jsx-a11y/click-events-have-key-events': 0,
    'jsx-a11y/no-static-element-interactions': 0,
    'jsx-a11y/no-noninteractive-element-interactions': 0,

    'no-use-before-define': 0,
    'no-unused-vars': 0,
    'implicit-arrow-linebreak': 0,
    'consistent-return': 0,
    'arrow-parens': 0,
    'object-curly-newline': 0,
    'operator-linebreak': 0,
    'import/no-extraneous-dependencies': 0,
    'import/extensions': 0,
    'import/no-unresolved': 0,
    'import/prefer-default-export': 0,
  },
}
```

新建文件`.eslintignore`，忽略一些文件的检查

```
/node_modules
/public
/dist
/.next
/coverage
```

## StyleLint

### sass/less/css

- `eslint-config-prettier`: 利用插件禁用与 Prettier 起冲突的规则
- `stylelint-config-rational-order`: 对关联属性进行分组和排序
- `stylelint-declaration-block-no-ignored-properties`: 矛盾样式忽略
- `stylelint-order`：强制你按照某个顺序编写 css

.stylelintrc

```
{
  extends: [
    'stylelint-config-standard',
    'stylelint-config-rational-order',
    'stylelint-config-prettier',
  ],
  plugins: ['stylelint-order', 'stylelint-declaration-block-no-ignored-properties'],
}
```

### styled-components

以上是使用 sass 或 less 可以完全照搬配置的，至于该脚手架我决定采用的 CSS 方案为`styled-components`，stylelint 配置 styled-components 目前有关库尚未实现自动修复，所以`--fix`目前是无效的，且需要安装另外的 stylelint 规则插件

```
yarn add styled-components
yarn add -D @types/styled-components stylelint-processor-styled-components stylelint-config-styled-components
```

.stylelintrc

```
{
  "processors": [
    "stylelint-processor-styled-components"
  ],
  "plugins": [
    "stylelint-order"
  ],
  "extends": [
    "stylelint-config-standard",
    "stylelint-config-styled-components"
  ]
}
```

再新建文件`babel.config.js`

```
{
  "presets": ["next/babel"],
  "plugins": [["styled-components", { "ssr": true }]]
}
```

你可以分`development`,`test`,`production`,对 styled-components 进行区分设置，比如[babel.config.js](https://github.com/Jacky-Summer/nextjs-ts-antd-redux-storybook-starter/blob/master/babel.config.js)

新建文件`pages/_document.tsx`，来自定义 Document 的方式来改写代码。它只有在服务器端渲染的时候才会被调用，主要用来修改服务器端渲染的文档内容，一般用来配合第三方 css-in-js 方案使用。

```
import React from 'react'
import Document, { DocumentContext } from 'next/document'
import { ServerStyleSheet } from 'styled-components'

class MyDocument extends Document {
  static async getInitialProps(ctx: DocumentContext) {
    const sheet = new ServerStyleSheet()
    const originalRenderPage = ctx.renderPage
    try {
      const initialProps = await Document.getInitialProps(ctx)

      ctx.renderPage = () =>
        originalRenderPage({
          enhanceApp: App => props => sheet.collectStyles(<App {...props} />),
        })

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

export default MyDocument
```

## .vscode

在根目录下新建文件夹`.vscode`，在该文件夹下新建文件 settings.json,该文件的配置优先于你自己 VSCode 全局的配置，不会因为团队不同成员的 VSCode 全局配置不同而导致格式化不同。

`settings.json`

```
{
  "search.exclude": {
    "**/node_modules": true,
    "dist": true,
    ".next": true,
    "yarn.lock": true
  },
  "editor.formatOnSave": true,
  "editor.tabSize": 2,
  "eslint.validate": ["javascript", "javascriptreact", "typescript", "typescriptreact"],
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true,
    "source.fixAll.stylelint": true
  }
}
```

## husky 与 lint-staged

每次提交代码都要对代码先进行 lint 和格式化，确保代码风格统一。于是我们安装`husky`来解决这个事情，可我们想每次 lint 格式化的时候，只处理我们修改的代码（暂存区），可以选择`lint-staged`

```
yarn add -D husky lint-staged
```

在`package.json`配置 git commit 钩子操作：

```
 "husky": {
    "hooks": {
      "commit-msg": "commitlint --config .commitlintrc.js -E HUSKY_GIT_PARAMS",
      "pre-commit": "lint-staged && yarn tsc"
    }
  },
  "lint-staged": {
    "*.{tsx,ts,js,jsx}": [
      "stylelint",
      "prettier --write",
      "eslint --fix"
    ],
    "*.{css,less,scss}": [
      "stylelint",
      "prettier --write"
    ],
    "*.{md,json,yaml,yml}": [
      "prettier --write"
    ]
  },
```

`prettier --write`中的`--write`表示将格式化后的代码写到源文件，不加的话会输出文件。

上面`"pre-commit": "lint-staged && yarn tsc"`我还加了`yarn tsc`，ts 检查类型有问题，那当然不给你提交，及早发现错误。

另外需不需要强制`--fix`看个人，因为有人会顾虑强制的话相当于黑盒，你不知道它对你代码做了什么。

## commitlint

我们提交的前文件已经会自动格式化了，接下来要搞搞 commit 提交规范问题。

```
yarn add @commitlint/cli @commitlint/config-conventional -D
```

默认类型 git commit 类型有如下几种，这是 angular 风格的 commitlint 配置，我自己平时习惯这一套规则。

```
[
  'build',
  'ci',
  'chore',
  'docs',
  'feat',
  'fix',
  'perf',
  'refactor',
  'revert',
  'style',
  'test'
];
```

在根目录新建`.commitlintrc.js`：

```
module.exports = {
  extends: ['@commitlint/config-conventional'],
}
```

那如果团队刚来的人没用过也记不住上面那些开头单词怎么办，于是我们弄个命令可以让他自己选择，安装插件

- `cz-conventional-changelog`：是一个适配器，一个符合 Angular 团队规范的 preset

```
yarn add cz-conventional-changelog -D
```

在`package.json`中配置

```
{
    "scripts": {
        "commit": "git-cz"
    },
    "config": {
        "commitizen": {
          "path": "node_modules/cz-conventional-changelog"
        }
    }
}
```

运行`yarn commit`，即出现该页面，供我们选择

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/34ecb85ac98b4bf6b17c7fda0edfb659~tplv-k3u1fbpfcp-watermark.image)

## Redux

基本项目规范配置就差不多了，接下来是做项目的状态管理工具，我这里选择了最经典的 Redux，异步处理选择`redux-saga`

```
yarn add redux react-redux redux-saga
yarn add -D @types/react-redux @types/redux-saga redux-devtools-extension next-redux-wrapper
```

社区还有其他 redux 简化方案，比如使用 [redux-actions](https://github.com/redux-utilities/redux-actions),但该项目维护似乎出现困难，就不加入使用了；还有 dva 等等或者采用其他状态管理库例如 mobx，各位可以自行考虑替换，这里只是给个常用方案。

`src/redux/index.ts`

```javascript
import { createWrapper, MakeStore } from 'next-redux-wrapper'
import { applyMiddleware, createStore } from 'redux'
import createSagaMiddleware from 'redux-saga'
import { composeWithDevTools } from 'redux-devtools-extension/developmentOnly'

import rootReducer from 'src/redux/rootReducers'
import rootSaga from 'src/redux/rootSagas'

const makeStore: MakeStore<Store.RootState> = () => {
  const sagaMiddleware = createSagaMiddleware()
  const store = createStore(rootReducer, composeWithDevTools(applyMiddleware(sagaMiddleware)))
  sagaMiddleware.run(rootSaga)
  return store
}

export const wrapper = createWrapper < Store.RootState > makeStore
```

再新建文件`pages/_app.tsx`引入 redux，覆盖 Next.js 默认的 App

```javascript
import React, { FC } from 'react'
import { AppProps } from 'next/app'
import { wrapper } from 'src/redux'
import Layout from 'src/components/Layout'

const WrappedApp: FC<AppProps> = ({ Component, pageProps }) => (
  <Layout>
    <Component {...pageProps} />
  </Layout>
)

export default wrapper.withRedux(WrappedApp)
```

其他代码及例子演示请看[源代码](https://github.com/Jacky-Summer/nextjs-ts-antd-redux-starter/tree/master/src/redux)

redux 的项目结构有几种，哪种好视乎项目大小和复杂程度选择，该脚手架只是展示一种，按模块来划分 redux 数据结构，并不是说此种方式有多好，具体还是依据项目来调整目录结构。

我做个小 demo 是 saga 请求用户数据，返回并展示在页面上，关于 redux `State`的类型定义，我放在了根目录`types`文件夹里。

当然这也只是一种参照方式，也可以在 redux 目录模块里新建`types`文件放置`State`类型定义。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50d165eeeda34712885de2c32c6e5a6a~tplv-k3u1fbpfcp-watermark.image)

## Ant Desgin 支持

```
yarn add antd
yarn add -D babel-plugin-import
```

`babel.config.js`

```javascript
module.exports = {
  plugins: [
    [
      'import',
      {
        libraryName: 'antd',
        libraryDirectory: 'lib',
        style: 'index.css',
      },
    ],
  ],
}
```

在`src/_app.tsx`中引入 antd 样式：

```
import 'antd/dist/antd.css'
```

## Travis 自动化部署

默认先这样设置了（后面加了 Jest 后再加入脚本`yarn test`）。不解释，不懂的看我这篇文章 [手把手带你入门 Travis 自动化部署](https://juejin.cn/post/6895508265304915982)

```
language: node_js

node_js:
  - "stable"

cache: yarn

install:
  - yarn
script:
  - yarn build
```

## Storybook 搭建组件文档

> Storybook 是在开发模式下与应用程序一起运行的. 它可以帮助您构建 UI 组件,并与应用程序的业务逻辑和上下文隔离开来

```
npx -p @storybook/cli sb init
```

安装完毕，运行即开启

```
yarn storybook
```

然后会有初始一些组件例子，看看就可以删了。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/53fa250176a84173a05177a461176da9~tplv-k3u1fbpfcp-watermark.image)

[How to write stories](https://storybook.js.org/docs/react/writing-stories/introduction) 通过给组件写 stories，可以让我们对整个项目用到的组件有大致了解，比如长什么样等等，还有包括是否 UI 改变，下面会写。

## 国际化

```
yarn add react-i18next i18next
```

当然也可以直接使用 [next-i18next](https://github.com/isaachinman/next-i18next)

`src/i18n/index.ts`

```javascript
import i18n from 'i18next'
import { initReactI18next } from 'react-i18next'
import zhCN from './locales/zh_CN.json'
import enUS from './locales/en_US.json'

i18n.use(initReactI18next).init({
  lng: 'zh_CN',
  fallbackLng: 'zh_CN',
  resources: {
    zh_CN: {
      translation: zhCN,
    },
    en_US: {
      translation: enUS,
    },
  },
  debug: false,
  interpolation: {
    escapeValue: false,
  },
})

export default i18n
```

具体还是直接看代码了，这里就介绍这么多；然后就可以切换语言，把项目用到的一些词语句子都集中到`zh_CN.json`和`en_US.json`等等写。

## Jest 单元测试

为了代码的健壮性，当然是加入单元测试。如果不懂单元测试，请先看我这篇 [一文带你了解 Jest 单元测试](https://juejin.cn/post/6891625768184070152)

```
yarn add -D jest @types/jest eslint-plugin-jest babel-jest @storybook/addon-storyshots
```

配置`.eslintrc.js`

```javascript
module.exports = {
  extends: ['plugin:jest/recommended'],
  plugins: ['jest'],
}
```

在根目录下新建文件`jest.config.js`

```javascript
module.exports = {
  moduleFileExtensions: ['ts', 'tsx', 'js', 'json'],
  testPathIgnorePatterns: ['<rootDir>/dist/', '<rootDir>/node_modules/', '<rootDir>/.next/'],
  moduleNameMapper: {
    '^src(.*)$': '<rootDir>/src$1',
    '^server(.*)$': '<rootDir>/server$1',
    '^pages(.*)$': '<rootDir>/pages$1',
  },
  collectCoverageFrom: [
    './{src,server}/**/*.{ts,tsx,js,jsx}',
    '!**/node_modules/**',
    '!**/dist/**',
    '!**/coverage/**',
    '!**/*.stories.{ts,tsx,js,jsx}',
    '!**/{config,constants,styles,types,__fixtures__}/**',
  ],
  watchPathIgnorePatterns: ['dist'],
}
```

### Storybook 和 Jest 的 Snapshots 结合

Jest 可以生成快照测试（Snapshot），通过 snapshot 变化给我们判断页面元素是否异常，缺失或增加或配置文件是否更改等等。上面安装了 storybook，如果是 react 组件的 snapshot，需要借助其他插件，这里我们转为依靠 storybook 的 stories 生成。

针对 react，Jest 将为虚拟 DOM 拍摄快照，将其转化为 json 数据，在下一次运行时比对两张快照是否有偏差。

```
yarn add -D @storybook/addon-storyshots
```

在根目录新建`jest.config.js`,针对 snapshot 的配置如下，其它配置按项目配置了，参考 [jest.config.js](https://github.com/Jacky-Summer/nextjs-ts-antd-redux-storybook-starter/blob/master/jest.config.js)

```javascript
module.exports = {
  transform: {
    '^.+\\.stories\\.[tj]sx?$': '@storybook/addon-storyshots/injectFileName',
    '^.+\\.[tj]sx?$': 'babel-jest',
  },
}
```

新建文件`src/__tests__/storyshot.test.ts`

```typescript
import initStoryshots, { multiSnapshotWithOptions } from '@storybook/addon-storyshots'

initStoryshots({
  test: multiSnapshotWithOptions(),
})
```

之后组件中有写`stories`的地方，使用`yarn jest`，除了运行测试，也会自动为`*.stories.tsx`比对/生成 snapshot。

对于生成的 snapshot 你会看到
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35bdab0b215b44b492fa58d36e0f7200~tplv-k3u1fbpfcp-watermark.image)

比如我写了`Footer`组件的，只有 HTML 标签和对应属性，这样检测还不够，因为不知道 css 类的属性做了什么改变，由于我用的 css 方案是`styled-components`，所以需要再进行配置:

```
yarn add -D jest-specific-snapshot jest-styled-components
```

配置`src/__tests__/storyshot.test.ts`

```javascript
import initStoryshots, { multiSnapshotWithOptions } from '@storybook/addon-storyshots'
import 'jest-styled-components'
import { addSerializer } from 'jest-specific-snapshot'
import styleSheetSerializer from 'jest-styled-components/src/styleSheetSerializer'

addSerializer(styleSheetSerializer)

initStoryshots({
  test: multiSnapshotWithOptions(),
})
```

再来看现在的 snapshot，已经有了一堆样式可以参考对比了，这样细微组件样式修改都可以被捕捉到了；
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/65ec364160d047ee919968fdeade6024~tplv-k3u1fbpfcp-watermark.image)

### Enzyme

Jest 可以对函数，类等等有充足的 API 来测试，但对于 React 组件，想详细进行测试，则需要安装其他插件来支持，如`react-test-library`和`enzyme`，这里我就选我用过相对多一点的 [enzyme](https://enzymejs.github.io/enzyme/) （出自 Airbnb 公司），同时需要安装它的适配器。

这里由于 React 已经升级到 17 版本了，但是 enzyme 官方适配器还没有升级到对应 17 版本的，有些测试方法可能会报错，所以暂时使用目前 Github 使用较多的代替版本的这个库`@wojtekmaj/enzyme-adapter-react-17`，等官方更新了再替换。这个只是供测试用，不会影响到线上环境，只要 enzyme 自带所有方法能按预期运行不报错就行，这样就能好好写我们的测试用例了。

```
yarn add enzyme @wojtekmaj/enzyme-adapter-react-17 -D
```

在根目录新建文件`jest.setup.ts`

```
import Enzyme from 'enzyme'
import Adapter from '@wojtekmaj/enzyme-adapter-react-17'

Enzyme.configure({ adapter: new Adapter() })
```

同时在`jest.config.js`中导入

```
module.exports = {
  setupFiles: ['<rootDir>/jest.setup.ts'],
}
```

通常情况，测试 React 组件是意义不大的，比较需要测试的就是比如 UI 组件，用得较多的通用组件，还有一些组件如一改动全身的容易有 bug 行为的来针对测试。

## 生成 CHANGELOG 和自动化版本管理

这里我使用`standard-version`。

```
yarn add -D standard-version
```

> standard-version 是一款遵循语义化版本（ semver）和 commit message 标准规范 的版本和 changlog 自动化工具。

```
"bump-version": "standard-version --skip.commit --skip.tag"
```

运行`yarn bump-version`后，会发现 package.json 的版本号变了（前提你有了 feat 或 fix 等等 commit 的改动），还有自动生成 CAHNGEALOG.md，这些都可以自定义配置

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e6493286d5c444e88f11320dfc747e0~tplv-k3u1fbpfcp-watermark.image)

## Github 打 tag 版本

点击 Github 项目页面右边创建 release
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c0c6529a09bd4eeb8b90e7172054ffec~tplv-k3u1fbpfcp-watermark.image)

然后填入版本号，详细信息我把 CHANGELOG.md 的内容直接搬过来，然后按`Publish release`就可以了。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/768d72503b254cf5b2b8b1d33d472c5a~tplv-k3u1fbpfcp-watermark.image)

## 完善 script 命令

在`package.json`中

```
"scripts": {
  "dev": "next dev",
  "build": "next build",
  "start": "next start",
  "commit": "git-cz",
  "test": "jest",
  "coverage": "yarn jest --coverage",
  "lint": "yarn lint:eslint && yarn lint:css",
  "lint:eslint": "eslint --ext js,jsx,ts,tsx .",
  "lint:css": "stylelint **/*.{ts,tsx}",
  "prettier": "prettier --write \"**/*.{js,jsx,tsx,ts,less,md,json}\"",
  "tsc:client": "tsc --noEmit -p tsconfig.json",
  "storybook": "start-storybook -p 6006",
  "build-storybook": "build-storybook -o ./dist_storybook",
  "bump-version": "standard-version --skip.commit --skip.tag"
},
```

## LICENSE

添加个开源协议，我选择宽松的 MIT 协议

```
MIT License

Copyright (c) 2020 nextjs-ts-redux-antd-starter

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## 运行项目

如上演示还略过一些细节其他配置，需要详细的就看[源码](https://github.com/Jacky-Summer/nextjs-ts-antd-redux-storybook-starter)吧。

整个脚手架我是不打算加入太多东西的，如下图所示，毕竟做为模板脚手架，加太多功能反而要用的时候要删除一大堆麻烦，因为做的不是某类型业务网站，有一些只能尽量有个 Demo 参考就行。所以我会尽量保持简洁，之后维护我会倾向于 Next.js 配置和前端工程化及性能优化的角度进行完善，然后就是一些通用的函数和功能。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8071fa608e01412c878c3dcc3e07a021~tplv-k3u1fbpfcp-watermark.image)

## 结语

脚手架到这里就完了？还没有，还有很多没加，比如整理 Next.js 的 config 配置，优化 SEO，发布到线上网站和 npm，一些兼容，特殊页面处理，响应式等等。

当然，在我写这篇文章时的脚手架多少也有写的不好或不完善的地方，因为刚起步，所以该脚手架会持续维护，把工作实践和学习到的最佳实践运行到该项目里，不断保持更新，敬请关注，欢迎 star 🌟🌟🌟 [https://github.com/Jacky-Summer/nextjs-ts-antd-redux-storybook-starter](https://github.com/Jacky-Summer/nextjs-ts-antd-redux-storybook-starter)

<br>

- ps：
  - [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)
  - [monki-ui](https://github.com/Jacky-Summer/monki-ui)：基于 React + TypeScript + Dumi + Jest + Enzyme 开发 UI 组件库，未来会写相关文章

觉得不错的话赏个 star，给我持续创作的动力吧！下次继续...

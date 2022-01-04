## 前言

Vite 出来好一段时间了，最开始支持 Vue，而现在已经不受框架限制了。而 Vite 解决的是对于项目每次启动与打包构建等待时间长的问题，Vite 就是解决了该问题，提高开发效率与体验，本文作个简单的学习记录。

## 初始化

通过 Vite 官方命令行选项直接指定项目名称和想要使用的模板。例如，要构建一个 Vite + TypeScript 项目

```
# npm 6.x
npm init @vitejs/app vite-react-ts-antd-starter --template react-ts

# npm 7+, 需要额外的双横线：
npm init @vitejs/app vite-react-ts-antd-starter -- --template react-ts
```

创建完安装依赖后，运行项目如图：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f07b8842060749629779cc6a2962fa67~tplv-k3u1fbpfcp-watermark.image?)

## 配置路由

```
npm i react-router-dom@5.3.0
```

由于 v6 目前试用 ts 提示等有一些问题，避免讲解复杂还是直接简单点用 v5 版本，用法不变。

首先新建三个页面文件，在 `src/pages`文件夹下新建三个页面

```tsx
// home
const Home = () => {
  return <div>home page</div>
}

export default Home

// about
const About = () => {
  return <div>about page</div>
}

export default About

// not found
const NotFound = () => {
  return <div>404 page</div>
}

export default NotFound
```

在 `src/config`目录下新建文件 `routes.ts`

```ts
import { lazy } from 'react'

const Home = lazy(() => import('../pages/home'))
const About = lazy(() => import('../pages/about'))
const NotFound = lazy(() => import('../pages/not_found'))

const routes = [
  {
    path: '/',
    exact: true,
    component: Home,
    name: 'Home',
  },
  {
    path: '/home',
    component: Home,
    name: 'Home',
  },
  {
    path: '/about',
    component: About,
    name: 'About',
  },
  {
    path: '/404',
    component: NotFound,
  },
]
export default routes
```

新建文件`src/router.tsx`，路由文件入口

```tsx
import { Suspense } from 'react'
import { Route, Switch } from 'react-router-dom'
import routes from './config/routes'

const Router = () => {
  const myRoutes = routes.map((item) => {
    return <Route key={item.path} {...item} component={item.component} />
  })
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Switch>{myRoutes}</Switch>
    </Suspense>
  )
}

export default Router
```

App.tsx

```tsx
import Router from './router'
// 这里我封装了，其实就使用react-router-dom的Link
import { Link } from './components'

function App() {
  return (
    <div className="App">
      <Link to="/">Home Page</Link>
      <Link to="/about">About Page</Link>
      <Router />
    </div>
  )
}

export default App
```

进入`http://localhost:3000`，此时就可以切换路由了

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe4e5541f73a4ca48a05a2887c7e32ab~tplv-k3u1fbpfcp-watermark.image?)

## 支持 Antd

> 在写该文章的时候，antd 最新版本为 4.18.1，使用 vite 打包会有错误，然后回退到 antd 的 4.17.1 就可以了，具体见 https://github.com/ant-design/ant-design/issues/33449

```
 npm i antd@4.17.1 @ant-design/icons
```

在 App.tsx 引入 antd 的按钮组件：

```tsx
import { Button } from 'antd'

// ...
;<Button type="primary">按钮</Button>
```

antd 使用的是 less，这时还需要支持 Less

```
npm i less less-loader -D
```

我们还要对 antd 按需引入，安装插件

```
npm i vite-plugin-imp -D
```

`vite.config.ts` 的配置如下：

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import vitePluginImp from 'vite-plugin-imp'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [
    react(),
    vitePluginImp({
      optimize: true,
      libList: [
        {
          libName: 'antd',
          libDirectory: 'es',
          style: (name) => `antd/es/${name}/style`,
        },
      ],
    }),
  ],
  css: {
    preprocessorOptions: {
      less: {
        javascriptEnabled: true, // 支持内联 JavaScript
      },
    },
  },
})
```

此时查看页面，确实单独引入了按钮的样式组件

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/67bd36094ff24e598e96cbe95dc6413e~tplv-k3u1fbpfcp-watermark.image?)

这样页面就正常显示出 antd 的按钮组件了

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e096ad9611b49f8a1cb10aa63ea7bd7~tplv-k3u1fbpfcp-watermark.image?)

## alias 别名设置

这个同 webpack 配置差不多，在 vite.config.js

```ts
import path from 'path'

export default defineConfig({
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
})
```

改一下 App.tsx 的 Link 组件引入，试验一下

```
- import { Link } from './components'
+ import { Link } from '@/components'
```

此时编辑器会看到红色警告：`Cannot find module '@/components' or its corresponding type declarations.`，是因为我们别名没有在 tsconfig.json 里面配置，修改：

```json
"compilerOptions": {
  "paths":{
    "@/*": ["src/*"],
   },
}
```

## eslint 与 Prettier 配置

```
npm i -D @typescript-eslint/parser eslint eslint-plugin-standard @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

`.eslintrc.js`文件参考：

```js
module.exports = {
  extends: ['eslint:recommended', 'plugin:react/recommended'],
  env: {
    browser: true,
    commonjs: true,
    es6: true,
  },
  parser: '@typescript-eslint/parser',
  parserOptions: {
    ecmaFeatures: {
      jsx: true,
      modules: true,
    },
    sourceType: 'module',
    ecmaVersion: 6,
  },
  plugins: ['react', 'standard', '@typescript-eslint'],
  settings: {
    'import/resolver': {
      node: {
        extensions: ['.tsx', '.ts', '.js', '.json'],
      },
      alias: [['@', './src']],
    },
  },
  rules: {
    semi: 0,
    indent: 0,
    'react/jsx-filename-extension': 0,
    'react/prop-types': 0,
    'react/jsx-props-no-spreading': 0,

    'jsx-a11y/click-events-have-key-events': 0,
    'jsx-a11y/no-static-element-interactions': 0,
    'jsx-a11y/no-noninteractive-element-interactions': 0,
    'jsx-a11y/anchor-is-valid': 0,

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

    '@typescript-eslint/ban-ts-comment': 0,
    '@typescript-eslint/no-var-requires': 0,
  },
}
```

Prettier 配置

```
npm i -D prettier
```

.prettierrc

```
{
  "singleQuote": true,
  "tabWidth": 2,
  "endOfLine": "lf",
  "trailingComma": "all",
  "printWidth": 100,
  "arrowParens": "avoid",
  "semi": false,
  "bracketSpacing": true
}
```

## 环境变量

新增`.env`，`.env.prod`文件，当使用自定义环境时变量要以`VITE`为前缀

```
.env                # 所有情况下都会加载
.env.[mode]         # 只在指定模式下加载
```

`.env`

```
NODE_ENV=development
VITE_APP_NAME=dev-name
```

`.env.prod`

```
NODE_ENV=production
VITE_APP_NAME=prod-name
```

### 获取环境变量

Vite 在`import.meta.env`对象上暴露环境变量。修改 `App.tsx`，直接打印：

```js
console.log('import.meta.env', import.meta.env)
```

重启运行 npm run dev

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f8d42a193ea94e8caf49ad31a109d793~tplv-k3u1fbpfcp-watermark.image?)

### TS 提示环境变量

在 src 目录下新建`env.d.ts`，接着按下面这样增加  `ImportMetaEnv`  的定义：

```ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_APP_NAME: string
  // 更多环境变量...
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

然后 `import.meta.env.VITE_APP_NAME`等这些自定义的环境变量就会有提示了

## 结语

现在 Vite 官方虽然支持了 React，但对于 React 生态来说完整的官方支持还有待完善，所以对于公司 webpack 项目开发环境迁移 Vite 还是持保守态度，待 Vite 继续发展，后续再慢慢跟进和学习再做升级。

本文项目地址：[vite-react-ts-antd-starter](https://github.com/Jacky-Summer/vite-react-ts-antd-starter)

参考文章

- [Vite 2.0 + React + Ant Design 4.0 搭建开发环境](https://juejin.cn/post/6938671679153373214)

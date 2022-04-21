# lerna 与 yarn 上手笔记

### 安装

全局安装 lerna

```
npm i lerna -g
```

### Lerna 使用

### lerna 命令

- `lerna init`: 初始化项目
- `lerna bootstrap`: 自动构建项目
- `lerna ls`: 列出当前项目所有包
- `lerna changed`: 检查自上次发布以来哪些软件包已经更新
- `lerna clean`: 清理`node_modules`文件夹
- `lerna add`: 添加依赖（类似 npm install)
- `lerna publish`: 发版
- `lerna run`: 在包含该脚本的包中运行 npm 脚本

### 初始化仓库

创建一个 git 仓库，在该目录下执行 `lerna init`

```
cd lerna-example
lerna init
```

上面初始化的时候也可以指定模式，比如使用独立模式

```
lerna init --independent
```

也可以在 lerna.json 手动配置独立模式

```json
"version": "independent",
```

在 Lerna 中，有两种模式：

1.  **固定(fixed)模式**：所有  `package`  的版本号保持一致，每次更新发包都是全量的
1.  **独立(independent)模式**：每个  `package`  版本号各自独立，互不影响，每次更新按需发包

看到项目生成如下结构：

```
lerna-example
├── packages
├── lerna.json
└── package.json
```

### 创建 packages

这里我往 packages 文件夹建了两个简单的仓库，一个安装了 rollup 用来打包，一个使用了 dumi 脚手架进行初始化

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ecd395c344d4d7ab818c937d2164aec~tplv-k3u1fbpfcp-watermark.image?)

### 配合 yarn 使用

`lerna`默认使用`npm`作为安装依赖包工具，我们也使用 yarn， yarn 提供了 workspaces 的功能，从更底层的地方提供了依赖提升。

依赖提升即是指把所有项目 npm 依赖文件都提升到项目根目录下，避免相同依赖包在不同项目安装多次

搭配 yarn 使用，一般是用 yarn 来处理依赖问题，用 lerna 用来处理发布问题

> 使用 lerna + yarn workspace 时，lerna 的--hoist 会被禁用，我们直接使用 yarn install 也可实现相同功能

只需在 lerna.json 添加如下配置代码即可：

```json
"npmClient": "yarn", // 依赖管理工具，默认为npm
"useWorkspaces": true // 开启workspaces模式
```

还要配置根目录的 package.json

```json
"workspaces": [
    "packages/*"
]
```

配置完把两个子项目的 node_modules 删掉，这时可以借助 `lerna clean` 命令删除

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a117e9f51daf4068b8f58e777453d168~tplv-k3u1fbpfcp-watermark.image?)

这次我们直接在根目录安装 packages 下包的的依赖，直接在根目录运行`yarn` 即可。此外，还可以单独为某个包安装

- 为某个 package 添加、删除依赖 `yarn workspace <package-name> [add|remove] <library> [-D|-S]`

举个例子，根目录下运行，就针对该包单独安装了 dayjs

```
yarn workspace test-package-utils add dayjs
```

安装完成如下图(packages 目录下包的 node_modules 只包含了可执行文件，相关依赖被提到外层的 node_modules)：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21d7f748fef140d4b905dc29bab4e513~tplv-k3u1fbpfcp-watermark.image?)

### 版本管理

在根目录下执行 `lerna publish`

> lerna publish 不会发布 package.json 中 private 设置为 true 的包

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35ee622da0ec41eea4f3082074c6484f~tplv-k3u1fbpfcp-watermark.image?)

由于两个包都是新初始化，所以两个包都有变更，由于设置了独立模式所以两个包的版本单独提示选择了，选完

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ffc282e57d044ae6ac6e85d54ff22ecf~tplv-k3u1fbpfcp-watermark.image?)

至于发包前需要 npm 登录`npm login`，这个我就不细说了，大家可以查阅多试试

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c28e7ca864ff406e896efdcb0b4571bf~tplv-k3u1fbpfcp-watermark.image?)

## 参考文章

- [几分钟了解前端 Monorepo - Lerna 的使用](https://juejin.cn/post/7064118504982577160)
- [# 基于 Lerna 实现 Monorepo 项目管理](https://juejin.cn/post/7030207667457130527)

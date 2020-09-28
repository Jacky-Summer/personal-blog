# 动手搞懂 HTTP 缓存机制

## 前言

HTTP 缓存策略相信大家都有一定的了解，但自己讲的话也是大概模糊的理解。于是，又回到重新归纳的环节了，这次，搞懂它！

## 浏览器缓存的作用

1. 缓存可以减少冗余的数据传输，节省了网络带宽，从而更快的加载页面以及减少用户的流量消耗
2. 缓存降低了服务器的要求，从而让服务器更快的响应。

## 概述

浏览器缓存策略分为两种：**强缓存**和**协商缓存**。一看到强缓存，就可能联想到"弱缓存"，这个"强"实际形容得不太恰当，其实强缓存指的是新鲜的响应，而协商缓存指的是陈旧的响应（已过期），而要复用陈旧的响应就得使用条件请求（If-xxx）进行缓存验证，我们下面会讲。

## 基本原理

1. 浏览器在加载资源时，根据请求头的 Expires 和 Cache-Control 判断是否命中强缓存，是则直接从缓存读取资源，不会发请求到服务器。
2. 如果没有命中强缓存，浏览器一定会发送一个请求到服务器，通过 Last-Modified 和 ETag 验证资源是否命中协商缓存，如果命中，则返回 304 读取缓存资源
3. 如果前面两者都没有命中，直接从服务器请求加载资源

## 缓存的文件到哪里去

一般看到的两种：memory cache（内存缓存） 和 disk cache（硬盘缓存）

- **200 from memory cache**：它是将资源文件缓存到内存中，而是直接从内存中读取数据。但该方式退出进程时数据会被清除，如关闭浏览器；一般会将将脚本、字体、图片会存储到内存缓存中。

- **200 from disk cache**：它是将资源文件缓存到硬盘中。关闭浏览器后，数据依然存在；一般会将非脚本的存放在硬盘中，比如 css。

优先访问 memory cache，其次是 disk cache，最后是请求网络资源；且内存读取速度比硬盘读取速度快，但也不能把所有数据放在内存中，因为内存也是有限的。

## 通过 express 搭建服务模拟演示

```javascript
const path = require('path')
const fs = require('fs')
const express = require('express')
const app = express()

app.get('/', (req, res) => {
  res.send(`<!DOCTYPE html>
  <html lang="en">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Init Demo</title>
  </head>
  <body>
    Hello Init Demo
    <script src="/test.js"></script>
  </body>
  </html>`)
})

app.get('/test.js', (req, res) => {
  let sourcePath = path.resolve(__dirname, '../public/test.js')
  let result = fs.readFileSync(sourcePath)
  res.end(result)
})

app.listen(3000, () => {
  console.log('listening on 3000')
})
```

如上代码，当进入`http://localhost:3000/`，请求过程如下：

- 浏览器请求静态资源 `test.js`
- 服务器读取磁盘文件 `test.js`，返回给浏览器
- 浏览器再次请求，服务器又重新读取磁盘文件 `test.js`，返给浏览器。

多次刷新网页，重新进入，可以看到每次都重新发了`test.js`的请求，这种方式下没有进行任何缓存。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa0b4111b21043629107fb09c50533d8~tplv-k3u1fbpfcp-zoom-1.image)

## 强缓存

强缓存通过 Expires 和 Cache-Control 两种响应头实现

原理：

> 浏览器在加载资源的时候，会先根据本地缓存资源的 header 中的信息(Expires 和 Cache-Control)来判断是否需要强制缓存。如果命中的话，则会直接使用缓存中的资源。否则的话，会继续向服务器发送请求。

### Expires

Expires 是 HTTP1.0 的规范，表示该资源过期时间，它描述的是一个绝对时间（值为 GMT 时间，即格林尼治时间），由服务器返回。

如果浏览器端当前时间小于过期时间，则直接使用缓存数据。

```javascript
app.get('/test.js', (req, res) => {
  let sourcePath = path.resolve(__dirname, '../public/test.js')
  let result = fs.readFileSync(sourcePath)
  res.setHeader(
    'Expires',
    moment().utc().add(1, 'm').format('ddd, DD MMM YYYY HH:mm:ss') + ' GMT' // 设置1分钟后过期
  )
  res.end(result)
})
```

上面代码我们添加了 Expires 响应头，第一次向服务器发起请求后，同时返回过期时间和文件；当再次刷新时，就会看到文件是直接从缓存(memory cache)中读取的；当 1 分钟过后过期了，又会重新请求了。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/184fe17b6c1444b58214d8d20e33b1e5~tplv-k3u1fbpfcp-zoom-1.image)

这种方式受限于本地时间，服务器的时间和客户端的时间不一样的情况下（比如浏览器设置了很后的时间，则一直处于过期状态），可能会造成缓存失效。而且过期后，不管文件有没有发生变化，服务器都会再次读取文件返回给浏览器。

Expires 是 HTTP 1.0 的东西，现在默认浏览器大部分使用 HTTP 1.1，它的作用基本忽略，因为会靠`Cache-Control`作为主要判断依据。

### Cache-Control

知道了 `Expires`的缺点后，在 HTTP 1.1 版开始，就加入了`Cache-Control`来替代，它是利用`max-age`判断缓存时间的，以秒为单位，它的优先级高于 `Expires`，表示的是相对时间。也就是说，如果`max-age`和`Expires`同时存在，则被 Cache-Control 的 max-age 覆盖。

```javascript
app.get('/test.js', (req, res) => {
  let sourcePath = path.resolve(__dirname, '../public/test.js')
  let result = fs.readFileSync(sourcePath)
  res.setHeader('Cache-Control', 'max-age=60') // 设置相对时间-60秒过期
  res.end(result)
})
```

同样第一次请求后再次请求，就可以看到是命中协商缓存了。
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/60cdeeb8bdd34665975bfe90f0a8cd0a~tplv-k3u1fbpfcp-zoom-1.image)

除了该字段，还有其他字段可设置：

#### public

表示可以被浏览器和代理服务器缓存，代理服务器一般可用 nginx 来做

#### private

只让客户端可以缓存该资源；代理服务器不缓存

#### no-cache

跳过设置强缓存，但是不妨碍设置协商缓存；一般如果你做了强缓存，只有在强缓存失效了才走协商缓存的，设置了 no-cache 就不会走强缓存了，每次请求都回询问服务端。

#### no-store

禁止使用缓存，每一次都要重新请求数据。

比如我设置禁止缓存，再重复上面操作，就每次都向服务器请求了

```javascript
res.setHeader('Cache-Control', 'no-store, max-age=60') // 禁止缓存
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c6b694c980734c649458db32c5966846~tplv-k3u1fbpfcp-zoom-1.image)

## 协商缓存

强缓存的缺点就是每次都根据时间来判断是否过期，但如果到了过期时间后，文件没有改动，再次去获取就有点浪费服务器的资源了，因此有了协商缓存。

当浏览器对某个资源的请求没有命中强缓存，就会发一个请求到服务器，验证协商缓存是否命中，如果协商缓存命中，请求响应返回的 http 状态为 304 告诉浏览器读取换出，并且会显示一个 Not Modified 的字符串；如果未命中，则返回请求的资源。

协商缓存是利用的是`Last-Modified/If-Modified-Since`和`ETag/If-None-Match`这两对标识来管理的

原理：

> 客户端向服务器端发出请求，服务端会检测是否有对应的标识，如果没有对应的标识，服务器端会返回一个对应的标识给客户端，客户端下次再次请求的时候，把该标识带过去，然后服务器端会验证该标识，如果验证通过了，则会响应 304，告诉浏览器读取缓存。如果标识没有通过，则返回请求的资源。

### Last-Modified

过程如下：

- 浏览器请求资源，服务器每次返回文件的同时，返回`Last-Modified`(最后的修改时间)到 Header 中
- 当浏览器的缓存文件过期后，浏览器带上请求头`If-Modified-Since`（值为上一次的`Last-Modified`）请求服务器
- 服务器比较请求头的`If-Modified-Since`和文件上次修改时间一样，则命中缓存返回 304；如果不一致就返回 200 响应和文件的内容还有更新`Last-Modified`的值，以此往复。

```javascript
app.get('/test.js', (req, res) => {
  let sourcePath = path.resolve(__dirname, '../public/test.js')
  let result = fs.readFileSync(sourcePath)
  let status = fs.statSync(sourcePath)
  let lastModified = status.mtime.toUTCString()
  if (lastModified === req.headers['if-modified-since']) {
    res.writeHead(304, 'Not Modified')
    res.end()
  } else {
    res.setHeader('Cache-Control', 'max-age=1') // 设置1秒后过期以方便我们马上能用Last-Modified判断
    res.setHeader('Last-Modified', lastModified)
    res.writeHead(200, 'OK')
    res.end(result)
  }
})
```

第一次请求如下：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/387b188f62744390a631aecb9cb96aa9~tplv-k3u1fbpfcp-zoom-1.image)

1 秒过期后，再次进行多次请求，发现返回的都是 304，命中协商缓存：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8ae57f2487a241d799dcb136b16854cc~tplv-k3u1fbpfcp-zoom-1.image)

在当中我遇到了个小问题，还是报的 200 响应，请求头里有条警告信息：

```
Provisional headers are shown. Disable cache to see full headers.
```

可能是插件原因导致，于是我开了无痕模式就解决了（如果各位有其他原因解释和解决方法可以留言告诉我）

### ETag

`Last-Modified` 也有它的缺点，比如修改时间是 GMT 时间，只能精确到秒，如果文件在 1 秒内有多次改动，服务器并不知道文件有改动，浏览器拿不到最新的文件。而且如果文件被修改后又撤销修改了，内容还是保持原样，但是最后修改时间变了，也要重新请求。也有可能存在服务器没有准确获取文件修改时间，或与代理服务器时间不一致的情况。

为了解决文件修改时间不精确带来的问题，服务器和浏览器再次协商，这次不返回时间，返回文件的唯一标识`ETag`。只有当文件内容改变时，`ETag`才改变。`ETag`的优先级高于`Last-Modified`。

过程如下:

- 浏览器请求资源，服务器每次返回文件的同时带上文件的唯一标识 ETag
- 当浏览器的缓存文件过期后，浏览器带上请求头`If-None-Match`（值为上一次的`ETag`）请求服务器
- 服务器比较请求头的`If-None-Match`和文件的 ETag 一样，则命中缓存返回 304；如果不一致就返回 200 响应和文件的内容还有更新`ETag`的值，以此往复。

```javascript
const md5 = require('md5')

app.get('/test.js', (req, res) => {
  let sourcePath = path.resolve(__dirname, '../public/test.js')
  let result = fs.readFileSync(sourcePath)
  let etag = md5(result)

  if (req.headers['if-none-match'] === etag) {
    res.writeHead(304, 'Not Modified')
    res.end()
  } else {
    res.setHeader('ETag', etag)
    res.writeHead(200, 'OK')
    res.end(result)
  }
})
```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/06e9be1e391a48a9841242a0fdc3a4e6~tplv-k3u1fbpfcp-zoom-1.image)

再次请求时：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d8a2199d885748ee88b6023b7c4f8b5b~tplv-k3u1fbpfcp-zoom-1.image)

- HTTP 中并没有指定如何生成`ETag`, 实际项目中生成哈希(hash)是比较常见的做法。

不过`ETag`每次服务端生成都需要进行读写操作，而`Last-Modified`只需要读取操作，`ETag`生成计算的消耗更大些。

## 缓存的优先级

首先明确的是强缓存的优先级高于协商缓存

在 HTTP1.0 的时候还有个`Pragma`字段，也属于强缓存，当该字段值为 no-cache 的时候，会告诉浏览器不要对该资源缓存，即每次都得向服务器发一次请求才行，它的优先级高于`Cache-Control`

一般我们现在就不会用它了，如果想见到`Pragma`的话，在 Chrome 的 devtools 中启用 disable cache 时或者按 `Ctrl + F5`强制刷新，就会在请求的 Request Header 上看到。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/78b4c599e5894dcd8b2505cc410de2f5~tplv-k3u1fbpfcp-zoom-1.image)

最后优先级为：

> Pragma > Cache-Control > Expires > ETag > Last-Modified

## 如何清除缓存

浏览器默认会缓存图片，css 和 js 等静态资源，有时在开发环境下可能会因为强缓存导致资源没有及时更新而看不到最新的效果，可以用如下几种方式：

1. 直接 Ctrl+F5 强制刷新（直接 F5 的话会跳过强缓存规则，直接走协商缓存）
2. 谷歌浏览器可以在 Network 里面选中`Disable cache`
3. 给资源文件加一个时间戳
4. 其他方式设置如 webpack

<br>

案例 github 代码：[http-cache-demo](https://github.com/Jacky-Summer/http-cache-demo)

## 参考

- [理解 http 浏览器的协商缓存和强制缓存](https://www.cnblogs.com/tugenhua0707/p/10807289.html)
- [前端也要懂 Http 缓存机制](https://juejin.im/post/6844903655619969038)

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，鼓励我继续写作吧~

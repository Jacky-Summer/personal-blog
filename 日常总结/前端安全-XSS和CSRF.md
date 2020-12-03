## XSS

### 概述

XSS（Cross Site Script），指**跨站脚本攻击**。原本缩写为 CSS，但因为与层叠样式表缩写（CSS）重名要做区分，所以改为 XSS。

### 攻击方式

攻击者通过向网站页面注入恶意脚本（一般是 JavaScript），通过恶意脚本对客户端网页进行篡改，达到窃取信息等目的，本质是数据被当作程序执行。

### XSS 的注入点

- HTML 的节点内容或属性，存在读取可输入数据
- javascript 代码，存在由后台注入的变量或用户输入的信息
- 富文本

### XSS 危害

- 通过 document.cookie 盗取 cookie
- 使用 js 或 css 破坏页面正常的结构与样式
- 流量劫持（通过访问某段具有 window.location.href 定位到其他页面:`<script>window.location.href="www.baidu.com";</script>`）
- Dos 攻击：利用合理的客户端请求来占用过多的服务器资源，从而使合法用户无法得到服务器响应
- 利用 iframe、frame、XMLHttpRequest 或上述 Flash 等方式，以（被攻击）用户的身份执行一些管理动作，或执行一些一般的如发微博、加好友、发私信等操作
- 利用可被攻击的域受到其他域信任的特点，以受信任来源的身份请求一些平时不允许的操作，如进行不当的投票活动
- 偷取网站任意数据、用户资料等等

### XSS 的类型

#### 反射型（非持久）

反射型 XSS，也叫非持久型 XSS，是指发生请求时，XSS 代码出现在请求 URL 中，作为参数提交到服务器，服务器解析并响应。响应内容包含 XSS 代码，返回给浏览器解析并执行，这个过程就像一次反射，所以叫反射型 XSS。

这种攻击方式通常需要攻击者诱使用户点击一个恶意链接，或者提交一个表单，或者进入一个恶意网站时，注入脚本进入被攻击者的网站。

该方式只会经过服务器，不会经过数据库。

例子：
比如用户进行搜索时，点击搜索按钮访问到如下链接

```
http://xxx.com/search?keyword="><script>alert('XSS攻击');</script>
```

当浏览器请求时，服务器解析参数 keyword，得到`"><script>alert('XSS攻击');</script>`，拼接到 HTML 中返回给浏览器，如下：

```html
<input type="text" value="" />
<script>
  alert('XSS攻击')
</script>
">
<button>搜索</button>
<div>
  您搜索的关键词是：">
  <script>
    alert('XSS攻击')
  </script>
</div>
```

因此将其执行了。

#### 存储型（持久）

存储型 XSS，也叫持久型 XSS，主要是将 XSS 代码发送到服务器，当浏览器请求数据时，脚本从服务器上传回并执行。与反射型 XSS 的差别在于，提交的代码会存储在服务器端，下次请求时目标页面时不用再提交 XSS 代码。

比较常见的场景就是网页的留言板，攻击者在留言板写下包含攻击性的脚本代码，发表之后所有访问该留言的用户，留言内容从服务器解析之后发现有 XSS 代码于是当做正常的 HTML 和 JS 解析执行，就发生了 XSS 攻击。

该方式会经过服务器，也会经过数据库。

#### DOM 型

基于 DOM 的 XSS 攻击是指通过恶意脚本修改页面的 DOM 结构，将攻击脚本写在 URL 中，诱导用户点击该 URL，如果 URL 被解析，那么攻击脚本就会被运行，和前两者 XSS 攻击区别是：取出和执行恶意代码由浏览器端完成，不经过服务端，属于前端 JavaScript 自身的安全漏洞，而其他两种 XSS 都属于服务端的安全漏洞主要在于 DOM 型攻击。

例子引用自：[DOM 型攻击例子](https://juejin.im/post/6844904090019840007#heading-5)

```html
<h2>XSS:</h2>
<input type="text" id="input" />
<button id="btn">Submit</button>
<div id="div"></div>
<script>
  const input = document.getElementById('input')
  const btn = document.getElementById('btn')
  const div = document.getElementById('div')

  let val

  input.addEventListener(
    'change',
    e => {
      val = e.target.value
    },
    false
  )

  btn.addEventListener(
    'click',
    () => {
      div.innerHTML = `<a href=${val}>testLink</a>`
    },
    false
  )
</script>
```

点击 Submit 按钮后，会在当前页面插入一个链接，其地址为用户的输入内容。如果用户在输入时构造了如下内容：

```
'' onclick=alert(/xss/)
```

用户提交之后，页面代码就变成了：

```html
<a href onlick="alert(/xss/)">testLink</a>
```

此时，用户点击生成的链接，就会执行对应的脚本。

### 如何防范 XSS

总体就是**不能将用户的输入直接存到服务器，需要对一些数据进行特殊处理**

#### 设置 HttpOnly

HttpOnly 是一个设置 cookie 是否可以被 javasript 脚本读取的属性，浏览器会禁止页面的 Javascript 访问带有 HttpOnly 属性的 Cookie。

严格来说，这种方式不是防御 XSS，而是在用户被 XSS 攻击之后，不被获取 Cookie 数据。

#### CSP 内容安全策略

CSP(content security policy)，是一个额外的安全层，用于检测并削弱某些特定类型的攻击，包括跨站脚本 (XSS) 和数据注入攻击等。

CSP 可以通过 HTTP 头部（Content-Security-Policy）或<meta>元素配置页面的内容安全策略，以控制浏览器可以为该页面获取哪些资源。比如一个可以上传文件和显示图片页面，应该允许图片来自任何地方，但限制表单的 action 属性只可以赋值为指定的端点。

现在主流的浏览器内置了防范 XSS 的措施，开启 CSP，即开启白名单，可阻止白名单以外的资源加载和运行

#### 输入检查

对于用户的任何输入要进行编码、解码和过滤：

- 编码：不能对用户输入的内容都保持原样，对用户输入的数据进行字符实体编码转义
- 解码：原样显示内容的时候必须解码，不然显示不到内容了
- 过滤：把输入的一些不合法的东西都过滤掉，从而保证安全性。如移除用户上传的 DOM 属性，如 onerror，移除用户上传的 Style 节点、iframe、script 节点等

对用户输入所包含的特殊字符或标签进行编码或过滤，如 `<`，`>`，`script`，防止 XSS 攻击

```javascript
function escHTML(str) {
  if (!str) return ''
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/x27/g, '&#039;')
    .replace(/x22/g, '&quto;')
}
```

#### 输出检查

用户的输入会存在问题，服务端的输出也会存在问题。一般来说，除富文本的输出外，在变量输出到 HTML 页面时，可以使用编码或转义的方式来防御 XSS 攻击。例如利用 [sanitize-html](https://github.com/apostrophecms/sanitize-html) 对输出内容进行有规则的过滤之后再输出到页面中。

#### 输入内容长度控制

对于不受信任的输入，都应该限定一个合理的长度。虽然无法完全防止 XSS 发生，但可以增加 XSS 攻击的难度。

#### 验证码

防止脚本冒充用户提交危险操作。

## CSRF

### 概述

CSRF(Cross Site Request Forgery)指的是**跨站请求伪造**，是一种劫持受信任用户向服务器发送非预期请求的攻击方式。跨域指的是请求来源于其他网站，伪造指的是非用户自身的意愿。

### 攻击方式

攻击者诱导受害者进入第三方网站，在第三方网站中，向被攻击网站发送跨站请求。利用受害者在被攻击网站已经获取的注册凭证，绕过后台的用户验证，达到冒充用户对被攻击的网站执行某项操作的目的。

与 XSS 攻击不同的是：XSS 是攻击者直接对我们的网站 A 进行注入攻击，CSRF 是通过网站 B 对我们的网站 A 进行伪造请求。

例子：你登录购物网站 A 之后点击一个恶意链接 B，B 请求了网站 A 的下单接口，结果是在网站 A 的帐号生成一个订单。其背后的原理是：网站 B 通过表单、get 请求来伪造网站 A 的请求，这时候请求会带上网站 A 的 cookies，若登录态是保存在 cookies 中，则实现了伪造攻击。

跨站请求可以用各种方式：图片 URL、超链接、CORS、Form 提交等等。部分请求方式可以直接嵌入在第三方论坛、文章中。

### CSRF 危害

- 用户的登录态被盗用
- 冒充用户完成操作或修改数据

### CSRF 的类型

#### GET 类型的 CSRF

例子引入自：[GET CSRF 例子](https://juejin.im/post/6844903904585449479#heading-1)

假设有这样一个场景：目标网站 A(`www.a.com`)，恶意网站 B(`www.b.com`)

两个网站的域名不一样，目标网站 A 上有一个删除文章的功能，通常是用户单击'删除文章'链接时才会删除指定的文章。这个链接是`www.a.com/blog/del?id=1`, id 代表不同的文章。实际上就是发起一个 GET 请求

- 无法使用 Ajax 发起 GET 请求。因为 CSRF 请求是跨域的，而 Ajax 有同源策略的限制
- 可以通过在恶意网站 B 上静态或者动态创建 img,script 等标签发起 GET 请求。将其 src 属性指向`www.a.com/blog/del?id=1`。通过标签的方式发起的请求不受同源策略的限制
- 最后欺骗已经登录目标网站 A 的用户访问恶意网站 B，那么就会携带网站 A 源的登录凭证向网站 A 后台发起请求，这样攻击就发生了

CSRF 攻击有以下几个关键点：

- 请求是跨域的，可以看出请求是从恶意网站 B 上发出的
- 通过 img, script 等标签来发起一个 GET 请求，因为这些标签不受同源策略的限制
- 发出的请求是身份认证后的

#### POST 类型的 CSRF

假如目标网站 A 上有发表文章的功能，那么我们就可以动态创建 form 标签，然后修改文章的题目。

在网站 B 中：

```javascript
function setForm() {
  var form = document.createElement('form')
  form.action = 'www.a.com/blog/article/update'
  form.methods = 'POST'
  var input = document.createElement('input')
  input.type = 'text'
  input.value = 'csfr攻击啦！'
  input.id = 'title'
  form.appendChild(input)
  document.body.appendChild(form)
  form.submit()
}
setForm()
```

上面代码可以看出，动态创建了 form 表单，然后调用 submit 方法，就可以通过跨域的伪造请求来实现修改目标网站 A 的某篇文章的标题了。

通常是利用自动提交的表单

```html
<form action="http://xxx.com/money" method="post">
  <input type="hidden" name="account" value="jacky" />
  <input type="hidden" name="amount" value="10000" />
</form>
<script>
  document.forms[0].submit()
</script>
```

#### 链接类型的 CSRF

链接类型的 CSRF 并不常见，比起其他两种用户打开页面就中招的情况，这种需要用户点击链接才会触发。这种类型通常是在论坛中发布的图片中嵌入恶意链接，或者以广告的形式诱导用户中招，攻击者通常会以比较夸张的词语诱骗用户点击，例如：

```html
<a href="http://test.com/csrf/withdraw.php?amount=1000&for=hacker" taget="_blank"> 重磅消息！！ </a>
```

### 如何防范 CSRF

#### 验证码

由于 CSRF 攻击伪造请求不会经过受攻击的网站，所以我们可以在网站加入验证码，这样必须通过验证码之后才能进行请求，有效的遏制了 CSRF 请求。

但是，这种方式不是万能的，并不是每个请求都加验证码，那样用户体验会非常不好，只能在部分请求添加，作为一种辅助的防御手段。

#### 验证 Referer

在 HTTP 协议中，头部有个 Referer 字段，他记录了该 HTTP 请求的来源地址，在服务端设置该字段的检验，通过检查该字段，就可以知道该请求是否合法，不过请求头也容易伪造。

#### cookie 设置 SameSite

设置 SameSite：设置 cookie 的 SameSite 值为 strict，这样只有同源网站的请求才会带上 cookie。这样 cookies 就不能被其他域名网站使用，达到了防御的目的。

#### 添加 token 验证

浏览器请求服务器时，服务器返回一个 token，每个请求都需要同时带上 token 和 cookie 才会被认为是合法请求

这是一种相对成熟的解决方案。要抵御 CSRF，关键在于在请求中放入攻击者所不能伪造的信息，并且该信息不存在于 Cookie 之中。在服务端随机生成 token，在 HTTP 请求中以参数的形式加入这个 token，并在服务器端建立一个拦截器来验证这个 token，如果请求中没有 token 或者 token 内容不正确，则认为可能是 CSRF 攻击而拒绝该请求。

#### 更换登录态方案

因为 CSRF 本质是伪造请求携带了保存在 cookies 中的信息，所以对 session 机制的登录态比较不利，如果更换 JWT（JSON Web Token）方案，其 token 信息一般设置到 HTTP 头部的，所以可以防御 CSRF 攻击。

## 参考

- [WEB 前端安全——XSS 和 CSRF](https://juejin.im/post/6844903876106125319)
  <br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，欢迎 star

## 前言

刚开始接触 WebRTC 的时候，容易被一些生僻的概念绕得团团转，在经历一段时间的查阅文章和实践后，本文就详细梳理下有关 WebRTC 入门的基础知识。

本文代码案例 [Github 地址](https://github.com/Jacky-Summer/webrtc-demo)

## RTC

RTC（Real-time Communications）实时通信，是实时音视频的一个简称。

## WebRTC 是什么

WebRTC (Web Real-Time Communications) 是 RTC 的一部分，是一项实时通讯技术，它允许网络应用或者站点，在不借助中间媒介的情况下，建立浏览器之间**点对点（Peer-to-Peer）的连接**，实现视频流/音频流或者其他任意数据的传输。

## WebRTC 的优缺点

### 优点

- 跨平台(Web、Windows、MacOS、Linux、iOS、Android)
- 免费、免插件、免安装，得到主流浏览器支持
- 强大的打洞能力，包含 STUN、ICE、TURN 的关键 NAT 和防火墙穿透技术

### 缺点

- 缺乏服务器方案的设计和部署
- 音频的设备适配问题

## 应用场景

- 音视频通话
- 视频/电话会议
- 远程访问主机
- 在线教育（直播连麦，屏幕录制，共享远程桌面）

## 采集音视频数据

该方法会提示用户给予使用媒体输入的许可，允许后会访问到电脑摄像头和麦克风，并拿到一个 MediaStream 对象，把该媒体流赋值给页面的 video 标签，就能看到和听到自己的音视频了，由此也拿到音视频流。

```js
async function createLocalMediaStream() {
  // (非https/localhost）下 navigator.mediaDevices 会返回 undefined
  const localStream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true,
  })
  document.getElementById('local').srcObject = localStream
}
```

可查阅：[MediaDevices.getUserMedia()](https://developer.mozilla.org/zh-CN/docs/Web/API/MediaDevices/getUserMedia)

## RTCPeerConnection

我们把发起 WebRTC 通信的两端被称为对等端，即是 Peer。所谓点对点通信（peer-to-peer）则是说两个客户端直连，发送数据不需要中间服务器，建立成功的连接则称为`PeerConnection`，而一次 WebRTC 通信可包含多个 `PeerConnection`。

[RTCPeerConnection](https://developer.mozilla.org/zh-CN/docs/Web/API/RTCPeerConnection) 则是创建点对点连接的 API，代表一个由本地计算机到远端的 WebRTC 连接。该接口提供了创建，保持，监控，关闭连接的方法的实现。

创建两个连接实例

```js
const peerA = new RTCPeerConnection()
const peerB = new RTCPeerConnection()
```

### API

- `pc.createOffer`：创建 offer 方法，方法会返回 SDP Offer 信息
- `pc.setLocalDescription` 设置本地 SDP 描述信息
- `pc.setRemoteDescription`：设置远端的 SDP 描述信息，即对方发过来的 SDP 信息
- `pc.createAnswer`：远端创建应答 Answer 方法，方法会返回 SDP Offer 信息
- `pc.ontrack`：设置完远端 SDP 描述信息后会触发该方法，接收对方的媒体流
- `pc.onicecandidate`：设置完本地 SDP 描述信息后会触发该方法，打开一个连接，开始运转媒体流
- `pc.addIceCandidate`：连接添加对方的网络信息
- `pc.setLocalDescription`：将 localDescription 设置为 offer，localDescription 即为我们需要发送给应答方的 sdp，此描述指定连接本地端的属性，包括媒体格式
- `pc.setRemoteDescription`：改变与连接相关的描述，该描述主要是描述有些关于连接的属性，例如对端使用的解码器

## WebRTC 音视频通信基本流程

1. 一方发起调用 getUserMedia 打开本地摄像头
2. 媒体协商（信令交换）
3. 建立通信

## 媒体协商

媒体协商主要指 SDP 交换。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2243cf299b4f4360b207d3d22175209c~tplv-k3u1fbpfcp-watermark.image?)

如上图，为媒体协商的大致流程：

1. 发起端 Amy 创建 Offer 并将 Offer 信息，并调用 setLocalDescription 将其保存起来，通过信令服务器传送给接收端 Bob
2. 接收端 Bob 收到对等端 Amy 的 Offer 信息后调用 setRemoteDescription 方法将其保存起来，并创建 Answer 信息，同理也将 Answer 消息通过 setLocalDescription 保存，并通过信令服务器传送给呼叫端 Amy
3. 呼叫端 Amy 收到对等端 Blob 的 Answer 信息后调用 setRemoteDescription 方法将其 Answer 保存起来

初次接触的看到这个流程估计有点懵到底在干嘛，这里我列举一些可能的疑惑点来进行逐一解释：

### 1. 为什么需要媒体协商?

媒体协商的作用就是让双方找到共同支持的媒体能力，从而能实现彼此之间的音视频通信。比如两个人想聊天，一个人只会讲中文，喜欢讨论前端；一个人只会讲英文，不喜欢前端技术；通过互换资料，发现这没法聊天。但如果出现另一个人会讲中文，喜欢研究代码和前端技术，跟第一个人交换信息后，发现这可以聊天。所以发起端与接收端能不能进行通信，

### 2. 媒体协商是在做什么？

媒体协商就是在**交换 SDP**的过程。会话发起者通过创建一个`offer`，经过信令服务器发送到接收方，接收方创建`answer`并返回给发送方，完成交换。

### 3. SDP 是什么？

SDP（Session Description Protocol）指**会话描述协议**，是一种通用的协议，基于文本，其本身并不属于传输协议，需要依赖其它的传输协议（如 RTP 交换媒体信息。

SDP 主要用来描述多媒体会话，用途包括会话声明、会话邀请、会话初始化等。通俗来讲，它可以表示各端的能力，记录有关于你音频编解码类型、编解码器相关的参数、传输协议等信息。

交换 SDP 时，通信的双方会将接受到的 SDP 和自己的 SDP 进行比较，取出他们之间的交集，这个交集就是协商的结果，也就是最终双方音视频通信时使用的音视频参数及传输协议。

### 4. offer 和 answer 是什么？

在双方要建立点对点通信时，发起端发送的 SDP 消息称为 Offer，接收端发送的 SDP 消息称为 Answer

所以，offer 和 answer 本质就是存有 SDP 信息的对象，所以也会叫做 SDP Offer 和 SDP Answer。

如下图，打印 offer 和 answer 信息

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6b249e531cfd48f297d1a808002ae568~tplv-k3u1fbpfcp-watermark.image?)

### 5. 信令与信令服务器

信令通常指的是为了网络中各种设备协调运作，在设备之间传递的控制信息。

对于 WebRTC 通信来说，发起端发送 Offer SDP 和接收端接受 Answer SDP，要怎么发给对方呢？这个过程还需要一种机制来协调通信并发送控制消息，这个过程就称为信令。

而信令对应的服务器就叫信令服务器，作为中间人帮助建立连接，主要负责：

- 信令的处理，如媒体协商消息的传递
- 管理房间信息。比如用户连接时告诉信令服务器自身的房间号，由信令服务器找到也在该房间号的对等端并开始尝试通信，也通知用户谁加入了房间和离开了房间，通知房间人数是否已满等等，所以也叫信令服务器也叫房间服务器。

WebRTC 并没有规定信令必须使用何种实现，目前业界使用较多的是 WebSocket + JSON/SDP 的方案。其中 **WebSocket** 用来提供信令传输通道，JSON/SDP 用来封装信令的具体内容。

## ICE

当媒体协商完成后，WebRTC 就开始建立网络连接，其过程称为 ICE（Interactive Connectivity Establishment）交互式连接建立。

ICE 不是一种协议，整合了 STUN 和 TURN 两种协议（用于 NAT 穿透）的框架。

ICE 是在各端调用 setLocalDescription() 后就开始了，其操作过程如下：

1. 收集 Candidate
2. 交换 Candidate
3. 按优先级尝试连接

在说完上述一些生僻的概念，我来逐一解释下涉及到的东西

### 1. 什么是 Candidate？

比如想用 socket 连接某台服务器，一定要知道这台服务器的一些基本信息，如服务器的 IP 地址、端口号以及使用的传输协议。只有知道了这些信息，才能与这台服务器建立连接。而 Candidate 正是 WebRTC 用来描述它可以连接的远端的基本信息，因此 Candidate 是至少包括 IP 地址、端口号、协议的一个信息集。

### 2. 收集 Candidate

在 WebRTC 中有三种类型的 **ICE 候选者（Candidate）**：

1. **主机候选者**：表示网卡自己的 IP 地址及端口。通过设备网卡获取，优先级最高。在 WebRTC 底层首先会尝试本地局域网内建立连接。

2. **反射候选者**：表示经过 NAT 之后的外网 IP 地址和端口，由 ICE（STUN）服务器获取，根据服务器的返回情况，来综合判断并知道自身在公网中的地址。其优先级低于主机候选者，当 WebRTC 尝试本地连接不通时，会尝试通过反射候选者获得的 IP 地址和端口进行连接。

3. **中继候选者**：表示的是中继(TURN)服务器的转发 IP 地址与端口，由 ICE 中继服务器提供。优先级最低，前两个都不行则会按该种方式。

在新建`RTCPeerConnection`时可在构造函数指定 ICE 服务器地址，没有指定的话则意味着这个连接只能在内网进行。

每次 WebRTC 找到/收集一个可用的 Candidate，都会触发一次`icecandidate`事件，为了将收集到的 Candidate 交换给对端，需要给`onicecandidate`方法设置一个回调函数，函数里面调用`addIceCandidate`方法来将候选者添加到通信中。

如下代码，通过该回调函数就可以获得 WebRTC 底层收集到的所有 Candidate 了。同时，还可以在该函数中将收集到的 Candidate 发送给对端。

```js
peer.onicecandidate = (event) => {
  if (event.candidate) {
    // ...
  }
}

// 接收到信令服务器发送过来的候选信息后调用，为本机添加 ICE 代理
peer.addIceCandidate(candidate)
```

### 3. 交换 Candidate

WebRTC 收集好 Candidate 后，会通过信令系统将它们发送给对端。对端接收到这些 Candidate 后，会与本地的 Candidate 形成 CandidatePair（即连接候选者对）。有了 CandidatePair，WebRTC 就可以开始尝试建立连接了。这里需要注意的是，Candidate 的交换不是等所有 Candidate 收集好后才进行的，而是边收集边交换。

> CandidatePair，候选者对，即一个本地 Candidate，一个远端 Candidate

当 WebRTC 形成 CandidatePair 后，便开始尝试进行连接。一旦 WebRTC 发现其中有一个可以连通的 CandidatePair 时，它就不再进行后面的连接尝试了，但发现新的 Candidate 时仍然可以继续进行交换。

### 4. NAT 是什么？

NAT：网络地址转换，它是一种解决专用网络内设备连接公网的技术。

这里我就不写了，社区已有通俗易懂的文章解释 NAT，推荐大家看完再继续往下看：

- [家庭网络中的「NAT」到底是什么？](https://sspai.com/post/68037)

### 4. STUN 是什么？

STUN（Session Traversal Utilities for NAT, NAT 会话穿越应用程序）。它允许位于 NAT（或多重 NAT）后的客户端找出自己对应的公网 IP 地址和端口，也就是俗称的"打洞"/"NAT 打洞"/"NAT 穿越"。

再直白点，STUN 服务器用于**获取计算机的公网 IP 地址**。

Google 提供了一个测试 STUN/TURN 服务的[网址](https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/)，可以测试对应的 STUN 服务。

打洞怎么理解呢？假如要翻越一座山，就要沿着山路从一遍的山脚到另一边的山脚。但如果山里有条直接可以通过的隧道，就相当于两个点（山脚）直接的进行连接，这就是"打洞"，实现点对点。

从这一点理解，打洞的机制叫 ICE，而帮忙打洞的服务器叫 STUN 服务。

在两个用户通信前，首先会向公网的 STUN 服务发送请求获取自己的公网地址，然后通过服务器将各自的公网地址转发给对等端，这样双方就知道了对方的公网地址，根据这个公网地址就可以直接点对点通信了。

可以使用谷歌提供的 SUTN 服务器进行配置

```js
const config = {
  iceServers: [
    {
      urls: 'stun:stun.l.google.com:19302',
    },
  ],
}

const peer = new RTCPeerConnection(config)
```

当 STUN 服务遇到对称型 NAT 时（这里如果不懂先去了解上方 NAT 相关的基础知识），就打洞失败了，这时就需要 TURN 服务。

### 5. TURN 是什么？

**TURN**(Traversal USing Replays around NAT)，即使用中继穿透 NAT，是 STUN 的扩展。

如果 STUN 分配公网 IP 的方式失败，则可以通过 TURN 服务器请求公网 IP 地址作为中继地址，将媒体数据通过 TURN 服务器进行中转，用作**对等连接失败的中继**，属于最终的备选方式。

目的就是**解决对称 NAT 无法穿越的问题**，不同于其它中继协议在于它允许客户机使用一个中继地址与多个对端同时进行通讯。完美弥补了 STUN 无法穿越对称型 NAT 的问题。

与`STUN`服务器不同的是，TURN 服务器会作为中转，转发多媒体数据会消耗大量的带宽。

ICE 打洞相关的过程，开发者只需配置好 STUN/TURN 对应的地址，在对应的函数调用就可以了，WebRTC 在底层帮我们去实现了。

## 最简易版的点对点通信实践

而如果是局域网本机来实现通信，都无需配置 STUN/TURN 服务。

在浏览器本地模拟最简单的点对点通信，我们直接本地代码同时新建两个对等端 Peer，这样无需信令服务传递 SDP 信息，不到 50 行 JS 代码就可以实现下图的效果：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/34ba6a1f505b447d9dc4a742eeb72924~tplv-k3u1fbpfcp-watermark.image?)

index.html

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>浏览器本地同页面模拟点对点连接（无信令服务版）</title>
    <style>
      video {
        width: 300px;
        height: 250px;
      }
    </style>
  </head>
  <body>
    我的本地视频：<video id="local" autoplay></video> 远程连接拿到我的本地视频<video
      id="local-from-remote"
      autoplay
    ></video>

    <script src="./index.js"></script>
  </body>
</html>
```

```javascript
let localStream

const createLocalMediaStream = async () => {
  localStream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: false,
  })
  document.getElementById('local').srcObject = localStream
}

const createPeerConnection = async () => {
  const peerA = new RTCPeerConnection()
  const peerB = new RTCPeerConnection()

  // 添加本地媒体流
  localStream.getTracks().forEach((track) => {
    peerA.addTrack(track, localStream)
  })

  // 监听 ice 候选项事件
  peerA.onicecandidate = (event) => {
    if (event.candidate) {
      peerB.addIceCandidate(event.candidate) // 设置 ice 候选项
    }
  }

  // 监听获取媒体数据（前提是peerA已添加了媒体流数据）
  peerB.ontrack = (event) => {
    document.getElementById('local-from-remote').srcObject = event.streams[0]
  }
  /**
   * 媒体协商（交换SDP）
   */
  const offer = await peerA.createOffer()
  await peerA.setLocalDescription(offer)

  await peerB.setRemoteDescription(offer)
  const answer = await peerB.createAnswer()
  await peerB.setLocalDescription(answer)

  await peerA.setRemoteDescription(answer)
}

const main = async () => {
  await createLocalMediaStream()
  await createPeerConnection()
}
main()
```

[代码完整地址](https://github.com/Jacky-Summer/webrtc-demo/tree/master/1-webrtc-local)

如果看完觉得 OK，可以去试下信令版（socket + 房间管理）的实现，

- 客户端：[webrtc-client](https://github.com/Jacky-Summer/webrtc-demo/tree/master/2-webrtc-client)
- 信令服务端：[webrtc-server](https://github.com/Jacky-Summer/webrtc-demo/tree/master/2-webrtc-server)

## 结语

本文对 WebRTC 流程做了大致讲解，但对相关概念进行具体解释说明，建议在学习的过程可以再多看一遍文章会更清晰。

## 参考资料

- [WenRTC 官网](https://webrtc.org)
- [《WebRTC 音视频实时互动技术》](https://book.douban.com/subject/35543112/)
- [从 0 到 1 打造一个 WebRTC 应用](https://juejin.cn/post/6896045087659130894)
- [Why WebRTC ｜“浅入深出”的工作原理详解](https://juejin.cn/post/6984747346009522212)
- [WebRTC：一个视频聊天的简单例子](https://juejin.cn/post/6844903906225602568)
- [有了 WebRTC，直播可以这样玩！](https://juejin.cn/post/6964571538729205773)
- [前端音视频 WebRTC 实时通讯的核心](https://juejin.cn/post/6884851075887661070)
- [# WebRTC: how two browsers agree on voice and video calls](https://sudonull.com/post/64126-WebRTC-how-two-browsers-agree-on-voice-and-video-calls-Voximplant-Blog)

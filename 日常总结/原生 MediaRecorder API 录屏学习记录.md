## 前言

很久没有更新文章了，最近一段时间在调整生活节奏。今天记录下最近公司业务的一个关于前端录屏的调研，写下其中 关于 MediaRecorder API 的学习记录。

## mediaDevices

首先简单的了解一下 mediaDevices

mediaDevices 接口提供访问连接媒体输入的设备，如照相机和麦克风，以及屏幕共享等。它可以使你取得任何硬件资源的媒体数据。

### enumerateDevices()

enumerateDevices() 请求一个可用的媒体输入和输出设备的列表，例如麦克风，摄像机，耳机设备等

```js
navigator.mediaDevices.enumerateDevices().then(deviceList => {
  // audioinput  麦克风
  // videoinput 摄像
  // audiooutput 扬声器
  console.log(deviceList)
})
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9683e985ca147fe82566d9d3d67f0a6~tplv-k3u1fbpfcp-watermark.image)

### getUserMedia()

该方法会提示用户给予使用媒体输入的许可，点击允许后会 resolve 回调一个 MediaStream 对象，若用户拒绝了使用权限，或者需要的媒体源不可用。

返回的 promise 对象可能既不会 resolve 也不会 reject，因为用户不是必须选择允许或拒绝。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fede118284194e1cb3bf518355adfa97~tplv-k3u1fbpfcp-watermark.image)

点击允许后：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07c185b3fbcf46f1a66c2ebd8523569c~tplv-k3u1fbpfcp-watermark.image)

### getDisplayMedia

这个方法会提示用户去选择和授权捕获展示的内容或部分内容（如一个窗口），录制电脑屏幕时其中的一个方法，调用方法成功后可以将这个流赋给 video 元素实现边录边看。

```js
navigator.mediaDevices.getDisplayMedia({ video: true }).then(
  stream => {
    console.log(stream)
  },
  error => console.log(error)
)
```

基于用户隐私安全考虑，会弹出一个选择框让你授权，然后根据自己选择需要共享哪一部分的内容。如可以共享当前屏幕，也可以共享其他的应用窗口和浏览器的其他标签页。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b51cbe3a7264218a64a3d14ecab53f5~tplv-k3u1fbpfcp-watermark.image)

## MediaRecord API

MediaRecorder 是 MediaStream Recording API 提供的用来进行媒体轻松录制的接口, 他需要通过调用 MediaRecorder() 构造方法进行实例化，它赋予了 web 页面独立录制音视频的能力。

- 摄像：getUserMedia（获取流） + MediaRecorder（存储）
- 录屏：getDisplayMedia（获取流） + MediaRecorder（存储）

这个 API 本身不复杂，看个例子就能理解了：

```html
<video id="video" width="500px" height="400px" autoplay controls></video>
<button id="start-screen">开始录制</button>
<button id="stop-screen">结束录制</button>
<button id="download">下载视频</button>
```

```js
const video = document.getElementById('video')
const startBtn = document.getElementById('start-screen')
const stopBtn = document.getElementById('stop-screen')
const download = document.getElementById('download')

startBtn.addEventListener('click', function (e) {
  startCapture()
})

stopBtn.addEventListener('click', function (evt) {
  stopCapture()
})

download.addEventListener('click', function (evt) {
  downloadVideo()
})

// 开始录制
async function startCapture() {
  try {
    captureStream = await navigator.mediaDevices.getDisplayMedia()
  } catch (err) {
    console.log('你取消了分享屏幕')
  }
  video.srcObject = captureStream
  recorder = new MediaRecorder(captureStream)
  recorder.start()
}

// 停止录制
function stopCapture() {
  recorder.stop()
  recorder.addEventListener('dataavailable', ({ data }) => {
    let videoUrl = URL.createObjectURL(data, { type: 'video/webm' })
    video.srcObject = null
    video.src = videoUrl
  })
}

// 下载视频
function downloadVideo() {
  const aTag = document.createElement('a')
  aTag.href = video.src
  aTag.download = `${new Date()}.webm`
  document.body.appendChild(aTag)
  aTag.click()
}
```

此时你就可以录制你电脑的屏幕并把它下载下来

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91acb0ef1f234420bbeafc391139686d~tplv-k3u1fbpfcp-watermark.image)

## 关于录屏一些调研

首先录制的视频流生成的格式，webm 支持度比较多。Chrome 浏览器使用 MediaRecord API 在最新版本也仍都不支持 mp4 格式，即是录屏生成的格式不支持 mp4，即使你改成 `.mp4`，播放器可以播放，但还是有问题，比如用火狐浏览器就无法播放那个 mp4 视频。

# 基于WebRTC视频流实现前端录屏

最近实现了基于 WebRTC 视频流实现录屏功能，本质还是直接使用原生的 MediaRecorder API。

关于 MediaRecorder 可以看看文档： [MediaRecorder](https://developer.mozilla.org/zh-CN/docs/Web/API/MediaRecorder)

## 遇到的一些问题解决

### webm格式视频第一次播放无法加载出进度条，只有播放完第二次才有进度条（视频时长）显示

> Chrome官方标记Won't Fix了，对此猜测Chrome不认为这是bug。视频长度这个如果没有在文件头部给出的话，就需要读取整个文件了，原因可能对较大size的视频加载不利

**解决方案**：

手动计算视频长度，分配给`blob`。

使用 `fix-webm-duration` 库， 用来补全duration字段，需要自己记录duration，不是很准确，仍有误差，误差在1s多以内，但侵入性较低解决起来简单

### webm视频补全进度条后仍无法自动聚焦后使用键盘左右键加减速

> 正常视频使用原生video标签的focus方法就可以使用键盘的左右键对视频加减速，但由于webm视频天生的不支持，即使赋了进度条依然不行

**解决方案**：

通过JS设置currentTime，直接把当前播放进度设到结尾，再把当前播放进度设到开头，模拟播放完成的情况，就修复了键盘左右快进后退了

```ts
// 修复webm视频键盘事件聚焦及播放速度控制

  useEffect(() => {

    const videoEle = document.querySelector(

      '#video-homework-popup',

    ) as HTMLVideoElement

    const duration = videoEle?.duration

    if (typeof duration === 'number' && !isNaN(duration)) {

      videoEle.currentTime = duration

 videoEle.currentTime = 0

    }

    videoEle?.focus()

    videoEle?.play()

  }, [homeworkVideoUrl])
```


## 抽离 useVideoRecorder

这里的录屏并不是调用电脑摄像头，也不是使用屏幕分享的API，而是基于远程视频，使用 canvas 不断的对视频进行绘制，将 canvas 绘制的流传入到 MediaRecorder 方法里面。

以下精简了除了业务之外的代码，纯属实现前端录屏的代码，当然代码很多优化的空间，仅做参考：

```ts
import React, { useEffect, useRef } from 'react'
import throttle from 'lodash/throttle'
import ysFixWebmDuration from 'fix-webm-duration'

const TimeInterval = 16
const DefaultMaxRecordMinutes = 15 // 默认最大录制时长约15分钟
const WatermarkParams = {
  width: 118,
  height: 42,
  marginRight: 25,
  marginTop: 17,
}
enum BitsPerSecond {
  '360P' = 1000000,
  '480P' = 2500000,
  '720P' = 5000000,
  '1080P' = 8000000,
}

interface RecorderOptions {
  videoRef: React.MutableRefObject<HTMLVideoElement | null> // 视频 video 标签
  videoContainerRef: React.MutableRefObject<HTMLDivElement | null> // video 标签外层的div
  watermark?: string
  maxRecordMinutes?: number // 视频最大录制时长（分）
  debug?: boolean
  getResolution: () => { width: number; height: number }
}

interface StartRecorderOptions {
  bitrate?: number
}

type CanvasCaptureMediaStreamTrack = MediaStreamTrack & {
  requestFrame: () => void
}

// 录屏当前的状态
enum RecordingState {
  INACTIVE = 'inactive', // 没有进行录制，原因可能是录制没有开始或已经停止
  PAUSED = 'paused', // 录制已开始，当前处于暂停状态
  RECORDING = 'recording', // 录制正在进行
}

const useVideoRecorder = ({
  videoRef,
  videoContainerRef,
  watermark,
  maxRecordMinutes = DefaultMaxRecordMinutes,
  debug,
  getResolution,
}: RecorderOptions) => {
  const recorder = useRef<MediaRecorder>()
  const recorderCanvas = useRef<HTMLCanvasElement>()
  const recorderChunks = useRef<Blob[]>([])
  const recorderStream = useRef<MediaStream | null>(null)
  const recorderVideoTrack = useRef<CanvasCaptureMediaStreamTrack>()
  const recorderContext = useRef<CanvasRenderingContext2D>()

  const watermarkImage = useRef<HTMLImageElement>()
  const cursorImage = useRef<HTMLImageElement>()
  const cursorContainer = useRef<HTMLDivElement>()
  const mousePosition = useRef<{ x: number; y: number }>({ x: 0, y: 0 })

  const refreshTimer = useRef<number>()
  const refreshTicks = useRef<number>(0)
  // 录制最大时长计算
  const recordTimer = useRef<number>()
  const durationTicks = useRef<number>(0)
  // 录制时长计算
  const startRecordTime = useRef<number>(0)
  const durationTime = useRef<number>(0)

  const isRecording = useRef<boolean>(false)

  // 初始化创建canvas
  useEffect(() => {
    recorderCanvas.current = document.createElement('canvas')
    const $recorderCanvas = recorderCanvas.current
    $recorderCanvas.setAttribute('style', 'display: none')
    $recorderCanvas.id = 'video-recorder-canvas'
    recorderContext.current = ($recorderCanvas.getContext(
      '2d',
    ) as unknown) as CanvasRenderingContext2D
    // debug canvas
    debug &&
      recorderCanvas.current.setAttribute(
        'style',
        'display: block; position: fixed; bottom: 0; left: 0; height: 350px; background: #fff; z-index: 10; border: 1px solid #fff',
      )

    document.body.appendChild(recorderCanvas.current)
    // 水印
    watermarkImage.current = document.createElement('img')
    watermark && watermarkImage.current.setAttribute('src', watermark)
    // 鼠标光标
    cursorImage.current = document.createElement('img')
    cursorContainer.current = document.createElement('div')
    cursorContainer.current.setAttribute(
      'style',
      'pointer-events: none; z-index: 100; display: inline-block; position: absolute;',
    )
    cursorContainer.current.appendChild(cursorImage.current)
  }, [])

  useEffect(() => {
    videoContainerRef.current?.addEventListener('mousemove', handleMousemove)

    return () => {
      videoContainerRef.current?.removeEventListener(
        'mousemove',
        handleMousemove,
      )
    }
  }, [])

  // 监听是否断网
  useEffect(() => {
    window.addEventListener('offline', resetVideoRecord)

    return () => {
      window.removeEventListener('offline', resetVideoRecord)
    }
  }, [])

  const handleMousemove = throttle((e: MouseEvent) => {
    mousePosition.current.x = e.offsetX
    mousePosition.current.y = e.offsetY
  }, 16)

  const onRefreshTimer = () => {
    refreshTicks.current++
    // 录屏
    if (
      isRecording.current &&
      refreshTicks.current % Math.round(64 / TimeInterval) === 0
    ) {
      recorderVideoTrack.current?.requestFrame()
      recorderDrawFrame()
    }
  }

  // 记录录屏时长
  const onRecordTimer = () => {
    durationTicks.current++
    if (durationTicks.current >= maxRecordMinutes * 60) {
      pauseRecord()
    }
  }

  const recorderDrawFrame = () => {
    const $recorderCanvas = recorderCanvas.current!
    const $player = videoRef.current!
    const ctx = recorderContext.current!
    const { width, height } = getResolution() // 获取视频实时宽高的方法
    $recorderCanvas.width = width // $player.videoWidth
    $recorderCanvas.height = height // $player.videoHeight

    ctx.drawImage(
      $player,
      0,
      0,
      $player.videoWidth,
      $player.videoHeight,
      0,
      0,
      $recorderCanvas.width,
      $recorderCanvas.height,
    )
    drawWatermark(ctx, width)
  }

  // 添加水印，图片水印需为base64格式
  const drawWatermark = (
    ctx: CanvasRenderingContext2D,
    canvasWidth: number,
  ) => {
    if (watermark) {
      ctx.drawImage(
        watermarkImage.current!,
        canvasWidth - WatermarkParams.width - WatermarkParams.marginRight,
        WatermarkParams.marginTop,
      )
    }
  }

  // 开始录屏
  const startRecord = (options: StartRecorderOptions = {}) => {
    if (
      recorder.current?.state === RecordingState.RECORDING ||
      recorder.current?.state === RecordingState.PAUSED
    ) {
      return
    }

    console.log('start record')
    recorderStream.current = recorderCanvas.current!.captureStream(0)
    recorderVideoTrack.current = recorderStream.current!.getVideoTracks()[0] as CanvasCaptureMediaStreamTrack
    const audioTrack = videoRef.current?.srcObject?.getAudioTracks()[0]
    if (audioTrack) {
      recorderStream.current!.addTrack(audioTrack) // 录入声音
    }

    if (!window.MediaRecorder) {
      return false
    }

    const mimeType = 'video/webm;codecs=vp8'
    recorder.current = new MediaRecorder(recorderStream.current, {
      mimeType,
      // 指定音频和视频的比特率
      bitsPerSecond: options.bitrate || BitsPerSecond['360P'],
    })
    isRecording.current = true
    refreshTimer.current = window.setInterval(onRefreshTimer, 16)
    recordTimer.current = window.setInterval(onRecordTimer, 1000)
    recorder.current.ondataavailable = handleRecordData // 停止录像以后的回调函数，返回一个存储Blob内容的录制数据
    recorder.current.start(10000) // 开始录制媒体
    startRecordTime.current = Date.now()
  }

  // 暂停录屏 - 适用于录屏超过录制最大时长
  const pauseRecord = () => {
    if (
      recorder.current &&
      recorder.current?.state === RecordingState.RECORDING
    ) {
      recorder.current.pause()
      isRecording.current = false
      clearInterval(recordTimer.current)
      clearInterval(refreshTimer.current)
      durationTime.current = Date.now() - startRecordTime.current
    }
  }

  // 停止录屏
  const stopRecord = () => {
    return new Promise((resolve, reject) => {
      if (
        recorder.current?.state === RecordingState.RECORDING ||
        recorder.current?.state === RecordingState.PAUSED
      ) {
        console.log('stop record')
        if (!window.MediaRecorder) {
          reject(new Error('Your Browser are not support MediaRecorder API'))
        }

        recorder.current?.stop()
        recorderVideoTrack.current!.stop()
        clearInterval(refreshTimer.current)
        clearInterval(recordTimer.current)
        isRecording.current = false
        recorder.current.onstop = () => {
          if (!durationTime.current) {
            durationTime.current = Date.now() - startRecordTime.current
          }

          // 修复 webm 视频录制无时长，赋值时长给 blob
          ysFixWebmDuration(
            new Blob(recorderChunks.current, { type: 'video/webm' }),
            durationTime.current,
            function (fixedBlob: Blob) {
              resolve(fixedBlob)
              recorderChunks.current = []
              durationTime.current = 0
            },
          )
        }
      } else {
        reject(new Error('Recorder is not started'))
      }
    })
  }

  const resetVideoRecord = () => {
    if (
      recorder.current?.state === RecordingState.RECORDING ||
      recorder.current?.state === RecordingState.PAUSED
    ) {
      recorder.current?.stop()
      recorderVideoTrack.current!.stop()
      recorder.current.onstop = () => {
        recorderChunks.current = []
        recorderStream.current = null
      }
    }
    isRecording.current = false
    clearInterval(refreshTimer.current)
    clearInterval(recordTimer.current)
  }

  // 处理录屏视频流数据
  const handleRecordData = (e: BlobEvent) => {
    if (e.data.size > 0 && recorderChunks.current) {
      recorderChunks.current.push(e.data)
    }
  }

  // 下载视频
  const download = (blob: Blob) => {
    if (recorder.current && blob.size > 0) {
      const name = new Date().getTime()
      const a = document.createElement('a')
      a.href = URL.createObjectURL(blob)
      a.download = `${name}.webm`
      document.body.appendChild(a)
      a.click()
    }
  }

  return {
    startRecord,
    stopRecord,
    resetVideoRecord,
    download,
  }
}

export default useVideoRecorder
```

## 兼容性

如果要做前端录屏，需要考虑兼容性的问题

### MediaRecorder API

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/893ea22069b140b3bcabc07092676710~tplv-k3u1fbpfcp-zoom-1.image)

-   对Safari低版本的兼容（主要考虑到 Mac 微信浏览器）

### Webm 格式

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3811781cf7f645619e6ef5afa5c77ba3~tplv-k3u1fbpfcp-zoom-1.image)
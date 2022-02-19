---
theme: channing-cyan
---

## 前言

平时开发中，大家用的比较多的会是 Element/Console/Network/Application 等面板，而 Performance 面板比较少用，而当定位性能优化或优化时就需要熟悉该性能面板的使用与分析，今天就要分享下 Performance 面板大致信息。

## 记录页面性能

大家也可以看谷歌官方的[Demo](https://googlechrome.github.io/devtools-samples/jank/)地址来是试试

选择开发者工具的 Performance，可以看到左边有三个按钮，一个点击是直接开始录制，第二个是重新加载页面并录制，第三个便是清除。

如果你想直接看首页性能评测，可以直接点击第二个按钮。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5050285ebabf4cd59b7061be0b308dff~tplv-k3u1fbpfcp-watermark.image?)

为了避免浏览器插件干扰，你可以打开无痕模式来试任意一个网站，点击后等待几秒后关闭，Performance 显示大致如下：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3aa2f243bae34ca5a90eec5ab74ad217~tplv-k3u1fbpfcp-watermark.image?)

## 设置面板

点击右边的设置按钮，可展开如下菜单：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42386f2e36c34530ae8d7ae919f9c77a~tplv-k3u1fbpfcp-watermark.image?)

- `Disable JavaScript samples`：禁用 JavaScript 采样。禁用之后记录中会忽略所有 JavaScript 的调用栈，记录的 Main 部分会比开启更简短。
- `Enable advanced paint instrumentation(slow)`：开启加速渲染工具。会带来大量的性能开销
- `Network`：控制网络
- `CPU`：控制录制过程中 CPU 工作频率。如 `4x slowdown` 选项会使你本地 CPU 运算速率比正常情况下降低 4 倍。

## FPS

FPS（frames per second）每秒帧数，是来衡量动画的一个性能指标。

正常如果网页流畅度较好的话，FPS 应该维持在 60 前后，即一秒之内进行 60 次重新渲染，每次重新渲染的时间不能超过 16.7ms 左右。如下图，一般绿色条部分越高，说明帧率越好。如果有红色长条，就代表帧率太低可能存在问题。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dfbfbf2a7a614934841f654bffefd4d8~tplv-k3u1fbpfcp-watermark.image?)

当然，红色条多的话，如下图，则说明页面掉帧影响到用户体验了。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/509862620d0d44cbbefd800499b2fb24~tplv-k3u1fbpfcp-watermark.image?)

鼠标移动到 FPS,CPU 或者 NET 图表上任意地方，都会显示在该时间节点上的屏幕截图（前提是勾选了 `Screenshots`），将你的鼠标左右移动，可以看到整个页面加载的进程。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d8bc05ce93094b4ebfc3bab694d1b772~tplv-k3u1fbpfcp-watermark.image?)

## CPU

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3eded62a844245ea947f994b8930e841~tplv-k3u1fbpfcp-watermark.image?)

FPS 下面就是 CPU 图表，图表中的颜色和面板底部的`Summary`tab 中的颜色是匹配的。CPU 图标颜色越丰富，代表在录制过程中 CPU 利用已最大化。当然如果这段丰富颜色的长条比较长，就说明一直在占用 CPU，此时就可能导致网页卡顿，这就需要介入代码优化。

## NET

NET 中每条横扛表示一种资源，横杠越长，表示请求资源所需的时间越长，与下面的`Network`一样的。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4313a72b6ae450cac14d71e4b16e284~tplv-k3u1fbpfcp-watermark.image?)

## 火焰图

### Network

可以看出网络请求的详细情况

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/493e4631f9e84dcbacee504674265eb0~tplv-k3u1fbpfcp-watermark.image?)

### Frame

表示每帧的运行情况。鼠标移至下方的 Frame 的绿色方块部分，可以看到该特定帧上的 FPS 值。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8c6fb4f2d49541d7b82067036e57c75d~tplv-k3u1fbpfcp-watermark.image?)

### Timings

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35a04527ab15423bbce1a358bf6a89c1~tplv-k3u1fbpfcp-watermark.image?)

- FP（first paint）：指的是首个像素开始绘制到屏幕上的时机，例如一个页面的背景色
- FCP（first contentful paint）：指的是开始绘制内容的时机，如文字或图片
- LCP（Largest Contentful Paint）：视口内可见的最大内容元素的渲染时间
- FMP（First Meaningful Paint）：首次有意义的绘制
- DCL（DOMContentLoaded）：表示 HTML 已经完全被加载和解析
- L（Onload）:页面所有资源加载完成事件

### Main

记录渲染进程中主线程的执行记录，点击 Main 可以看到某个任务执行的具体情况，可以分析主线程的 Event Loop，分析每个 Task 的耗时、调用栈等信息

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7cc77938f6c046b2bee913358a56eeca~tplv-k3u1fbpfcp-watermark.image?)

面板中会有很多的 Task，如果是耗时长的 Task，其右上角会**标红**，这个时候就可以选中标红的 Task，定位到耗时函数，然后针对性去优化。

### Compositor

Compositor 合成线程的执行记录，用来记录 html 绘制阶段 (Paint)结束后的图层合成操作

### Raster

光栅化线程池，用来让 GPU 执行光栅化的任务

### GPU

可以直观看到何时启动 GPU 加速

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1d1d73b751b74e2fb4da656010526c0b~tplv-k3u1fbpfcp-watermark.image?)

## 统计面板

### Summary

统计图。展示各个事件阶段耗费的时间

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3732f3f740994f3a8679d32400b25c72~tplv-k3u1fbpfcp-watermark.image?)

- 蓝色(`Loading`)：网络通信和 HTML 解析
  事件 | 描述 |
  | ---------------- | ----------------------------------- |
  |Parse HTML | 浏览器执行 HTML 解析 |
  | Finish Loading | 网络请求完毕事件 |
  | Receive Data | 请求的响应数据到达事件，如果响应数据很大（拆包），可能会多次触发该事件 |
  | Receive Response | 响应头报文到达时触发 |
  | Send Request | 发送网络请求时触发

- 黄色(`Scripting`)：JavaScript 执行
  事件 | 描述 |
  | ---------------- | ----------------------------------- |
  Animation Frame Fired | 一个定义好的动画帧发生并开始回调处理时触发 |
  | Cancel Animation Frame | 取消一个动画帧时触发 |
  | GC Event | 垃圾回收时触发 |
  | DOMContentLoaded | 当页面中的 DOM 内容加载并解析完毕时触发 |
  | Evaluate Script | A script was evaluated. |
  | Event | js 事件 |
  | Function Call | 只有当浏览器进入到 js 引擎中时触发 |
  | Install Timer | 创建计时器（调用 setTimeout()和 setInterval()）时触发 |
  | Request Animation Frame | requestAnimationFrame（）调用预定一个新帧 |
  | Remove Timer | 当清除一个计时器时触发 |
  | Time | 调用 console.time()触发 |
  | Time End | 调用 console.timeEnd()触发 |
  | Timer Fired | 定时器激活回调后触发 |
  | XHR Ready State Change | 当一个异步请求为就绪状态后触发 |
  | XHR Load | 当一个异步请求完成加载后触发
- 紫色(`Rendering`)：样式计算和布局
  事件 | 描述 |
  | ---------------- | ----------------------------------- |
  |Invalidate layout | 当 DOM 更改导致页面布局失效时触发 |
  | Layout | 页面布局计算执行时触发 |
  | Recalculate style | Chrome 重新计算元素样式时触发 |
  | Scroll | 内嵌的视窗滚动时触发
- 绿色(`Painting`)
  事件 | 描述 |
  | ---------------- | ----------------------------------- |
  |Composite Layers | Chrome 的渲染引擎完成图片层合并时触发 |
  | Image Decode | 一个图片资源完成解码后触发 |
  | Image Resize | 一个图片被修改尺寸后触发 |
  | Paint | 合并后的层被绘制到对应显示区域后触发
- 灰色(`other`)：其它事件花费的时间
- 白色(`Idle`)：空闲时间

### Bottom-Up

当想分析那些活动占用时间更多时，可以在该 Tab 看到各个事件消耗时间排序

- `Self Time`：指除去子事件这个事件本身消耗的时间
- `Total Ttime`：这个事件从开始到结束消耗的时间（包含子事件）

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9760ecda04874ac79e92cb2f9bea3a35~tplv-k3u1fbpfcp-watermark.image?)

### Call Tree

调用栈。Main 选择一个事件，表示事件调用顺序列表

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e8d6704f4d564a5c8d1e092d943adcbf~tplv-k3u1fbpfcp-watermark.image?)

### Event Log

表示事件发生的顺序列表

- 可以看到事件的开始触发时间（start time），右边有事件描述信息

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f7bb7feb970e436cacf65b37298b926d~tplv-k3u1fbpfcp-watermark.image?)

## 参考文章

- [Performance features reference](https://developer.chrome.com/docs/devtools/evaluate-performance/reference/)
- [网页性能分析不完全指南](https://segmentfault.com/a/1190000012243560)
- [chrome-performance 页面性能分析使用教程](https://www.cnblogs.com/ranyonsue/p/9342839.html)

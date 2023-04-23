# 【解读 ahooks 源码系列】DOM 篇（四）

## 前言

本文是 ahooks 源码（[v3.7.4](https://github.com/alibaba/hooks/tree/v3.7.4)）系列的第五篇，也是 DOM 篇的完结篇，往期文章：

- [【解读 ahooks 源码系列】（开篇）如何获取和监听 DOM 元素](https://juejin.cn/post/7201889983170592824)：useEffectWithTarget
- [【解读 ahooks 源码系列】DOM 篇（一）](https://juejin.cn/post/7202254039215800375)：useEventListener、useClickAway、useDocumentVisibility、useDrop、useDrag
- [【解读 ahooks 源码系列】DOM 篇（二）](https://juejin.cn/post/7202633255043465271)：useEventTarget、useExternal、useTitle、useFavicon、useFullscreen、useHover
- [【解读 ahooks 源码系列】DOM 篇（三）](https://juejin.cn/post/7202996870251380791)：useMutationObserver、useInViewport、useKeyPress、useLongPress

本文主要解读 `useMouse`、`useResponsive`、`useScroll`、`useSize`、`useFocusWithin`的源码实现

## useMouse

监听鼠标位置。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-mouse)

### 基本用法

API：

```ts
const state: {
  screenX: number, // 距离显示器左侧的距离
  screenY: number, // 距离显示器顶部的距离
  clientX: number, // 距离当前视窗左侧的距离
  clientY: number, // 距离当前视窗顶部的距离
  pageX: number, // 距离完整页面左侧的距离
  pageY: number, // 距离完整页面顶部的距离
  elementX: number, // 距离指定元素左侧的距离
  elementY: number, // 距离指定元素顶部的距离
  elementH: number, // 指定元素的高
  elementW: number, // 指定元素的宽
  elementPosX: number, // 指定元素距离完整页面左侧的距离
  elementPosY: number, // 指定元素距离完整页面顶部的距离
} = useMouse(target?: Target);
```

[官方在线 Demo](https://ahooks.js.org/zh-CN/hooks/use-mouse)

```ts
import React, { useRef } from 'react'
import { useMouse } from 'ahooks'

export default () => {
  const ref = useRef(null)
  const mouse = useMouse(ref.current)

  return (
    <>
      <div
        ref={ref}
        style={{
          width: '200px',
          height: '200px',
          backgroundColor: 'gray',
          color: 'white',
          lineHeight: '200px',
          textAlign: 'center',
        }}
      >
        element
      </div>
      <div>
        <p>
          Mouse In Element - x: {mouse.elementX}, y: {mouse.elementY}
        </p>
        <p>
          Element Position - x: {mouse.elementPosX}, y: {mouse.elementPosY}
        </p>
        <p>
          Element Dimensions - width: {mouse.elementW}, height: {mouse.elementH}
        </p>
      </div>
    </>
  )
}
```

### 核心实现

实现原理：通过监听 [mousemove](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/mousemove_event) 方法，获取鼠标的位置。通过 [getBoundingClientRect](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/getBoundingClientRect)（提供了元素的大小及其相对于视口的位置） 获取到 target 元素的位置大小，计算出鼠标相对于元素的位置。

```ts
export default (target?: BasicTarget) => {
  const [state, setState] = useRafState(initState)

  useEventListener(
    'mousemove',
    (event: MouseEvent) => {
      const { screenX, screenY, clientX, clientY, pageX, pageY } = event
      const newState = {
        screenX,
        screenY,
        clientX,
        clientY,
        pageX,
        pageY,
        elementX: NaN,
        elementY: NaN,
        elementH: NaN,
        elementW: NaN,
        elementPosX: NaN,
        elementPosY: NaN,
      }
      const targetElement = getTargetElement(target)
      if (targetElement) {
        const { left, top, width, height } = targetElement.getBoundingClientRect()

        // 计算鼠标相对于元素的位置
        newState.elementPosX = left + window.pageXOffset // window.pageXOffset：window.scrollX 的别名
        newState.elementPosY = top + window.pageYOffset // scrollY 的别名
        newState.elementX = pageX - newState.elementPosX
        newState.elementY = pageY - newState.elementPosY
        newState.elementW = width
        newState.elementH = height
      }
      setState(newState)
    },
    {
      target: () => document,
    }
  )

  return state
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useMouse/index.ts)

## useResponsive

获取响应式信息。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-mouse)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/useresponsive-demo1/)

```ts
import React from 'react'
import { configResponsive, useResponsive } from 'ahooks'

configResponsive({
  small: 0,
  middle: 800,
  large: 1200,
})

export default function () {
  const responsive = useResponsive()
  return (
    <>
      <p>Please change the width of the browser window to see the effect: </p>
      {Object.keys(responsive).map((key) => (
        <p key={key}>
          {key} {responsive[key] ? '✔' : '✘'}
        </p>
      ))}
    </>
  )
}
```

### 实现思路

1. 监听 [resize](https://developer.mozilla.org/en-US/docs/Web/API/Window/resize_event) 事件，在 resize 事件处理函数中需要计算，且判断是否需要更新处理（性能优化）。
2. 计算：遍历对比 `window.innerWidth` 与配置项的每一种屏幕宽度，大于设置为 true，否则为 false

### 核心实现

```ts
type Subscriber = () => void

const subscribers = new Set<Subscriber>()

type ResponsiveConfig = Record<string, number>
type ResponsiveInfo = Record<string, boolean>

let info: ResponsiveInfo

// 默认的响应式配置和 bootstrap 是一致的
let responsiveConfig: ResponsiveConfig = {
  xs: 0,
  sm: 576,
  md: 768,
  lg: 992,
  xl: 1200,
}

function handleResize() {
  const oldInfo = info
  calculate()
  if (oldInfo === info) return // 没有更新，不处理
  for (const subscriber of subscribers) {
    subscriber()
  }
}

let listening = false // 避免多次监听

// 计算当前的屏幕宽度与配置比较
function calculate() {
  const width = window.innerWidth // 返回窗口的的宽度
  const newInfo = {} as ResponsiveInfo
  let shouldUpdate = false // 判断是否需要更新
  for (const key of Object.keys(responsiveConfig)) {
    newInfo[key] = width >= responsiveConfig[key]
    if (newInfo[key] !== info[key]) {
      shouldUpdate = true
    }
  }
  if (shouldUpdate) {
    info = newInfo
  }
}

// 自定义配置响应式断点（只需配置一次）
export function configResponsive(config: ResponsiveConfig) {
  responsiveConfig = config
  if (info) calculate()
}

export function useResponsive() {
  if (isBrowser && !listening) {
    info = {}
    calculate()
    window.addEventListener('resize', handleResize)
    listening = true
  }
  const [state, setState] = useState<ResponsiveInfo>(info)

  useEffect(() => {
    if (!isBrowser) return

    // In React 18's StrictMode, useEffect perform twice, resize listener is remove, so handleResize is never perform.
    // https://github.com/alibaba/hooks/issues/1910
    if (!listening) {
      window.addEventListener('resize', handleResize)
    }

    const subscriber = () => {
      setState(info)
    }
    // 添加订阅
    subscribers.add(subscriber)
    return () => {
      // 组件卸载时取消订阅
      subscribers.delete(subscriber)
      // 当全局订阅器不再有订阅器，则移除 resize 监听事件
      if (subscribers.size === 0) {
        window.removeEventListener('resize', handleResize)
        listening = false
      }
    }
  }, [])

  return state
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useResponsive/index.ts)

## useScroll

监听元素的滚动位置。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-scroll)

### 基本用法

[官方在线 Demo，下方代码的执行结果](https://ahooks.js.org/~demos/usescroll-demo1/)

```ts
import React, { useRef } from 'react'
import { useScroll } from 'ahooks'

export default () => {
  const ref = useRef(null)
  const scroll = useScroll(ref)
  return (
    <>
      <p>{JSON.stringify(scroll)}</p>
      <div
        style={{
          height: '160px',
          width: '160px',
          border: 'solid 1px #000',
          overflow: 'scroll',
          whiteSpace: 'nowrap',
          fontSize: '32px',
        }}
        ref={ref}
      >
        <div>
          Lorem ipsum dolor sit amet, consectetur adipisicing elit. A aspernatur atque, debitis ex excepturi explicabo
          iste iure labore molestiae neque optio perspiciatis
        </div>
        <div>
          Aspernatur cupiditate, deleniti id incidunt mollitia omnis! A aspernatur assumenda consequuntur culpa cumque
          dignissimos enim eos, et fugit natus nemo nesciunt
        </div>
        <div>
          Alias aut deserunt expedita, inventore maiores minima officia porro rem. Accusamus ducimus magni modi mollitia
          nihil nisi provident
        </div>
        <div>
          Alias aut autem consequuntur doloremque esse facilis id molestiae neque officia placeat, quia quisquam
          repellendus reprehenderit.
        </div>
        <div>
          Adipisci blanditiis facere nam perspiciatis sit soluta ullam! Architecto aut blanditiis, consectetur corporis
          cum deserunt distinctio dolore eius est exercitationem
        </div>
        <div>Ab aliquid asperiores assumenda corporis cumque dolorum expedita</div>
        <div>
          Culpa cumque eveniet natus totam! Adipisci, animi at commodi delectus distinctio dolore earum, eum expedita
          facilis
        </div>
        <div>
          Quod sit, temporibus! Amet animi fugit officiis perspiciatis, quis unde. Cumque dignissimos distinctio, dolor
          eaque est fugit nisi non pariatur porro possimus, quas quasi
        </div>
      </div>
    </>
  )
}
```

### 核心实现

```ts
function useScroll(
  target?: Target, // DOM 节点或者 ref
  shouldUpdate: ScrollListenController = () => true // 控制是否更新滚动信息
): Position | undefined {
  const [position, setPosition] = useRafState<Position>()

  const shouldUpdateRef = useLatest(shouldUpdate) // 控制是否更新滚动信息，默认值： () => true

  useEffectWithTarget(
    () => {
      const el = getTargetElement(target, document)
      if (!el) {
        return
      }
      // 核心处理
      const updatePosition = () => {}

      updatePosition()

      // 监听 scroll 事件
      el.addEventListener('scroll', updatePosition)
      return () => {
        el.removeEventListener('scroll', updatePosition)
      }
    },
    [],
    target
  )

  return position // 滚动容器当前的滚动位置
}
```

接下来看看`updatePosition`方法的实现：

```ts
const updatePosition = () => {
  let newPosition: Position
  // target属性传 document
  if (el === document) {
    // scrollingElement 返回滚动文档的 Element 对象的引用。
    // 在标准模式下，这是文档的根元素， document.documentElement。
    // 当在怪异模式下，scrollingElement 属性返回 HTML body 元素（若不存在返回 null）
    if (document.scrollingElement) {
      newPosition = {
        left: document.scrollingElement.scrollLeft,
        top: document.scrollingElement.scrollTop,
      }
    } else {
      // 怪异模式的处理：取 window.pageYOffset, document.documentElement.scrollTop, document.body.scrollTop 三者中最大值
      // https://developer.mozilla.org/zh-CN/docs/Web/API/Document/scrollingElement
      // https://stackoverflow.com/questions/28633221/document-body-scrolltop-firefox-returns-0-only-js
      newPosition = {
        left: Math.max(window.pageXOffset, document.documentElement.scrollLeft, document.body.scrollLeft),
        top: Math.max(window.pageYOffset, document.documentElement.scrollTop, document.body.scrollTop),
      }
    }
  } else {
    newPosition = {
      left: (el as Element).scrollLeft, // 获取滚动条到元素左边的距离（滚动条滚动了多少像素）
      top: (el as Element).scrollTop,
    }
  }
  // 	判断是否更新滚动信息
  if (shouldUpdateRef.current(newPosition)) {
    setPosition(newPosition)
  }
}
```

- [Element.scrollLeft](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/scrollLeft) 获取滚动条到元素左边的距离
- [Element.scrollTop](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/scrollTop) 获取滚动条到元素顶部的距离

## useSize

监听 DOM 节点尺寸变化的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-size)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usesize-demo1/)

```ts
import React, { useRef } from 'react'
import { useSize } from 'ahooks'

export default () => {
  const ref = useRef(null)
  const size = useSize(ref)
  return (
    <div ref={ref}>
      <p>Try to resize the preview window </p>
      <p>
        width: {size?.width}px, height: {size?.height}px
      </p>
    </div>
  )
}
```

### 核心实现

这里涉及 [ResizeObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/ResizeObserver)

源码较容易理解，就不展开了

```ts
// 	目标 DOM 节点的尺寸
type Size = { width: number; height: number }

function useSize(target: BasicTarget): Size | undefined {
  const [state, setState] = useRafState<Size>()

  useIsomorphicLayoutEffectWithTarget(
    () => {
      const el = getTargetElement(target)

      if (!el) {
        return
      }
      // Resize Observer API 提供了一种高性能的机制，通过该机制，代码可以监视元素的大小更改，并且每次大小更改时都会向观察者传递通知
      const resizeObserver = new ResizeObserver((entries) => {
        entries.forEach((entry) => {
          // 返回 DOM 节点的尺寸
          const { clientWidth, clientHeight } = entry.target
          setState({
            width: clientWidth,
            height: clientHeight,
          })
        })
      })

      // 监听目标元素
      resizeObserver.observe(el)
      return () => {
        resizeObserver.disconnect()
      }
    },
    [],
    target
  )

  return state
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useSize/index.ts)

## useFocusWithin

监听当前焦点是否在某个区域之内，同 css 属性: [focus-within](https://developer.mozilla.org/en-US/docs/Web/CSS/:focus-within)

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-focus-within)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usefocuswithin-demo1/)

使用 ref 设置需要监听的区域。可以通过鼠标点击外部区域，或者使用键盘的 tab 等按键来切换焦点。

```ts
import React, { useRef } from 'react'
import { useFocusWithin } from 'ahooks'
import { message } from 'antd'

export default () => {
  const ref = useRef(null)
  const isFocusWithin = useFocusWithin(ref, {
    onFocus: () => {
      message.info('focus')
    },
    onBlur: () => {
      message.info('blur')
    },
  })
  return (
    <div>
      <div
        ref={ref}
        style={{
          padding: 16,
          backgroundColor: isFocusWithin ? 'red' : '',
          border: '1px solid gray',
        }}
      >
        <label style={{ display: 'block' }}>
          First Name: <input />
        </label>
        <label style={{ display: 'block', marginTop: 16 }}>
          Last Name: <input />
        </label>
      </div>
      <p>isFocusWithin: {JSON.stringify(isFocusWithin)}</p>
    </div>
  )
}
```

### 核心实现

主要还是监听了 [focusin](https://developer.mozilla.org/en-US/docs/Web/API/Element/focusin_event) 和 [focusout](https://developer.mozilla.org/en-US/docs/Web/API/Element/focusout_event) 事件

- focusin：当元素聚焦时会触发。和 focus 一样，只是 focusin 事件支持冒泡；
- focusout：当元素即将失去焦点时会被触发。和 blur 一样，只是 focusout 事件支持冒泡。

触发顺序：

在同时支持四种事件的浏览器中，当焦点在两个元素之间切换时，触发顺序如下（不同浏览器效果可能不同）：

- focusin 在第一个目标元素获得焦点前触发
- focus  在第一个目标元素获得焦点后触发
- focusout  第一个目标失去焦点时触发
- focusin  第二个元素获得焦点前触发
- blur  第一个元素失去焦点时触发
- focus  第二个元素获得焦点后触发

参考：[focus/blur VS focusin/focusout](https://juejin.cn/post/6888302279753990158#heading-7)

`MouseEvent.relatedTarget` 属性返回与触发鼠标事件的元素相关的元素：
![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bab123a0cf794e4dbe3075d183ff7535~tplv-k3u1fbpfcp-watermark.image?)

```ts
export default function useFocusWithin(target: BasicTarget, options?: Options) {
  const [isFocusWithin, setIsFocusWithin] = useState(false)
  const { onFocus, onBlur, onChange } = options || {}
  // 监听 focusin 事件
  useEventListener(
    'focusin',
    (e: FocusEvent) => {
      if (!isFocusWithin) {
        onFocus?.(e)
        onChange?.(true)
        setIsFocusWithin(true)
      }
    },
    {
      target,
    }
  )

  // 监听 focusout 事件
  useEventListener(
    'focusout',
    (e: FocusEvent) => {
      // relatedTarget 属性返回与触发鼠标事件的元素相关的元素。
      // https://developer.mozilla.org/zh-CN/docs/Web/API/MouseEvent/relatedTarget
      if (isFocusWithin && !(e.currentTarget as Element)?.contains?.(e.relatedTarget as Element)) {
        onBlur?.(e)
        onChange?.(false)
        setIsFocusWithin(false)
      }
    },
    {
      target,
    }
  )

  return isFocusWithin // 焦点是否在当前区域
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useFocusWithin/index.ts)

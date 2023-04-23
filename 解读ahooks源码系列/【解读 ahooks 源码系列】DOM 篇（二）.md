# 【解读 ahooks 源码系列】DOM 篇（二）

## 前言

本文是 ahooks 源码系列的第三篇，往期文章：

- [【解读 ahooks 源码系列】（开篇）如何获取和监听 DOM 元素](https://juejin.cn/post/7201889983170592824)
- [【解读 ahooks 源码系列】DOM 篇（一）](https://juejin.cn/post/7202254039215800375)

本文主要解读 `useEventTarget`、`useExternal`、`useTitle`、`useFavicon`、`useFullscreen`、`useHover` 源码实现

## useEventTarget

常见表单控件(通过 e.target.value 获取表单值) 的 onChange 跟 value 逻辑封装，支持自定义值转换和重置功能。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-event-target)

```ts
export interface Options<T, U> {
  initialValue?: T // 初始值
  transformer?: (value: U) => T // 自定义回调值的转化
}
```

### 基本用法

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2090254caac64d8fa62f115dadb83ce0~tplv-k3u1fbpfcp-watermark.image?)

```ts
import React from 'react'
import { useEventTarget } from 'ahooks'

export default () => {
  const [value, { reset, onChange }] = useEventTarget({ initialValue: 'this is initial value' })

  return (
    <div>
      <input value={value} onChange={onChange} style={{ width: 200, marginRight: 20 }} />
      <button type="button" onClick={reset}>
        reset
      </button>
    </div>
  )
}
```

### 使用场景

适用于较为简单的表单受控控件（如 input 输入框）管理

### 实现思路

1. 监听表单的 onChange 事件，拿到值后更新 value 值
2. 支持自定义回调值的转化，对外暴露 value 值、onChange 和 reset 方法

### 核心实现

这个实现比较简单，这里结尾代码有个`as const`，它表示强制 TypeScript 将变量或表达式的类型视为不可变的

具体可以看下这篇文章： [杀手级的 TypeScript 功能：const 断言](https://juejin.cn/post/6844903848939634696)

```ts
function useEventTarget<T, U = T>(options?: Options<T, U>) {
  const { initialValue, transformer } = options || {}
  const [value, setValue] = useState(initialValue)

  const transformerRef = useLatest(transformer)

  const reset = useCallback(() => setValue(initialValue), [])

  const onChange = useCallback((e: EventTarget<U>) => {
    const _value = e.target.value
    if (isFunction(transformerRef.current)) {
      return setValue(transformerRef.current(_value))
    }
    // no transformer => U and T should be the same
    return setValue(_value as unknown as T)
  }, [])

  return [
    value,
    {
      onChange,
      reset,
    },
  ] as const // 将数组变为只读元组，可以确保其内容不会在其声明和函数调用之间发生变化
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useEventTarget/index.ts)

## useExternal

动态注入 JS 或 CSS 资源，useExternal 可以保证资源全局唯一。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-external)

### 基本用法

```ts
import React from 'react'
import { useExternal } from 'ahooks'

export default () => {
  const status = useExternal('/useExternal/test-external-script.js', {
    js: {
      async: true,
    },
  })

  return (
    <>
      <p>
        Status: <b>{status}</b>
      </p>
      <p>
        Response: <i>{status === 'ready' ? window.TEST_SCRIPT?.start() : '-'}</i>
      </p>
    </>
  )
}
```

### 实现思路

原理：通过 script 标签加载 JS 资源 / 创建 link 标签加载 CSS 资源，再通过创建标签返回的 Element 元素监听 load 和 error 事件 获取加载状态

1. 正则判断传入的路径 path 是 JS 还是 CSS
2. 加载 CSS/JS：创建 link/script 标签传入 path，支持传入 link/script 标签支持的属性，添加到 head/body 中，并返回 Element 元素与加载状态；这里需判断标签路径匹配是否存在，存在则返回上一次结果，以保证资源全局唯一
3. 利用创建标签返回的 Element 元素监听 load 和 error 事件，并在回调中改变加载状态

### 核心实现

主体实现结构：

```ts
export interface Options {
  type?: 'js' | 'css'
  js?: Partial<HTMLScriptElement>
  css?: Partial<HTMLStyleElement>
}

const useExternal = (path?: string, options?: Options) => {
  const [status, setStatus] = useState<Status>(path ? 'loading' : 'unset')

  const ref = useRef<Element>()

  useEffect(() => {
    if (!path) {
      setStatus('unset')
      return
    }
    const pathname = path.replace(/[|#].*$/, '')
    if (options?.type === 'css' || (!options?.type && /(^css!|\.css$)/.test(pathname))) {
      const result = loadCss(path, options?.css)
    } else if (options?.type === 'js' || (!options?.type && /(^js!|\.js$)/.test(pathname))) {
      const result = loadScript(path, options?.js)
    } else {
    }

    if (!ref.current) {
      return
    }

    const handler = (event: Event) => {}

    ref.current.addEventListener('load', handler)
    ref.current.addEventListener('error', handler)
    return () => {
      // 移除监听 & 清除操作
    }
  }, [path])

  return status
}
```

主函数中判断加载 CSS 还是 JS 资源：

```ts
const pathname = path.replace(/[|#].*$/, '')
if (options?.type === 'css' || (!options?.type && /(^css!|\.css$)/.test(pathname))) {
  const result = loadCss(path, options?.css) // 加载 css 资源并返回结果
  ref.current = result.ref // 返回创建 link 标签返回的 Element 元素，用于后续绑定监听 load 和 error事件
  setStatus(result.status) // 设置加载状态
} else if (options?.type === 'js' || (!options?.type && /(^js!|\.js$)/.test(pathname))) {
  const result = loadScript(path, options?.js)
  ref.current = result.ref
  setStatus(result.status)
} else {
  // do nothing
  console.error(
    "Cannot infer the type of external resource, and please provide a type ('js' | 'css'). " +
      'Refer to the https://ahooks.js.org/hooks/dom/use-external/#options'
  )
}
```

loadCss 方法：

> 往 HTML 标签上添加任意以 "data-" 为前缀来设置我们需要的自定义属性，可以进行一些数据的存放

```ts
const loadCss = (path: string, props = {}): loadResult => {
  const css = document.querySelector(`link[href="${path}"]`)
  // 不存在则创建
  if (!css) {
    const newCss = document.createElement('link')

    newCss.rel = 'stylesheet'
    newCss.href = path
    // 设置 link 标签支持的属性
    Object.keys(props).forEach((key) => {
      newCss[key] = props[key]
    })
    // IE9+
    const isLegacyIECss = 'hideFocus' in newCss
    // use preload in IE Edge (to detect load errors)
    if (isLegacyIECss && newCss.relList) {
      newCss.rel = 'preload'
      newCss.as = 'style'
    }
    // 设置自定义属性[data-status]为loading状态
    newCss.setAttribute('data-status', 'loading')
    // 添加到 head 标签
    document.head.appendChild(newCss)

    // 标签路径匹配存在则直接返回现有结果，保证全局资源全局唯一
    return {
      ref: newCss,
      status: 'loading',
    }
  }
  // 如果标签存在则直接返回，并取 data-status 中的值
  return {
    ref: css,
    status: (css.getAttribute('data-status') as Status) || 'ready',
  }
}
```

loadScript 方法的实现也类似：

```ts
const loadScript = (path: string, props = {}): loadResult => {
  const script = document.querySelector(`script[src="${path}"]`)

  if (!script) {
    const newScript = document.createElement('script')
    newScript.src = path
    // 设置 script 标签支持的属性
    Object.keys(props).forEach((key) => {
      newScript[key] = props[key]
    })

    newScript.setAttribute('data-status', 'loading')
    // 添加到 body 标签
    document.body.appendChild(newScript)

    return {
      ref: newScript,
      status: 'loading',
    }
  }

  return {
    ref: script,
    status: (script.getAttribute('data-status') as Status) || 'ready',
  }
}
```

前面获取到 Element 元素后，监听 Element 的 load 和 error 事件，判断其加载状态并更新状态

```ts
const handler = (event: Event) => {
  const targetStatus = event.type === 'load' ? 'ready' : 'error'
  ref.current?.setAttribute('data-status', targetStatus)
  setStatus(targetStatus)
}

ref.current.addEventListener('load', handler)
ref.current.addEventListener('error', handler)
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useExternal/index.ts)

## useTitle

用于设置页面标题。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-title)

### 基本用法

```ts
import React from 'react'
import { useTitle } from 'ahooks'

export default () => {
  useTitle('Page Title')

  return (
    <div>
      <p>Set title of the page.</p>
    </div>
  )
}
```

### 使用场景

当进入某页面需要改浏览器 Tab 中展示的标题时

### 核心实现

这个实现比较简单

```ts
const DEFAULT_OPTIONS: Options = {
  restoreOnUnmount: false, // 组件卸载时，是否恢复上一个页面标题
}

function useTitle(title: string, options: Options = DEFAULT_OPTIONS) {
  const titleRef = useRef(isBrowser ? document.title : '')
  useEffect(() => {
    document.title = title
  }, [title])

  useUnmount(() => {
    if (options.restoreOnUnmount) {
      // 组件卸载时，恢复上一个页面标题
      document.title = titleRef.current
    }
  })
}
```

如果项目中我们自己实现的话，有个需要注意的地方，不要把`document.title = title;`写在外层，要写在 useEffect 里面，具体见该文：[检测意外的副作用](https://juejin.cn/post/6854573210663387149)

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useTitle/index.ts)

## useFavicon

设置页面的 favicon。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-favicon)

> favicon 指显示在浏览器收藏夹、地址栏和标签标题前面的个性化图标

### 基本用法

```ts
import React, { useState } from 'react'
import { useFavicon } from 'ahooks'

export const DEFAULT_FAVICON_URL = 'https://ahooks.js.org/simple-logo.svg'

export const GOOGLE_FAVICON_URL = 'https://www.google.com/favicon.ico'

export default () => {
  const [url, setUrl] = useState<string>(DEFAULT_FAVICON_URL)

  useFavicon(url)

  return (
    <>
      <p>
        Current Favicon: <span>{url}</span>
      </p>
      <button
        style={{ marginRight: 16 }}
        onClick={() => {
          setUrl(GOOGLE_FAVICON_URL)
        }}
      >
        Change To Google Favicon
      </button>
      <button
        onClick={() => {
          setUrl(DEFAULT_FAVICON_URL)
        }}
      >
        Back To AHooks Favicon
      </button>
    </>
  )
}
```

### 使用场景

当需要改浏览器 Tab 中展示的图标 icon 时

### 核心实现

原理：通过 link 标签设置 favicon

更多 favicon 知识可见： [详细介绍 HTML favicon 尺寸 格式 制作等相关知识](https://www.zhangxinxu.com/wordpress/2019/06/html-favicon-size-ico-generator/)

源代码仅支持图标四种类型：

```ts
const ImgTypeMap = {
  SVG: 'image/svg+xml',
  ICO: 'image/x-icon',
  GIF: 'image/gif',
  PNG: 'image/png',
}

type ImgTypes = keyof typeof ImgTypeMap
```

```ts
const useFavicon = (href: string) => {
  useEffect(() => {
    if (!href) return

    const cutUrl = href.split('.')
    // 取出文件后缀
    const imgSuffix = cutUrl[cutUrl.length - 1].toLocaleUpperCase() as ImgTypes

    const link: HTMLLinkElement = document.querySelector("link[rel*='icon']") || document.createElement('link')

    link.type = ImgTypeMap[imgSuffix]
    // 指定被链接资源的地址
    link.href = href
    // rel 属性用于指定当前文档与被链接文档的关系，直接使用 rel=icon 就可以，源码下方的 `shortcut icon` 是一种过时的用法
    link.rel = 'shortcut icon'

    document.getElementsByTagName('head')[0].appendChild(link)
  }, [href])
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useFavicon/index.ts)

## useFullscreen

管理 DOM 全屏的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-fullscreen)

### 基本用法

```ts
import React, { useRef } from 'react'
import { useFullscreen } from 'ahooks'

export default () => {
  const ref = useRef(null)
  const [isFullscreen, { enterFullscreen, exitFullscreen, toggleFullscreen }] = useFullscreen(ref)
  return (
    <div ref={ref} style={{ background: 'white' }}>
      <div style={{ marginBottom: 16 }}>{isFullscreen ? 'Fullscreen' : 'Not fullscreen'}</div>
      <div>
        <button type="button" onClick={enterFullscreen}>
          enterFullscreen
        </button>
        <button type="button" onClick={exitFullscreen} style={{ margin: '0 8px' }}>
          exitFullscreen
        </button>
        <button type="button" onClick={toggleFullscreen}>
          toggleFullscreen
        </button>
      </div>
    </div>
  )
}
```

### 原生全屏 API

- [Element.requestFullscreen()](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/requestFullscreen)：用于发出异步请求使元素进入全屏模式
- [Document.exitFullscreen()](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/exitFullscreen)：用于让当前文档退出全屏模式。调用这个方法会让文档回退到上一个调用 Element.requestFullscreen()方法进入全屏模式之前的状态
- ~~[已过时不建议使用]：[Document.fullscreen](https://developer.mozilla.org/en-US/docs/Web/API/Document/fullscreen)：只读属性报告文档当前是否以全屏模式显示内容~~
- [Document.fullscreenElement](https://developer.mozilla.org/en-US/docs/Web/API/Document/fullscreenElement)：返回当前文档中正在以全屏模式显示的 Element 节点，如果没有使用全屏模式，则返回 null
- [Document.fullscreenEnabled](https://developer.mozilla.org/en-US/docs/Web/API/Document/fullscreenEnabled)：返回一个布尔值，表明浏览器是否支持全屏模式。全屏模式只在那些不包含窗口化的插件的页面中可用
- [fullscreenchange](https://developer.mozilla.org/en-US/docs/Web/API/Element/fullscreenchange_event)：元素过渡到或过渡到全屏模式时触发的全屏更改事件的事件
- [fullscreenerror](https://developer.mozilla.org/en-US/docs/Web/API/Element/fullscreenerror_event)：在 Element 过渡到或退出全屏模式发生错误后处理事件

### screenfull 库

useFullscreen 内部主要是依赖 [screenfull](https://github.com/sindresorhus/screenfull) 这个库进行实现的。

screenfull 对各种浏览器全屏的 API 进行封装，兼容性好。

下面是该库的 API：

- .request(element, options?)：使元素或者页面切换到全屏
- .exit()：退出全屏
- .toggle(element, options?)：在全屏和非全屏之间切换
- .on(event, function)：添加一个监听器，监听全屏切换或者错误事件。event 支持 `change` 或者 `error`
- .off(event, function)：移除之前注册的事件监听
- .isFullscreen：判断是否为全屏
- .isEnabled：判断当前环境是否支持全屏
- .element：返回该元素是否是全屏模式展示，否则返回 undefined

### 实现思路

看看 `useFullscreen` 的导出值：

```ts
return [
  state,
  {
    enterFullscreen: useMemoizedFn(enterFullscreen),
    exitFullscreen: useMemoizedFn(exitFullscreen),
    toggleFullscreen: useMemoizedFn(toggleFullscreen),
    isEnabled: screenfull.isEnabled,
  },
] as const
```

那么实现的方向就比较简单了：

1. 内部封装并暴露 toggleFullscreen、enterFullscreen、exitFullscreen 方法，暴露内部是否全屏的状态，还有是否支持全屏的状态
2. 通过 screenfull 库监听`change`事件，在`change`事件里面改变全屏状态与处理执行回调

### 核心实现

三个方法的实现：

```ts
// 进入全屏方法
const enterFullscreen = () => {
  const el = getTargetElement(target)
  if (!el) {
    return
  }

  if (screenfull.isEnabled) {
    try {
      screenfull.request(el)
      screenfull.on('change', onChange)
    } catch (error) {
      console.error(error)
    }
  }
}

// 退出全屏方法
const exitFullscreen = () => {
  const el = getTargetElement(target)
  if (screenfull.isEnabled && screenfull.element === el) {
    screenfull.exit()
  }
}

const toggleFullscreen = () => {
  if (state) {
    exitFullscreen()
  } else {
    enterFullscreen()
  }
}
```

onChange 方法

```ts
const onChange = () => {
  if (screenfull.isEnabled) {
    const el = getTargetElement(target)
    // screenfull.element：当前元素以全屏模式显示
    if (!screenfull.element) {
      // 退出全屏
      onExitRef.current?.()
      setState(false)
      screenfull.off('change', onChange) // 卸载 change 事件
    } else {
      // 全屏模式展示
      const isFullscreen = screenfull.element === el // 判断当前全屏元素是否为目标元素
      if (isFullscreen) {
        onEnterRef.current?.()
      } else {
        onExitRef.current?.()
      }
      setState(isFullscreen)
    }
  }
}
```

上方`onChange`以及`exitFullscreen`执行退出全屏前有行需要判断的代码注意下，具体原因可以看下[修复 useFullScreen 当全屏后，子元素重复全屏和退出全屏操作后父元素也会退出全屏](https://github.com/alibaba/hooks/pull/1862)

```ts
// 判断当前全屏元素是否为目标元素，支持对多个元素同时全屏
const isFullscreen = screenfull.element === el
```

screenfull.element 的实现：

```js
element: {
  enumerable: true,
  get: () => document[nativeAPI.fullscreenElement] ?? undefined,
},
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useFullscreen/index.ts)

## useHover

监听 DOM 元素是否有鼠标悬停。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-hover)

### 基本用法

```ts
import React, { useRef } from 'react'
import { useHover } from 'ahooks'

export default () => {
  const ref = useRef(null)
  const isHovering = useHover(ref)
  return <div ref={ref}>{isHovering ? 'hover' : 'leaveHover'}</div>
}
```

### 鼠标监听事件

- [mouseenter](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/mouseenter_event)：第一次移动到触发事件元素中的激活区域时触发
- [mouseleave](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/mouseleave_event)：在定点设备（通常是鼠标）的指针移出某个元素时被触发

扩展下几个鼠标事件的区别：

- mouseenter：当鼠标移入某元素时触发。
- mouseleave：当鼠标移出某元素时触发。
- mouseover：当鼠标移入某元素时触发，移入和移出其子元素时也会触发。
- mouseout：当鼠标移出某元素时触发，移入和移出其子元素时也会触发。
- mousemove：鼠标在某元素上移动时触发，即使在其子元素上也会触发。

### 核心实现

原理是监听 `mouseenter` 触发 `onEnter` 回调，切换状态为 true；监听 `mouseleave` 触发 `onLeave`回调，切换状态为 false。

完整实现：

```ts
export interface Options {
  onEnter?: () => void
  onLeave?: () => void
  onChange?: (isHovering: boolean) => void
}

export default (target: BasicTarget, options?: Options): boolean => {
  const { onEnter, onLeave, onChange } = options || {}

  // useBoolean：优雅的管理 boolean 状态的 Hook
  const [state, { setTrue, setFalse }] = useBoolean(false)

  // 监听 mouseenter 判断有鼠标进入目标元素
  useEventListener(
    'mouseenter',
    () => {
      onEnter?.()
      setTrue()
      onChange?.(true)
    },
    {
      target,
    }
  )

  // 监听 mouseleave 判断有鼠标是否移出目标元素
  useEventListener(
    'mouseleave',
    () => {
      onLeave?.()
      setFalse()
      onChange?.(false)
    },
    {
      target,
    }
  )

  return state
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useHover/index.ts)

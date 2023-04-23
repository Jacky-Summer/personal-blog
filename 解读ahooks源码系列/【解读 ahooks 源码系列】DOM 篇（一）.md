# 【解读 ahooks 源码系列】DOM 篇（一）

## 前言

本文是 ahooks 源码系列的第二篇，下面链接是第一篇 DOM 篇的前置讲解：

- [【解读 ahooks 源码系列】（开篇）如何获取和监听 DOM 元素](https://juejin.cn/post/7201889983170592824)

后续的文章将会直入主题，每篇文章解读四至六个 Hooks 源码实现。

## useEventListener

优雅的使用 addEventListener。

- [官方文档](https://ahooks.js.org/zh-CN/hooks/use-event-listener)

### 用法

```ts
import React, { useState, useRef } from 'react'
import { useEventListener } from 'ahooks'

export default () => {
  const [value, setValue] = useState(0)
  const ref = useRef(null)

  useEventListener(
    'click',
    () => {
      setValue(value + 1)
    },
    { target: ref }
  )

  return (
    <button ref={ref} type="button">
      You click {value} times
    </button>
  )
}
```

### 使用场景

通用事件监听 Hook，简化写法（无需在 useEffect 卸载函数中手动移除监听函数，由内部去移除）

### 实现思路

1. 判断是否支持 addEventListener
2. 在单独只有 useEffect 实现事件监听移除的基础上，将相关参数都由外部传入，并添加到依赖项
3. 处理事件参数的 TS 类型，addEventListener 的第三个参数也需要由外部传入

### 核心实现

- [EventTarget.addEventListener()](https://developer.mozilla.org/zh-CN/docs/Web/API/EventTarget/addEventListener)：将指定的监听器注册到 EventTarget 上，当该对象触发指定的事件时，指定的回调函数就会被执行

EventTarget 指任何其他支持事件的对象/元素 `HTMLElement | Element | Document | Window`

符合 EventTarget 接口的都具有下列三个方法

```js
EventTarget.addEventListener()
EventTarget.removeEventListener()
EventTarget.dispatchEvent()
```

- TS 函数重载

> 函数重载指使用相同名称和不同参数数量或类型创建多个方法，让我们定义以多种方式调用的函数。在 TS 中为同一个函数提供多个函数类型定义来进行函数重载

```ts
function useEventListener<K extends keyof HTMLElementEventMap>(
  eventName: K,
  handler: (ev: HTMLElementEventMap[K]) => void,
  options?: Options<HTMLElement>
): void
function useEventListener<K extends keyof ElementEventMap>(
  eventName: K,
  handler: (ev: ElementEventMap[K]) => void,
  options?: Options<Element>
): void
function useEventListener<K extends keyof DocumentEventMap>(
  eventName: K,
  handler: (ev: DocumentEventMap[K]) => void,
  options?: Options<Document>
): void
function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (ev: WindowEventMap[K]) => void,
  options?: Options<Window>
): void
```

实现：

```ts
function useEventListener(eventName: string, handler: noop, options: Options = {}) {
  const handlerRef = useLatest(handler)

  useEffectWithTarget(
    () => {
      const targetElement = getTargetElement(options.target, window)
      if (!targetElement?.addEventListener) {
        return
      }

      const eventListener = (event: Event) => {
        return handlerRef.current(event)
      }

      // 添加监听事件
      targetElement.addEventListener(eventName, eventListener, {
        // true 表示事件在捕获阶段执行，false（默认） 表示事件在冒泡阶段执行
        capture: options.capture,
        // true 表示事件在触发一次后移除，默认 false
        once: options.once,
        // true 表示 listener 永远不会调用 preventDefault()。如果 listener 仍然调用了这个函数，客户端将会忽略它并抛出一个控制台警告
        passive: options.passive,
      })

      // 移除监听事件
      return () => {
        targetElement.removeEventListener(eventName, eventListener, {
          capture: options.capture,
        })
      }
    },
    [eventName, options.capture, options.once, options.passive],
    options.target
  )
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useEventListener/index.ts)

## useClickAway

监听目标元素外的点击事件。

- [官方文档](https://ahooks.js.org/zh-CN/hooks/use-click-away)

```ts
type Target = Element | (() => Element) | React.MutableRefObject<Element>;

/**
 * 监听目标元素外的点击事件。
 * @param onClickAway 触发函数
 * @param target DOM 节点或者 Ref，支持数组
 * @param eventName DOM 节点或者 Ref，支持数组，默认事件是 click
 */
useClickAway<T extends Event = Event>(
  onClickAway: (event: T) => void,
  target: Target | Target[],
  eventName?: string | string[]
);
```

### 用法

```ts
import React, { useState, useRef } from 'react'
import { useClickAway } from 'ahooks'

export default () => {
  const [counter, setCounter] = useState(0)
  const ref = useRef<HTMLButtonElement>(null)
  useClickAway(() => {
    setCounter((s) => s + 1)
  }, ref)

  return (
    <div>
      <button ref={ref} type="button">
        box
      </button>
      <p>counter: {counter}</p>
    </div>
  )
}
```

### 使用场景

比如点击显示弹窗之后，此时点击弹窗之外的任意区域时（如弹窗的全局蒙层），该弹窗要自动隐藏。简而言之，属于"点击页面其他元素，XX 组件自动关闭"的功能。

### 实现思路

1. 在 document 上绑定全局事件。如默认支持点击事件，组件卸载的时候移除事件监听
2. 触发事件后，可通过**事件代理**获取到触发事件的对象的引用 e，如果该目标元素 e.target 不在外部传入的 target 元素(列表)中，则触发 onClickAway 函数

### 核心实现

假如只支持点击事件，只能传单个元素且只能是 Ref 类型，实现代码如下：

```js
export default function useClickAway<T extends HTMLElement>(
  onClickAway: (event: MouseEvent) => void,
  refObject: React.RefObject<T>,
) {
  useEffect(() => {
    const handleClick = (e: MouseEvent) => {
      if (
        !refObject.current ||
        refObject.current.contains(e.target as HTMLElement)
      ) {
        return
      }
      onClickAway(e)
    }

    document.addEventListener('click', handleClick)

    return () => {
      document.removeEventListener('click', handleClick)
    }
  }, [refObject, onClickAway])
}
```

ahooks 则继续拓展，思路如下：

1. 同时支持传入 DOM 节点、Ref：需要区分是 DOM 节点、函数、还是 Ref，获取的时候要兼顾所有情况
2. 可传入多个目标元素（支持数组）：通过循环绑定事件，用数组 some 方法判断任一元素包含则触发
3. 可指定监听事件（支持数组）：eventName 由外部传入，不传默认为 click 事件

来看看源码整体实现：

第 1、2 点的实现

```ts
// documentOrShadow 这部分忽略不深究，一般开发场景就是 document
const documentOrShadow = getDocumentOrShadow(target)

const eventNames = Array.isArray(eventName) ? eventName : [eventName]
// 循环绑定事件
eventNames.forEach((event) => documentOrShadow.addEventListener(event, handler))

return () => {
  eventNames.forEach((event) => documentOrShadow.removeEventListener(event, handler))
}
```

第 3 点 handler 函数的实现：

```ts
const handler = (event: any) => {
  const targets = Array.isArray(target) ? target : [target]
  if (
    // 判断点击的目标元素是否在外部传入的元素（列表）中，是则 return 不执行回调
    targets.some((item) => {
      const targetElement = getTargetElement(item) // 这里处理了传入的target是函数、DOM节点、Ref 类型的情况
      return !targetElement || targetElement.contains(event.target)
    })
  ) {
    return
  }
  // 触发事件
  onClickAwayRef.current(event)
}
```

1. 这里注意触发事件的代码是：`onClickAwayRef.current(event);`，实际是为了保证能拿到最新的函数，可以避免闭包问题

```js
const onClickAwayRef = useLatest(onClickAway)

// 等同于
const onClickAwayRef = useRef(onClickAway)
onClickAwayRef.current = onClickAway
```

2. `getTargetElement` 方法获取目标元素实现如下：

```js
if (isFunction(target)) {
  targetElement = target()
} else if ('current' in target) {
  targetElement = target.current
} else {
  targetElement = target
}
```

- [完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useClickAway/index.ts)

### 注意 React17+版本的坑

Reactv17 前，React 将事件委托到 document 上，在 Reactv17 及之后版本，则委托到根节点，具体见该文：

- [ahooks 的 useClickAway 在 React 17 中不工作了！](https://juejin.cn/post/7110470986419404813)

解决方案是给 useClickAway 的事件类型设置为 mousedown 和 touchstart

在写这篇文章的时候，还没更新：
具体可见 [useClickAway 判断不对](https://github.com/alibaba/hooks/issues/1641)

### 其它写法实现参考

总体来说 ahooks 的实现功能更齐全考虑的场景更多，但业务开发如果是自己写 Hooks 实现的话，推荐下面的写法，足以覆盖日常开发场景：

- [react-use 的 useClickAway](https://github.com/streamich/react-use/blob/master/src/useClickAway.ts)
- [useHooks 的 useOnClickOutside](https://usehooks.com/useOnClickOutside/)

## useDocumentVisibility

监听页面是否可见。

- [官方文档](https://ahooks.js.org/zh-CN/hooks/use-document-visibility)

### 用法

```ts
import React, { useEffect } from 'react'
import { useDocumentVisibility } from 'ahooks'

export default () => {
  const documentVisibility = useDocumentVisibility()

  useEffect(() => {
    console.log(`Current document visibility state: ${documentVisibility}`)
  }, [documentVisibility])

  return <div>Current document visibility state: {documentVisibility}</div>
}
```

### 使用场景

当页面在背景中或窗口最小化时禁止或开启某些活动，如离开页面停止播放音视频、暂停轮询接口请求

### 实现思路

1. 定义并暴露给外部`document.visibilityState`状态值，通过该字段判断页面是否可见
2. 监听 `visibilitychange` 事件（使用 document 注册），触发回调时更新状态值

### Document.visibilityState 与 visibilitychange 事件

**[Document.visibilityState](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/visibilityState)**（只读属性）

返回 document 的可见性，即当前可见元素的上下文环境。由此可以知道当前文档 (即为页面) 是在背后，或是不可见的隐藏的标签页，或者 (正在) 预渲染，共有三个可能的值。

- visible: 此时页面内容至少是部分可见。即此页面在前景标签页中，并且窗口没有最小化。
- hidden: 此时页面对用户不可见。即文档处于背景标签页或者窗口处于最小化状态，或者操作系统正处于 '锁屏状态' .
- prerender: 页面此时正在渲染中，因此是不可见的。文档只能从此状态开始，永远不能从其他值变为此状态。（prerender 状态只在支持"预渲染"的浏览器上才会出现）。

---

**[visibilitychange](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/visibilitychange_event)**

当其选项卡的内容变得可见或被隐藏时，会在文档上触发 visibilitychange (能见度更改) 事件。

> 警告： 出于兼容性原因，请确保使用 document.addEventListener 而不是 window.addEventListener 来注册回调。Safari <14.0 仅支持前者。

推荐阅读：[Page Visibility API 教程](http://www.ruanyifeng.com/blog/2018/10/page_visibility_api.html)

### 核心实现

```ts
type VisibilityState = 'hidden' | 'visible' | 'prerender' | undefined

const getVisibility = () => {
  if (!isBrowser) {
    return 'visible'
  }
  // 返回document的可见性，即当前可见元素的上下文环境
  return document.visibilityState
}

function useDocumentVisibility(): VisibilityState {
  const [documentVisibility, setDocumentVisibility] = useState(() => getVisibility())

  // 监听事件
  useEventListener(
    'visibilitychange',
    () => {
      setDocumentVisibility(getVisibility())
    },
    {
      target: () => document,
    }
  )

  return documentVisibility
}

export default useDocumentVisibility
```

- [完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useDocumentVisibility/index.ts)

## useDrop

处理元素拖拽的 Hook。

- [官方文档](https://ahooks.js.org/zh-CN/hooks/use-drop)

### 用法

```ts
import React, { useRef, useState } from 'react'
import { useDrop, useDrag } from 'ahooks'

const DragItem = ({ data }) => {
  const dragRef = useRef(null)

  const [dragging, setDragging] = useState(false)

  useDrag(data, dragRef, {
    onDragStart: () => {
      setDragging(true)
    },
    onDragEnd: () => {
      setDragging(false)
    },
  })

  return (
    <div
      ref={dragRef}
      style={{
        border: '1px solid #e8e8e8',
        padding: 16,
        width: 80,
        textAlign: 'center',
        marginRight: 16,
      }}
    >
      {dragging ? 'dragging' : `box-${data}`}
    </div>
  )
}

export default () => {
  const [isHovering, setIsHovering] = useState(false)

  const dropRef = useRef(null)

  useDrop(dropRef, {
    onText: (text, e) => {
      console.log(e)
      alert(`'text: ${text}' dropped`)
    },
    onFiles: (files, e) => {
      console.log(e, files)
      alert(`${files.length} file dropped`)
    },
    onUri: (uri, e) => {
      console.log(e)
      alert(`uri: ${uri} dropped`)
    },
    onDom: (content: string, e) => {
      alert(`custom: ${content} dropped`)
    },
    onDragEnter: () => setIsHovering(true),
    onDragLeave: () => setIsHovering(false),
  })

  return (
    <div>
      <div ref={dropRef} style={{ border: '1px dashed #e8e8e8', padding: 16, textAlign: 'center' }}>
        {isHovering ? 'release here' : 'drop here'}
      </div>

      <div style={{ display: 'flex', marginTop: 8 }}>
        {['1', '2', '3', '4', '5'].map((e, i) => (
          <DragItem key={e} data={e} />
        ))}
      </div>
    </div>
  )
}
```

### 使用场景

- useDrop 可以单独使用来接收文件、文字和网址的拖拽。
- 向节点内触发粘贴动作也会被视为拖拽

### 涉及的拖拽 API

拖拽相关事件：

- [dragenter](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/dragenter_event)：事件在可拖动的元素或者被选择的文本进入一个有效的放置目标时触发。
- [dragleave](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/dragleave_event)：在拖动的元素或选中的文本离开一个有效的放置目标时被触发。
- [dragover](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/dragover_event)：在可拖动的元素或者被选择的文本被拖进一个有效的放置目标时（每几百毫秒）触发。
- [drop](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/drop_event)：当一个元素或是选中的文字被拖拽释放到一个有效的释放目标位置时，drop 事件被抛出。
- [paste](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/paste_event)：当用户在浏览器用户界面发起“粘贴”操作时，会触发 paste 事件。

### 实现思路

1. 监听以上 5 个事件
2. 另外在 drop 和 paste 事件中获取到 DataTransfer 数据，并根据数据类型进行特定的处理，将处理好的数据通过回调（onText/onFiles/onUri/onDom）给外部直接获取使用。

```ts
export interface Options {
  // 根据 drop 事件数据类型自定义回调函数
  onFiles?: (files: File[], event?: React.DragEvent) => void
  onUri?: (url: string, event?: React.DragEvent) => void
  onDom?: (content: any, event?: React.DragEvent) => void
  onText?: (text: string, event?: React.ClipboardEvent) => void
  // 原生事件
  onDragEnter?: (event?: React.DragEvent) => void
  onDragOver?: (event?: React.DragEvent) => void
  onDragLeave?: (event?: React.DragEvent) => void
  onDrop?: (event?: React.DragEvent) => void
  onPaste?: (event?: React.ClipboardEvent) => void
}

const useDrop = (target: BasicTarget, options: Options = {}) => {}
```

### 核心实现

主函数实现比较简单，需要注意的时候在特定事件需要阻止默认事件`event.preventDefault();`和阻止事件冒泡`event.stopPropagation();`，让拖拽能正常的工作

```ts
const useDrop = (target: BasicTarget, options: Options = {}) => {
  const optionsRef = useLatest(options)

  // https://stackoverflow.com/a/26459269
  const dragEnterTarget = useRef<any>()

  useEffectWithTarget(
    () => {
      const targetElement = getTargetElement(target)
      if (!targetElement?.addEventListener) {
        return
      }

      // 处理 DataTransfer 不同数据类型数据
      const onData = (dataTransfer: DataTransfer, event: React.DragEvent | React.ClipboardEvent) => {}

      const onDragEnter = (event: React.DragEvent) => {
        event.preventDefault()
        event.stopPropagation()

        dragEnterTarget.current = event.target
        optionsRef.current.onDragEnter?.(event)
      }

      const onDragOver = (event: React.DragEvent) => {
        event.preventDefault() // 调用 event.preventDefault() 使得该元素能够接收 drop 事件
        optionsRef.current.onDragOver?.(event)
      }

      const onDragLeave = (event: React.DragEvent) => {
        if (event.target === dragEnterTarget.current) {
          optionsRef.current.onDragLeave?.(event)
        }
      }

      const onDrop = (event: React.DragEvent) => {
        event.preventDefault()
        onData(event.dataTransfer, event)
        optionsRef.current.onDrop?.(event)
      }

      const onPaste = (event: React.ClipboardEvent) => {
        onData(event.clipboardData, event)
        optionsRef.current.onPaste?.(event)
      }

      targetElement.addEventListener('dragenter', onDragEnter as any)
      targetElement.addEventListener('dragover', onDragOver as any)
      targetElement.addEventListener('dragleave', onDragLeave as any)
      targetElement.addEventListener('drop', onDrop as any)
      targetElement.addEventListener('paste', onPaste as any)

      return () => {
        targetElement.removeEventListener('dragenter', onDragEnter as any)
        targetElement.removeEventListener('dragover', onDragOver as any)
        targetElement.removeEventListener('dragleave', onDragLeave as any)
        targetElement.removeEventListener('drop', onDrop as any)
        targetElement.removeEventListener('paste', onPaste as any)
      }
    },
    [],
    target
  )
}
```

在 drop 和 paste 事件中，获取到 DataTransfer 数据并传给 onData 方法，根据数据类型进行特定的处理

- [DataTransfer](https://developer.mozilla.org/zh-CN/docs/Web/API/DataTransfer)：DataTransfer 对象用于保存拖动并放下（drag and drop）过程中的数据。它可以保存一项或多项数据，这些数据项可以是一种或者多种数据类型。关于拖放的更多信息，请参见 [Drag and Drop](https://developer.mozilla.org/zh-CN/docs/Web/API/HTML_Drag_and_Drop_API)
- [DataTransfer.getData()](https://developer.mozilla.org/zh-CN/docs/Web/API/DataTransfer/getData)接受指定类型的拖放（以 DOMString 的形式）数据。如果拖放行为没有操作任何数据，会返回一个空字符串。数据类型有：text/plain，text/uri-list
- [DataTransferItem](https://developer.mozilla.org/zh-CN/docs/Web/API/DataTransferItem)：拖拽项。

```ts
const onData = (dataTransfer: DataTransfer, event: React.DragEvent | React.ClipboardEvent) => {
  const uri = dataTransfer.getData('text/uri-list') // URL格式列表（链接）
  const dom = dataTransfer.getData('custom') // 自定义数据，需要与 useDrag 搭配使用

  // 根据数据类型进行特定的处理
  // 拖拽/粘贴自定义 DOM 节点的回调
  if (dom && optionsRef.current.onDom) {
    let data = dom
    try {
      data = JSON.parse(dom)
    } catch (e) {
      data = dom
    }
    optionsRef.current.onDom(data, event as React.DragEvent)
    return
  }

  // 拖拽/粘贴链接的回调
  if (uri && optionsRef.current.onUri) {
    optionsRef.current.onUri(uri, event as React.DragEvent)
    return
  }

  // 拖拽/粘贴文件的回调
  // dataTransfer.files：拖动操作中的文件列表，操作中每个文件的一个列表项。如果拖动操作没有文件，此列表为空
  if (dataTransfer.files && dataTransfer.files.length && optionsRef.current.onFiles) {
    optionsRef.current.onFiles(Array.from(dataTransfer.files), event as React.DragEvent)
    return
  }

  // 拖拽/粘贴文字的回调
  if (dataTransfer.items && dataTransfer.items.length && optionsRef.current.onText) {
    // dataTransfer.items：拖动操作中 数据传输项的列表。该列表包含了操作中每一项目的对应项，如果操作没有项目，则列表为空
    // getAsString：使用拖拽项的字符串作为参数执行指定回调函数
    dataTransfer.items[0].getAsString((text) => {
      optionsRef.current.onText!(text, event as React.ClipboardEvent)
    })
  }
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useDrop/index.ts)

## useDrag

处理元素拖拽的 Hook。

- [官方文档](https://ahooks.js.org/zh-CN/hooks/use-drop)

### 使用场景

useDrag 允许一个 DOM 节点被拖拽，需要配合 useDrop 使用。

### 涉及的拖拽事件

- [dragstart](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/dragstart_event): 在用户开始拖动元素或被选择的文本时调用。
- [dragend](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/dragend_event): 在拖放操作结束时触发（通过释放鼠标按钮或单击 escape 键）。

### 实现思路

1. 内部监听 dragstart 和 dragend 方法触发回调给外部使用
2. dragstart 事件触发时支持设置自定义数据到 dataTransfer 中

### 核心实现

```ts
export interface Options {
  // 在用户开始拖动元素或被选择的文本时调用
  onDragStart?: (event: React.DragEvent) => void
  // 在拖放操作结束时触发（通过释放鼠标按钮或单击 escape 键）
  onDragEnd?: (event: React.DragEvent) => void
}

const useDrag = <T>(data: T, target: BasicTarget, options: Options = {}) => {
  const optionsRef = useLatest(options)
  const dataRef = useLatest(data)
  useEffectWithTarget(
    () => {
      const targetElement = getTargetElement(target)
      if (!targetElement?.addEventListener) {
        return
      }

      const onDragStart = (event: React.DragEvent) => {
        optionsRef.current.onDragStart?.(event)
        // 设置自定义数据到 dataTransfer 中，搭配 useDrop 的 onDom 回调可获取当前设置的内容
        event.dataTransfer.setData('custom', JSON.stringify(dataRef.current))
      }

      const onDragEnd = (event: React.DragEvent) => {
        optionsRef.current.onDragEnd?.(event)
      }

      targetElement.setAttribute('draggable', 'true')

      targetElement.addEventListener('dragstart', onDragStart as any)
      targetElement.addEventListener('dragend', onDragEnd as any)

      return () => {
        targetElement.removeEventListener('dragstart', onDragStart as any)
        targetElement.removeEventListener('dragend', onDragEnd as any)
      }
    },
    [],
    target
  )
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useDrag/index.ts)

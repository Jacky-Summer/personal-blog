# 【解读 ahooks 源码系列】DOM 篇（三）

## 前言

本文是 ahooks 源码系列的第四篇，往期文章：

- [【解读 ahooks 源码系列】（开篇）如何获取和监听 DOM 元素](https://juejin.cn/post/7201889983170592824)：useEffectWithTarget
- [【解读 ahooks 源码系列】DOM 篇（一）](https://juejin.cn/post/7202254039215800375)：useEventListener、useClickAway、useDocumentVisibility、useDrop、useDrag
- [【解读 ahooks 源码系列】DOM 篇（二）](https://juejin.cn/post/7202633255043465271)：useEventTarget、useExternal、useTitle、useFavicon、useFullscreen、useHover

本文主要解读 `useMutationObserver`、`useInViewport`、`useKeyPress`、`useLongPress` 源码实现

## useMutationObserver

一个监听指定的 DOM 树发生变化的 Hook

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-mutation-observer)

### MutationObserver API

MutationObserver 接口提供了监视对 DOM 树所做更改的能力。利用 MutationObserver API 我们可以监视 DOM 的变化，比如节点的增加、减少、属性的变动、文本内容的变动等等。

可参考学习：

- [MutationObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)
- [你不知道的 MutationObserver](http://www.semlinker.com/you-dont-know-mutation-observer/)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/zh-CN/hooks/use-mutation-observer)

点击按钮，改变 width，触发 div 的 width 属性变更，打印的 mutationsList 如下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d820fa1d0b048bdbb77ab07a63e3b97~tplv-k3u1fbpfcp-watermark.image?)

```ts
import { useMutationObserver } from 'ahooks'
import React, { useRef, useState } from 'react'

const App: React.FC = () => {
  const [width, setWidth] = useState(200)
  const [count, setCount] = useState(0)

  const ref = useRef<HTMLDivElement>(null)

  useMutationObserver(
    (mutationsList) => {
      mutationsList.forEach(() => setCount((c) => c + 1))
    },
    ref,
    { attributes: true }
  )

  return (
    <div>
      <div ref={ref} style={{ width, padding: 12, border: '1px solid #000', marginBottom: 8 }}>
        current width：{width}
      </div>
      <button onClick={() => setWidth((w) => w + 10)}>widening</button>
      <p>Mutation count {count}</p>
    </div>
  )
}
```

### 核心实现

这个实现比较简单，主要还是理解 MutationObserver API：

```ts
useMutationObserver(
  callback: MutationCallback, // 触发的回调函数
  target: Target,
  options?: MutationObserverInit, // 设置项：https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver/observe#parameters
);
```

```ts
const useMutationObserver = (
  callback: MutationCallback,
  target: BasicTarget,
  options: MutationObserverInit = {}
): void => {
  const callbackRef = useLatest(callback)

  useDeepCompareEffectWithTarget(
    () => {
      const element = getTargetElement(target)
      if (!element) {
        return
      }
      // 创建一个观察器实例并传入回调函数
      const observer = new MutationObserver(callbackRef.current)
      observer.observe(element, options) // 启动监听，指定所要观察的 DOM 节点
      return () => {
        if (observer) {
          observer.disconnect() // 停止观察变动
        }
      }
    },
    [options],
    target
  )
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useMutationObserver/index.ts)

## useInViewport

观察元素是否在可见区域，以及元素可见比例。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-in-viewport)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/useinviewport-demo1/)

```ts
import React, { useRef } from 'react'
import { useInViewport } from 'ahooks'

export default () => {
  const ref = useRef(null)
  const [inViewport] = useInViewport(ref)
  return (
    <div>
      <div style={{ width: 300, height: 300, overflow: 'scroll', border: '1px solid' }}>
        scroll here
        <div style={{ height: 800 }}>
          <div
            ref={ref}
            style={{
              border: '1px solid',
              height: 100,
              width: 100,
              textAlign: 'center',
              marginTop: 80,
            }}
          >
            observer dom
          </div>
        </div>
      </div>
      <div style={{ marginTop: 16, color: inViewport ? '#87d068' : '#f50' }}>
        inViewport: {inViewport ? 'visible' : 'hidden'}
      </div>
    </div>
  )
}
```

### 使用场景

- 图片懒加载：当图片滚动到可见位置的时候才加载
- 无限滚动加载：滑动到底部时开始加载新的内容
- 检测广告的曝光率：广告是否被用户看到
- 用户看到某个区域时执行任务或播放动画

### IntersectionObserver API

> IntersectionObserver API，可以自动"观察"元素是否可见。由于可见（visible）的本质是，目标元素与视口产生一个交叉区，所以这个 API 叫做"交叉观察器"。

- [Intersection Observer API](https://developer.mozilla.org/zh-CN/docs/Web/API/Intersection_Observer_API)
- 可参考学习：[IntersectionObserver API 使用教程](https://www.ruanyifeng.com/blog/2016/11/intersectionobserver_api.html)

### 实现思路

1. 监听目标元素，支持传入原生 `IntersectionObserver` API 选项
2. 对 `IntersectionObserver` 构造函数的回调函数设置可见状态与可见比例值
3. 借助 [intersection-observer](https://www.npmjs.com/package/intersection-observer) 库实现 polyfill

### 核心实现

```ts
export interface Options {
  // 根(root)元素的外边距
  rootMargin?: string
  // 可以控制在可见区域达到该比例时触发 ratio 更新。默认值是 0 (意味着只要有一个 target 像素出现在 root 元素中，回调函数将会被执行)。该值为 1.0 含义是当 target 完全出现在 root 元素中时候 回调才会被执行。
  threshold?: number | number[]
  // 指定根(root)元素，用于检查目标的可见性
  root?: BasicTarget<Element>
}
```

```ts
function useInViewport(target: BasicTarget, options?: Options) {
  const [state, setState] = useState<boolean>() // 是否可见
  const [ratio, setRatio] = useState<number>() // 当前可见比例

  useEffectWithTarget(
    () => {
      const el = getTargetElement(target)
      if (!el) {
        return
      }
      // 可以自动观察元素是否可见，返回一个观察器实例
      const observer = new IntersectionObserver(
        (entries) => {
          // callback函数的参数（entries）是一个数组，每个成员都是一个IntersectionObserverEntry对象。如果同时有两个被观察的对象的可见性发生变化，entries数组就会有两个成员。
          for (const entry of entries) {
            setRatio(entry.intersectionRatio) // 设置当前目标元素的可见比例
            setState(entry.isIntersecting) // isIntersecting：如果目标元素与交集观察者的根相交，则该值为true
          }
        },
        {
          ...options,
          root: getTargetElement(options?.root),
        }
      )

      observer.observe(el) // 开始监听一个目标元素

      return () => {
        observer.disconnect() // 停止监听目标
      }
    },
    [options?.rootMargin, options?.threshold],
    target
  )

  return [state, ratio] as const
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useInViewport/index.ts)

## useKeyPress

监听键盘按键，支持组合键，支持按键别名。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-key-press)

### KeyEvent 基础

**JS 的键盘事件**

- [keydown](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/keydown_event)：触发于键盘按键按下的时候。
- [keyup](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/keyup_event)：在按键被松开时触发。
- ~~（已过时）[keypress](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/keypress_event)：按下有值的键时触发，即按下 Ctrl、Alt、Shift、Meta 这样无值的键，这个事件不会触发。对于有值的键，按下时先触发 keydown 事件，再触发 keypress 事件~~

**关于 keyCode**
`（已过时）event.keyCode（返回按下键的数字代码）`，虽然目前大部分代码依然使用并保持兼容。但如果我们自己实现的话应该尽可能使用 `event.key（按下的键的实际值）`属性。具体可见[KeyboardEvent](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent)

**如何监听按键组合**

修饰键有四个

```ts
const modifierKey = {
  ctrl: (event: KeyboardEvent) => event.ctrlKey,
  shift: (event: KeyboardEvent) => event.shiftKey,
  alt: (event: KeyboardEvent) => event.altKey,
  meta: (event: KeyboardEvent) => {
    if (event.type === 'keyup') {
      // 这里使用数组判断是因为 meta 键分左边和右边的键(MetaLeft 91、MetaRight 93)
      return aliasKeyCodeMap['meta'].includes(event.keyCode)
    }
    return event.metaKey
  },
}
```

- 当按下的组合键包含 `Ctrl` 键时，`event.ctrlKey` 属性为 true
- 当按下的组合键包含 `Shift` 键时，`event.shiftKey` 属性为 true
- 当按下的组合键包含 `Alt` 键时，`event.altKey` 属性为 true
- 当按下的组合键包含 `meta` 键时，`event.meta` 属性为 true（Mac 是 command 键，Windows 电脑是 win 键）

如按下 `Alt+K` 组合键，会触发两次 `keydown`事件，其中 `Alt` 键和 `K` 键打印的 `altKey` 都为 true，可以这么判断：

```js
if (event.altKey && keyCode === 75) {
  console.log('按下了 Alt + K 键')
}
```

### 在线测试

这里推荐个在线网站 [Keyboard Events Playground](https://keyevents.netlify.app/)测试键盘事件，只需要输入任意键即可查看有关它打印的信息，还可以通过复选框来过滤事件，辅助我们开发验证。

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usekeypress-demo1/)

在看源码之前，需要了解下该 Hook 支持的用法：

```ts
// 支持键盘事件中的 keyCode 和别名
useKeyPress('uparrow', () => {
  // TODO
})

// keyCode value for ArrowDown
useKeyPress(40, () => {
  // TODO
})

// 监听组合按键
useKeyPress('ctrl.alt.c', () => {
  // TODO
})

// 开启精确匹配。比如按 [shift + c] ，不会触发 [c]
useKeyPress(
  ['c'],
  () => {
    // TODO
  },
  {
    exactMatch: true,
  }
)

// 监听多个按键。如下 a s d f, Backspace, 8
useKeyPress([65, 83, 68, 70, 8, '8'], (event) => {
  setKey(event.key)
})

// 自定义监听方式。支持接收一个返回 boolean 的回调函数，自己处理逻辑。
const filterKey = ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9']
useKeyPress(
  (event) => !filterKey.includes(event.key),
  (event) => {
    // TODO
  },
  {
    events: ['keydown', 'keyup'],
  }
)

// 自定义 DOM。默认监听挂载在 window 上的事件，也可以传入 DOM 指定监听区域，如常见的监听输入框事件
useKeyPress(
  'enter',
  (event: any) => {
    // TODO
  },
  {
    target: inputRef,
  }
)
```

useKeyPress 的参数：

```ts
type keyType = number | string;
// 支持 keyCode、别名、组合键、数组，自定义函数
type KeyFilter = keyType | keyType[] | ((event: KeyboardEvent) => boolean);
// 回调函数
type EventHandler = (event: KeyboardEvent) => void;

type KeyEvent = 'keydown' | 'keyup';
type Options = {
  events?: KeyEvent[]; // 触发事件
  target?: Target; // DOM 节点或者 ref
  exactMatch?: boolean; // 精确匹配。如果开启，则只有在按键完全匹配的情况下触发事件。比如按键 [shif + c] 不会触发 [c]
  useCapture?: boolean; // 是否阻止事件冒泡
};

// useKeyPress 参数
useKeyPress(
  keyFilter: KeyFilter,
  eventHandler: EventHandler,
  options?: Options
);
```

### 实现思路

1. 监听 `keydown` 或 `keyup` 事件，处理事件回调函数。
2. 在事件回调函数中传入 keyFilter 配置进行判断，兼容自定义函数、keyCode、别名、组合键、数组，支持精确匹配
3. 如果满足该回调最终判断结果，则触发 eventHandler 回调

### 核心实现

> - genKeyFormatter：键盘输入预处理方法
> - genFilterKey：判断按键是否激活

沿着上述三点，我们来看这部分精简代码：

```ts
function useKeyPress(keyFilter: KeyFilter, eventHandler: EventHandler, option?: Options) {
  const { events = defaultEvents, target, exactMatch = false, useCapture = false } = option || {}
  const eventHandlerRef = useLatest(eventHandler)
  const keyFilterRef = useLatest(keyFilter)

  // 监听元素（深比较）
  useDeepCompareEffectWithTarget(
    () => {
      const el = getTargetElement(target, window)
      if (!el) {
        return
      }

      // 事件回调函数
      const callbackHandler = (event: KeyboardEvent) => {
        // 键盘输入预处理方法
        const genGuard: KeyPredicate = genKeyFormatter(keyFilterRef.current, exactMatch)
        // 判断是否匹配 keyFilter 配置结果，返回 true 则触发传入的回调函数
        if (genGuard(event)) {
          return eventHandlerRef.current?.(event)
        }
      }

      // 监听事件（默认事件：keydown）
      for (const eventName of events) {
        el?.addEventListener?.(eventName, callbackHandler, useCapture)
      }
      return () => {
        // 取消监听
        for (const eventName of events) {
          el?.removeEventListener?.(eventName, callbackHandler, useCapture)
        }
      }
    },
    [events],
    target
  )
}
```

上面的代码看起来比较好理解，需要推敲的就是 `genKeyFormatter` 函数。

```ts
/**
 * 键盘输入预处理方法
 * @param [keyFilter: any] 当前键
 * @returns () => Boolean
 */
function genKeyFormatter(keyFilter: KeyFilter, exactMatch: boolean): KeyPredicate {
  // 支持自定义函数
  if (isFunction(keyFilter)) {
    return keyFilter
  }
  // 支持 keyCode、别名、组合键
  if (isString(keyFilter) || isNumber(keyFilter)) {
    return (event: KeyboardEvent) => genFilterKey(event, keyFilter, exactMatch)
  }
  // 支持数组
  if (Array.isArray(keyFilter)) {
    return (event: KeyboardEvent) => keyFilter.some((item) => genFilterKey(event, item, exactMatch))
  }
  // 等同 return keyFilter ? () => true : () => false;
  return () => Boolean(keyFilter)
}
```

看完发现上面的重点实现还是在 `genFilterKey` 函数：

- [aliasKeyCodeMap](https://github.com/alibaba/hooks/blob/master/packages/hooks/src/useKeyPress/index.ts#L24)

这段逻辑需要各位代入实际数值帮助理解，如输入组合键 `shift.c`

```ts
/**
 * 判断按键是否激活
 * @param [event: KeyboardEvent]键盘事件
 * @param [keyFilter: any] 当前键
 * @returns Boolean
 */
function genFilterKey(event: KeyboardEvent, keyFilter: keyType, exactMatch: boolean) {
  // 浏览器自动补全 input 的时候，会触发 keyDown、keyUp 事件，但此时 event.key 等为空
  if (!event.key) {
    return false
  }

  // 数字类型直接匹配事件的 keyCode
  if (isNumber(keyFilter)) {
    return event.keyCode === keyFilter
  }

  // 字符串依次判断是否有组合键
  const genArr = keyFilter.split('.') // 如 keyFilter 可以传 ctrl.alt.c，['shift.c']
  let genLen = 0

  for (const key of genArr) {
    // 组合键
    const genModifier = modifierKey[key] // ctrl/shift/alt/meta
    // keyCode 别名
    const aliasKeyCode: number | number[] = aliasKeyCodeMap[key.toLowerCase()]

    if ((genModifier && genModifier(event)) || (aliasKeyCode && aliasKeyCode === event.keyCode)) {
      genLen++
    }
  }

  /**
   * 需要判断触发的键位和监听的键位完全一致，判断方法就是触发的键位里有且等于监听的键位
   * genLen === genArr.length 能判断出来触发的键位里有监听的键位
   * countKeyByEvent(event) === genArr.length 判断出来触发的键位数量里有且等于监听的键位数量
   * 主要用来防止按组合键其子集也会触发的情况，例如监听 ctrl+a 会触发监听 ctrl 和 a 两个键的事件。
   */
  if (exactMatch) {
    return genLen === genArr.length && countKeyByEvent(event) === genArr.length
  }
  return genLen === genArr.length
}

// 根据 event 计算激活键数量
function countKeyByEvent(event: KeyboardEvent) {
  // 计算激活的修饰键数量
  const countOfModifier = Object.keys(modifierKey).reduce((total, key) => {
    // (event: KeyboardEvent) => Boolean
    if (modifierKey[key](event)) {
      return total + 1
    }

    return total
  }, 0)

  // 16 17 18 91 92 是修饰键的 keyCode，如果 keyCode 是修饰键，那么激活数量就是修饰键的数量，如果不是，那么就需要 +1
  return [16, 17, 18, 91, 92].includes(event.keyCode) ? countOfModifier : countOfModifier + 1
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useKeyPress/index.ts)

## useLongPress

监听目标元素的长按事件。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-long-press)

### 基本用法

支持参数：

```ts
export interface Options {
  delay?: number
  moveThreshold?: { x?: number; y?: number }
  onClick?: (event: EventType) => void
  onLongPressEnd?: (event: EventType) => void
}
```

[官方在线 Demo](https://ahooks.js.org/~demos/uselongpress-demo1/)

```ts
import React, { useState, useRef } from 'react'
import { useLongPress } from 'ahooks'

export default () => {
  const [counter, setCounter] = useState(0)
  const ref = useRef<HTMLButtonElement>(null)

  useLongPress(() => setCounter((s) => s + 1), ref)

  return (
    <div>
      <button ref={ref} type="button">
        Press me
      </button>
      <p>counter: {counter}</p>
    </div>
  )
}
```

### touch 事件

- [touchstart](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/touchstart_event)：在一个或多个触点与触控设备表面接触时被触发
- [touchmove](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/touchmove_event)：在触点于触控平面上移动时触发
- [touchend](https://developer.mozilla.org/en-US/docs/Web/API/Element/touchend_event)：当触点离开触控平面时触发 touchend 事件

### 实现思路

1. 判断当前环境是否支持 touch 事件：支持则监听 `touchstart`、`touchend` 事件，不支持则监听 `mousedown`、`mouseup`、`mouseleave` 事件
2. 根据触发监听事件和定时器共同来判断是否达到长按事件，达到则触发外部回调
3. 如果外部有传 `moveThreshold（按下后移动阈值）`参数 ，则需要监听 `mousemove` 或`touchmove` 事件进行处理

### 核心实现

根据[实现思路]第一条，很容易看懂实现大致框架代码：

```ts
type EventType = MouseEvent | TouchEvent

// 是否支持 touch 事件
const touchSupported =
  isBrowser &&
  // @ts-ignore
  ('ontouchstart' in window || (window.DocumentTouch && document instanceof DocumentTouch))

function useLongPress(
  onLongPress: (event: EventType) => void,
  target: BasicTarget,
  { delay = 300, moveThreshold, onClick, onLongPressEnd }: Options = {}
) {
  const onLongPressRef = useLatest(onLongPress)
  const onClickRef = useLatest(onClick)
  const onLongPressEndRef = useLatest(onLongPressEnd)

  const timerRef = useRef<ReturnType<typeof setTimeout>>()
  const isTriggeredRef = useRef(false)
  // 是否有设置移动阈值
  const hasMoveThreshold = !!((moveThreshold?.x && moveThreshold.x > 0) || (moveThreshold?.y && moveThreshold.y > 0))

  useEffectWithTarget(
    () => {
      const targetElement = getTargetElement(target)
      if (!targetElement?.addEventListener) {
        return
      }

      const overThreshold = (event: EventType) => {}

      function getClientPosition(event: EventType) {}

      const onStart = (event: EventType) => {}

      const onMove = (event: TouchEvent) => {}

      const onEnd = (event: EventType, shouldTriggerClick: boolean = false) => {}

      const onEndWithClick = (event: EventType) => onEnd(event, true)

      if (!touchSupported) {
        // 不支持 touch 事件
        targetElement.addEventListener('mousedown', onStart)
        targetElement.addEventListener('mouseup', onEndWithClick)
        targetElement.addEventListener('mouseleave', onEnd)
        if (hasMoveThreshold) targetElement.addEventListener('mousemove', onMove)
      } else {
        // 支持 touch 事件
        targetElement.addEventListener('touchstart', onStart)
        targetElement.addEventListener('touchend', onEndWithClick)
        if (hasMoveThreshold) targetElement.addEventListener('touchmove', onMove)
      }

      // 卸载函数解绑监听事件
      return () => {
        // 清除定时器，重置状态
        if (timerRef.current) {
          clearTimeout(timerRef.current)
          isTriggeredRef.current = false
        }
        if (!touchSupported) {
          targetElement.removeEventListener('mousedown', onStart)
          targetElement.removeEventListener('mouseup', onEndWithClick)
          targetElement.removeEventListener('mouseleave', onEnd)
          if (hasMoveThreshold) targetElement.removeEventListener('mousemove', onMove)
        } else {
          targetElement.removeEventListener('touchstart', onStart)
          targetElement.removeEventListener('touchend', onEndWithClick)
          if (hasMoveThreshold) targetElement.removeEventListener('touchmove', onMove)
        }
      }
    },
    [],
    target
  )
}
```

对于是否支持 touch 事件的判断代码，需要了解一种场景，在搜的时候发现一篇文章可以看下：[touchstart 与 click 不得不说的故事](https://juejin.cn/post/6844903588683071495)

**如何判断长按事件**：

1. 在 onStart 设置一个定时器 setTimeout 用来判断长按时间，在定时器回调将 isTriggeredRef.current 设置为 true，表示触发了长按事件；
2. 在 onEnd 清除定时器并判断 isTriggeredRef.current 的值，true 代表触发了长按事件；false 代表没触发 setTimeout 里面的回调，则不触发长按事件。

```ts
const onStart = (event: EventType) => {
  timerRef.current = setTimeout(() => {
    // 达到设置的长按时间
    onLongPressRef.current(event)
    isTriggeredRef.current = true
  }, delay)
}

const onEnd = (event: EventType, shouldTriggerClick: boolean = false) => {
  // 清除 onStart 设置的定时器
  if (timerRef.current) {
    clearTimeout(timerRef.current)
  }
  // 判断是否达到长按时间
  if (isTriggeredRef.current) {
    onLongPressEndRef.current?.(event)
  }
  // 是否触发点击事件
  if (shouldTriggerClick && !isTriggeredRef.current && onClickRef.current) {
    onClickRef.current(event)
  }
  // 重置
  isTriggeredRef.current = false
}
```

实现了[实现思路]的前两点，接下来需要实现第三点，传 `moveThreshold` 的情况

```ts
const hasMoveThreshold = !!((moveThreshold?.x && moveThreshold.x > 0) || (moveThreshold?.y && moveThreshold.y > 0))
```

> clientX、clientY：点击位置距离当前 body 可视区域的 x，y 坐标

```ts
const onStart = (event: EventType) => {
  if (hasMoveThreshold) {
    const { clientX, clientY } = getClientPosition(event)
    // 记录首次点击/触屏时的位置
    pervPositionRef.current.x = clientX
    pervPositionRef.current.y = clientY
  }
  // ...
}

// 传 moveThreshold 需绑定 onMove 事件
const onMove = (event: TouchEvent) => {
  if (timerRef.current && overThreshold(event)) {
    // 超过移动阈值不触发长按事件，并清除定时器
    clearInterval(timerRef.current)
    timerRef.current = undefined
  }
}

// 判断是否超过移动阈值
const overThreshold = (event: EventType) => {
  const { clientX, clientY } = getClientPosition(event)
  const offsetX = Math.abs(clientX - pervPositionRef.current.x)
  const offsetY = Math.abs(clientY - pervPositionRef.current.y)

  return !!((moveThreshold?.x && offsetX > moveThreshold.x) || (moveThreshold?.y && offsetY > moveThreshold.y))
}

function getClientPosition(event: EventType) {
  if (event instanceof TouchEvent) {
    return {
      clientX: event.touches[0].clientX,
      clientY: event.touches[0].clientY,
    }
  }

  if (event instanceof MouseEvent) {
    return {
      clientX: event.clientX,
      clientY: event.clientY,
    }
  }

  console.warn('Unsupported event type')

  return { clientX: 0, clientY: 0 }
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useLongPress/index.ts)

# 【解读 ahooks 源码系列】Effect 篇（二）

本文是 ahooks 源码（[v3.7.4](https://github.com/alibaba/hooks/tree/v3.7.4)）系列的第十一篇——Effect 篇（二）

往期文章：

- [【解读 ahooks 源码系列】（开篇）如何获取和监听 DOM 元素](https://juejin.cn/post/7201889983170592824)：useEffectWithTarget
- [【解读 ahooks 源码系列】DOM 篇（一）](https://juejin.cn/post/7202254039215800375)：useEventListener、useClickAway、useDocumentVisibility、useDrop、useDrag
- [【解读 ahooks 源码系列】DOM 篇（二）](https://juejin.cn/post/7202633255043465271)：useEventTarget、useExternal、useTitle、useFavicon、useFullscreen、useHover
- [【解读 ahooks 源码系列】DOM 篇（三）](https://juejin.cn/post/7202996870251380791)：useMutationObserver、useInViewport、useKeyPress、useLongPress
- [【解读 ahooks 源码系列】DOM 篇（四）](https://juejin.cn/post/7203397626527891515)：useMouse、useResponsive、useScroll、useSize、useFocusWithin
- [【解读 ahooks 源码系列】Dev 篇——useTrackedEffect 和 useWhyDidYouUpdate](https://juejin.cn/post/7204683121374232635)
- [【解读 ahooks 源码系列】Advanced 篇](https://juejin.cn/post/7207810396420669477)：useControllableValue、useCreation、useIsomorphicLayoutEffect、useEventEmitter、useLatest、useMemoizedFn、useReactive
- [【解读 ahooks 源码系列】State 篇（一）](https://juejin.cn/post/7210786286570324025)：useSetState、useToggle、useBoolean、useCookieState、useLocalStorageState、useSessionStorageState、useDebounce、useThrottle
- [【解读 ahooks 源码系列】State 篇（二）](https://juejin.cn/post/7212263304395620408)：useMap、useSet、usePrevious、useRafState、useSafeState、useGetState、useResetState
- [【解读 ahooks 源码系列】Effect 篇（一）](https://juejin.cn/post/7214858677173715001)：useUpdateEffect、useUpdateLayoutEffect、useAsyncEffect、useDebounceFn、useDebounceEffect、useThrottleFn、useThrottleEffect

本文主要解读 `useDeepCompareEffect`、`useDeepCompareLayoutEffect`、`useInterval`、`useTimeout`、`useRafInterval`、`useRafTimeout`、`useLockFn`、`useUpdate` 的源码实现

## useDeepCompareEffect & useDeepCompareLayoutEffect

用法与 `useEffect/useLayoutEffect` 一致，但 deps 通过 [lodash isEqual](https://lodash.com/docs/4.17.15#isEqual) 进行深比较。

`useDeepCompareEffect` 与 `useDeepCompareLayoutEffect` 的区别只是参数不同，都调用了 `createDeepCompareEffect` 方法，下面就只介绍 `useDeepCompareEffect` 做例子

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-deep-compare-effect/)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usedeepcompareeffect-demo1/)

```ts
import { useDeepCompareEffect } from 'ahooks'
import React, { useEffect, useState, useRef } from 'react'

export default () => {
  const [count, setCount] = useState(0)
  const effectCountRef = useRef(0)
  const deepCompareCountRef = useRef(0)

  useEffect(() => {
    effectCountRef.current += 1
  }, [{}])

  useDeepCompareEffect(() => {
    deepCompareCountRef.current += 1
    return () => {
      // do something
    }
  }, [{}])

  return (
    <div>
      <p>effectCount: {effectCountRef.current}</p>
      <p>deepCompareCount: {deepCompareCountRef.current}</p>
      <p>
        <button type="button" onClick={() => setCount((c) => c + 1)}>
          reRender
        </button>
      </p>
    </div>
  )
}
```

### 核心实现

通过 useRef 保存上一次的依赖值，与当前的依赖对比（使用 lodash 的 isEqual 深比较方法），不同则将`signalRef.current` 的值加 1，并作为 useEffect 的依赖项，更新了 effect 函数就会重新执行。

- [lodash.isEqual](https://lodash.com/docs/4.17.15#isEqual)：通过深比较来确定两者的值是否相等

使用了 createDeepCompareEffect 方法

```ts
export default createDeepCompareEffect(useEffect)
```

createDeepCompareEffect 函数实现：

```ts
type EffectHookType = typeof useEffect | typeof useLayoutEffect
type CreateUpdateEffect = (hook: EffectHookType) => EffectHookType

const depsEqual = (aDeps: DependencyList = [], bDeps: DependencyList = []) => {
  return isEqual(aDeps, bDeps)
}

export const createDeepCompareEffect: CreateUpdateEffect = (hook) => (effect, deps) => {
  // ref 用于保存上一次 deps 依赖的值
  const ref = useRef<DependencyList>()
  // 通过 signalRef 的值改变来触发 hook(useEffect/useLayoutEffect) 中的回调函数
  const signalRef = useRef<number>(0)

  // 判断当前依赖于上一次依赖值相不相等（深比较）
  if (deps === undefined || !depsEqual(deps, ref.current)) {
    ref.current = deps // 更新保存当前依赖作为上一次依赖值
    signalRef.current += 1 // 值改变，触发 effect 函数执行
  }

  hook(effect, [signalRef.current])
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useDeepCompareEffect/index.tsx)

## useInterval & useTimeout

- useInterval：一个可以处理 setInterval 的 Hook。
- useTimeout：一个可以处理 setTimeout 计时器函数的 Hook。

官方文档：

- [useInterval 官方文档](https://ahooks.js.org/zh-CN/hooks/use-interval)
- [useTimeout 官方文档](https://ahooks.js.org/zh-CN/hooks/use-interval)

`useInterval` 与 `useTimeout` 的实现思路基本一致（除了 `useTimeout` 没有 `immediate` 参数选）。下面就只举例 `useInterval` 了

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usedeepcompareeffect-demo1/)

每 1000ms，执行一次

```ts
import React, { useState } from 'react'
import { useInterval } from 'ahooks'

export default () => {
  const [count, setCount] = useState(0)

  useInterval(() => {
    setCount(count + 1)
  }, 1000)

  return <div>count: {count}</div>
}
```

### 核心实现

跟我们自己写 `setInterval` 的区别：

- 无需手动清除定时器的逻辑，简化代码
- 支持 `immediate` 参数，通过 `immediate` 可以在首次渲染时立即执行
- 支持 `delay` 参数变更时重新启动定时器

```ts
function useInterval(
  fn: () => void, // 要定时调用的函数
  delay: number | undefined, // 间隔时间，当设置值为 undefined 时会停止计时器
  options: {
    immediate?: boolean // 是否在首次渲染时立即执行
  } = {}
) {
  // 是否首次立即执行
  const { immediate } = options
  // 要执行的函数，使用 useLatest 拿到最新的引用
  const fnRef = useLatest(fn)
  const timerRef = useRef<NodeJS.Timer | null>(null)

  // 如果没有传入间隔 则不执行定时器
  useEffect(() => {
    if (!isNumber(delay) || delay < 0) {
      return
    }
    // 立即执行
    if (immediate) {
      fnRef.current()
    }
    timerRef.current = setInterval(() => {
      fnRef.current()
    }, delay)
    return () => {
      // 组件卸载时内部做清除操作
      if (timerRef.current) {
        clearInterval(timerRef.current)
      }
    }
  }, [delay])

  // 清除定时器
  const clear = useCallback(() => {
    if (timerRef.current) {
      clearInterval(timerRef.current)
    }
  }, [])

  return clear
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useInterval/index.ts)

## useRafInterval & useRafTimeout

用 requestAnimationFrame 模拟实现 setInterval，API 和 useInterval 保持一致，好处是可以在页面不渲染的时候停止执行定时器，比如页面隐藏或最小化等。

请注意，如下两种情况下很可能是不适用的，优先考虑 useInterval ：

- 时间间隔小于 16ms
- 希望页面不渲染的情况下依然执行定时器

> Node 环境下 requestAnimationFrame 会自动降级到 setInterval

官方文档：

- [useRafInterval 官方文档](https://ahooks.js.org/zh-CN/hooks/use-raf-interval)
- [useRafTimeout 官方文档](https://ahooks.js.org/zh-CN/hooks/use-raf-timeout)

`useRafInterval` 与 `useRafTimeout` 的实现思路基本一致。下面就只举例 `useRafInterval`

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/userafinterval-demo1/)

每 1000ms，执行一次

```ts
import React, { useState } from 'react'
import { useRafInterval } from 'ahooks'

export default () => {
  const [count, setCount] = useState(0)

  useRafInterval(() => {
    setCount(count + 1)
  }, 1000)

  return <div>count: {count}</div>
}
```

### 使用场景

假如希望在页面不可见的时候，不执行定时器，可以选择 useRafInterval 和 useRafTimeout，其内部是使用 requestAnimationFrame 进行实现。这是因为当 requestAnimationFrame() 运行在后台标签页或者隐藏的`<iframe>` 里时，requestAnimationFrame() 会被暂停调用。

### 核心实现

主要是借助 requestAnimationFrame API：

> window.requestAnimationFrame() 告诉浏览器——你希望执行一个动画，并且要求浏览器在下次重绘之前调用指定的回调函数更新动画。该方法需要传入一个回调函数作为参数，该回调函数会在浏览器下一次重绘之前执行

主函数实现：

可以看出主函数实现与 `useInterval` 几乎一致，区别是封装了 `setRafInterval` 和 `clearRafInterval`

```ts
function useRafInterval(
  fn: () => void, // 要定时调用的函数
  delay: number | undefined, // 间隔时间，当取值 undefined 时会停止计时器
  options?: {
    immediate?: boolean // 是否在首次渲染时立即执行
  }
) {
  const immediate = options?.immediate

  const fnRef = useLatest(fn)
  const timerRef = useRef<Handle>()

  useEffect(() => {
    if (!isNumber(delay) || delay < 0) return
    if (immediate) {
      fnRef.current()
    }
    timerRef.current = setRafInterval(() => {
      fnRef.current()
    }, delay)
    return () => {
      if (timerRef.current) {
        clearRafInterval(timerRef.current)
      }
    }
  }, [delay])

  const clear = useCallback(() => {
    if (timerRef.current) {
      clearRafInterval(timerRef.current)
    }
  }, [])

  return clear
}
```

`setRafInterval` 的实现：

1. 判断是否支持 requestAnimationFrame，不支持则降级使用 setInterval
2. 定义 loop 函数，通过 requestAnimationFrame 去执行。
3. loop 函数实现：每次执行都需要记录当前时间，并用当前时间（current） - 开始时间（start），相减判断是否间隔时间，大于则执行回调函数，并更新最新开始时间（start）

```ts
const setRafInterval = function (callback: () => void, delay: number = 0): Handle {
  // 如果不支持 requestAnimationFrame，则降级使用 setInterval
  if (typeof requestAnimationFrame === typeof undefined) {
    return {
      id: setInterval(callback, delay),
    }
  }
  // 开始时间
  let start = new Date().getTime()
  const handle: Handle = {
    id: 0,
  }
  const loop = () => {
    const current = new Date().getTime()
    // 现在的时间减去开始时间是否大于间隔时间，是则更新开始时间
    if (current - start >= delay) {
      // 达到则执行我们的 callback 函数
      callback()
      start = new Date().getTime()
    }
    handle.id = requestAnimationFrame(loop)
  }
  handle.id = requestAnimationFrame(loop)
  return handle
}
```

`clearRafInterval` 函数的实现：

```ts
function cancelAnimationFrameIsNotDefined(t: any): t is NodeJS.Timer {
  return typeof cancelAnimationFrame === typeof undefined
}

// 清除定时器
const clearRafInterval = function (handle: Handle) {
  if (cancelAnimationFrameIsNotDefined(handle.id)) {
    return clearInterval(handle.id)
  }
  // cancelAnimationFrame：取消一个先前通过调用 window.requestAnimationFrame()方法添加到计划中的动画帧请求
  // 支持 requestAnimationFrame 则用 cancelAnimationFrame 清除定时器
  cancelAnimationFrame(handle.id)
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useRafInterval/index.ts)

## useLockFn

用于给一个异步函数增加竞态锁，防止并发执行。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-lock-fn)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/uselockfn-demo1/)

**防止重复提交**

在 submit 函数执行完成前，其余的点击动作都会被忽略。

```ts
import { useLockFn } from 'ahooks'
import { message } from 'antd'
import React, { useState } from 'react'

function mockApiRequest() {
  return new Promise<void>((resolve) => {
    setTimeout(() => {
      resolve()
    }, 2000)
  })
}

export default () => {
  const [count, setCount] = useState(0)

  const submit = useLockFn(async () => {
    message.info('Start to submit')
    await mockApiRequest()
    setCount((val) => val + 1)
    message.success('Submit finished')
  })

  return (
    <>
      <p>Submit count: {count}</p>
      <button onClick={submit}>Submit</button>
    </>
  )
}
```

### 使用场景

业务中点击某个按钮进行请求，当请求未完成时 ，再次点击不进行处理，需要等请求结果返回后才能发起下一次请求，防止并发执行

### 核心实现

实现思路：

1. 使用 useRef 记录锁的状态，请求时设置为 true，请求完成或请求失败时设置为 false。
2. 请求前判断锁的状态是否为 true，为 true 则不处理

```ts
function useLockFn<P extends any[] = any[], V extends any = any>(fn: (...args: P) => Promise<V>) {
  // 记录锁的状态
  const lockRef = useRef(false)

  return useCallback(
    async (...args: P) => {
      // 如果处于锁状态，则不执行
      if (lockRef.current) return
      // 请求中，上锁
      lockRef.current = true
      try {
        // 执行请求函数
        const ret = await fn(...args)
        // 请求完成，解锁
        lockRef.current = false
        return ret
      } catch (e) {
        // 请求失败，也需要解锁
        lockRef.current = false
        throw e
      }
    },
    [fn]
  )
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useLockFn/index.ts)

## useUpdate

useUpdate 会返回一个函数，调用该函数会强制组件重新渲染。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-update)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/zh-CN/hooks/use-update)

强制组件重新渲染。

```ts
import React from 'react'
import { useUpdate } from 'ahooks'

export default () => {
  const update = useUpdate()

  return (
    <>
      <div>Time: {Date.now()}</div>
      <button type="button" onClick={update} style={{ marginTop: 8 }}>
        update
      </button>
    </>
  )
}
```

### 核心实现

这个实现比较简单，暴露一个函数，每次该函数执行的时候都是 `setState({})`，而对于 state 状态值本身并不重要。该 Hook 即是简化我们写法，当有特殊场景强制更新的时候。

```ts
const useUpdate = () => {
  const [, setState] = useState({})
  // 设置一个新的状态（新的空对象）以强制更新，暴露该函数
  return useCallback(() => setState({}), [])
}
```

这个也可以有其他方式的实现，比如 [react-use](https://github.com/streamich/react-use/blob/master/src/useUpdate.ts)：

```ts
import { useReducer } from 'react'

// 将 num 递增 1，然后对 1000000 取模；当 num 达到 1000000 时，它会重新回到 0。这是为了防止 num 变得过大
const updateReducer = (num: number): number => (num + 1) % 1_000_000

export default function useUpdate(): () => void {
  const [, update] = useReducer(updateReducer, 0)

  return update
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useUpdate/index.ts)

# 【解读 ahooks 源码系列】Effect 篇（一）

本文是 ahooks 源码（[v3.7.4](https://github.com/alibaba/hooks/tree/v3.7.4)）系列的第十篇——Effect 篇（一）

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

本文主要解读 `useUpdateEffect`、`useUpdateLayoutEffect`、`useAsyncEffect`、`useDebounceFn`、`useDebounceEffect`、`useThrottleFn`、`useThrottleEffect` 的源码实现

## useUpdateEffect

`useUpdateEffect` 用法等同于 `useEffect`，但是会忽略首次执行，只在依赖更新时执行。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-update-effect)

API 与 React.useEffect 完全一致。

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/useupdateeffect-demo1/)

```ts
import React, { useEffect, useState } from 'react'
import { useUpdateEffect } from 'ahooks'

export default () => {
  const [count, setCount] = useState(0)
  const [effectCount, setEffectCount] = useState(0)
  const [updateEffectCount, setUpdateEffectCount] = useState(0)

  useEffect(() => {
    setEffectCount((c) => c + 1)
  }, [count])

  useUpdateEffect(() => {
    setUpdateEffectCount((c) => c + 1)
    return () => {
      // do something
    }
  }, [count]) // you can include deps array if necessary

  return (
    <div>
      <p>effectCount: {effectCount}</p>
      <p>updateEffectCount: {updateEffectCount}</p>
      <p>
        <button type="button" onClick={() => setCount((c) => c + 1)}>
          reRender
        </button>
      </p>
    </div>
  )
}
```

### 实现思路

- 初始化一个 isMounted 标识，默认为 false；首次 useEffect 执行完后置为 true
- 后续执行 useEffect 的时候判断 isMounted 标识是否为 true，true 则执行外部传入的 effect 函数；卸载的时候将 isMounted 标识重置为 false

### 核心实现

里面其实是实现了 createUpdateEffect 这个函数：

```ts
export default createUpdateEffect(useEffect)
```

```ts
type EffectHookType = typeof useEffect | typeof useLayoutEffect

export const createUpdateEffect: (hook: EffectHookType) => EffectHookType = (hook) => (effect, deps) => {
  const isMounted = useRef(false)

  // for react-refresh
  hook(() => {
    // 卸载时重置 isMounted 为 false
    return () => {
      isMounted.current = false
    }
  }, [])

  hook(() => {
    if (!isMounted.current) {
      // 首次执行完设置为 true
      isMounted.current = true
    } else {
      // 第一次后则执行函数
      return effect()
    }
  }, deps)
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useUpdateEffect/index.ts)

## useUpdateLayoutEffect

`useUpdateLayoutEffect` 用法等同于 `useLayoutEffect`，但是会忽略首次执行，只在依赖更新时执行。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-update-layout-effect)

API 与 React.useLayoutEffect 完全一致。

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/useupdatelayouteffect-demo1/)

使用上与 useLayoutEffect 完全相同，只是它忽略了首次执行，且只在依赖项更新时执行。

```ts
import React, { useLayoutEffect, useState } from 'react'
import { useUpdateLayoutEffect } from 'ahooks'

export default () => {
  const [count, setCount] = useState(0)
  const [layoutEffectCount, setLayoutEffectCount] = useState(0)
  const [updateLayoutEffectCount, setUpdateLayoutEffectCount] = useState(0)

  useLayoutEffect(() => {
    setLayoutEffectCount((c) => c + 1)
  }, [count])

  useUpdateLayoutEffect(() => {
    setUpdateLayoutEffectCount((c) => c + 1)
    return () => {
      // do something
    }
  }, [count]) // you can include deps array if necessary

  return (
    <div>
      <p>layoutEffectCount: {layoutEffectCount}</p>
      <p>updateLayoutEffectCount: {updateLayoutEffectCount}</p>
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

和 useUpdateEffect 一样，都是调用了 createUpdateEffect 方法，区别只是传入的 hook 是 useLayoutEffect。其余源码同上，就不再列举了

```ts
export default createUpdateEffect(useLayoutEffect)
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useLayoutEffect/index.ts)

## useDebounceFn

用来处理防抖函数的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-debounce-fn)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usedebouncefn-demo1/)

频繁调用 run，但只会在所有点击完成 500ms 后执行一次相关函数

```ts
import { useDebounceFn } from 'ahooks'
import React, { useState } from 'react'

export default () => {
  const [value, setValue] = useState(0)
  const { run } = useDebounceFn(
    () => {
      setValue(value + 1)
    },
    {
      wait: 500,
    }
  )

  return (
    <div>
      <p style={{ marginTop: 16 }}> Clicked count: {value} </p>
      <button type="button" onClick={run}>
        Click fast!
      </button>
    </div>
  )
}
```

### 核心实现

支持的选项，都是 [lodash.debounce](https://lodash.com/docs/4.17.15#debounce) 里面的参数：

```ts
interface DebounceOptions {
  wait?: number // 等待时间，单位为毫秒
  leading?: boolean // 是否在延迟开始前调用函数
  trailing?: boolean // 是否在延迟开始后调用函数
  maxWait?: number // 最大等待时间，单位为毫秒
}
```

```ts
function useDebounceFn<T extends noop>(fn: T, options?: DebounceOptions) {
  if (isDev) {
    if (!isFunction(fn)) {
      console.error(`useDebounceFn expected parameter is a function, got ${typeof fn}`)
    }
  }

  // 最新的 fn 防抖函数
  const fnRef = useLatest(fn)

  // 默认是 1000 毫秒
  const wait = options?.wait ?? 1000

  // 防抖函数
  const debounced = useMemo(
    () =>
      debounce(
        (...args: Parameters<T>): ReturnType<T> => {
          return fnRef.current(...args)
        },
        wait,
        options
      ),
    []
  )

  // 卸载时取消防抖函数调用
  useUnmount(() => {
    debounced.cancel()
  })

  return {
    run: debounced, // 触发执行 fn
    cancel: debounced.cancel, // 取消当前防抖
    flush: debounced.flush, // 当前防抖立即调用
  }
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useDebounceFn/index.ts)

## useDebounceEffect

为 useEffect 增加防抖的能力。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-debounce-effect)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usedebounceeffect-demo1/)

```ts
import { useDebounceEffect } from 'ahooks'
import React, { useState } from 'react'

export default () => {
  const [value, setValue] = useState('hello')
  const [records, setRecords] = useState<string[]>([])
  useDebounceEffect(
    () => {
      setRecords((val) => [...val, value])
    },
    [value],
    {
      wait: 1000,
    }
  )
  return (
    <div>
      <input
        value={value}
        onChange={(e) => setValue(e.target.value)}
        placeholder="Typed value"
        style={{ width: 280 }}
      />
      <p style={{ marginTop: 16 }}>
        <ul>
          {records.map((record, index) => (
            <li key={index}>{record}</li>
          ))}
        </ul>
      </p>
    </div>
  )
}
```

### 核心实现

它的实现依赖于 `useDebounceFn` hook。

实现逻辑：本来 deps 更新，effect 函数就立即执行的；现在 deps 更新，执行 防抖函数 setFlag 来更新 flag，而 flag 又被 useUpdateEffect 监听，通过 useUpdateEffect Hook 来执行 effect 函数

```ts
function useDebounceEffect(
  // 执行函数
  effect: EffectCallback,
  // 依赖数组
  deps?: DependencyList,
  // 配置防抖的行为
  options?: DebounceOptions
) {
  const [flag, setFlag] = useState({})

  const { run } = useDebounceFn(() => {
    setFlag({}) // 设置新的空对象，强制触发更新
  }, options)

  useEffect(() => {
    return run()
  }, deps)

  // 只在 flag 依赖更新时执行，但是会忽略首次执行
  useUpdateEffect(effect, [flag])
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useDebounceEffect/index.ts)

## useThrottleFn

用来处理函数节流的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-debounce-effect)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/zh-CN/hooks/use-throttle-fn)

频繁调用 run，但只会每隔 500ms 执行一次相关函数。

```ts
import React, { useState } from 'react'
import { useThrottleFn } from 'ahooks'

export default () => {
  const [value, setValue] = useState(0)
  const { run } = useThrottleFn(
    () => {
      setValue(value + 1)
    },
    { wait: 500 }
  )

  return (
    <div>
      <p style={{ marginTop: 16 }}> Clicked count: {value} </p>
      <button type="button" onClick={run}>
        Click fast!
      </button>
    </div>
  )
}
```

### 核心实现

实现原理是调用封装 lodash 的 throttle 方法。

支持的选项，都是 [lodash.throttle](https://lodash.com/docs/4.17.15#throttle) 里面的参数：

```ts
interface ThrottleOptions {
  wait?: number // 等待时间，单位为毫秒
  leading?: boolean // 是否在延迟开始前调用函数
  trailing?: boolean // 是否在延迟开始后调用函数
}
```

```ts
function useThrottleFn<T extends noop>(fn: T, options?: ThrottleOptions) {
  if (isDev) {
    if (!isFunction(fn)) {
      console.error(`useThrottleFn expected parameter is a function, got ${typeof fn}`)
    }
  }

  // 最新的 fn 节流函数
  const fnRef = useLatest(fn)

  // 默认是 1000 毫秒
  const wait = options?.wait ?? 1000

  // 节流函数
  const throttled = useMemo(
    () =>
      throttle(
        (...args: Parameters<T>): ReturnType<T> => {
          return fnRef.current(...args)
        },
        wait,
        options
      ),
    []
  )

  // 卸载时取消节流函数调用
  useUnmount(() => {
    throttled.cancel()
  })

  return {
    run: throttled, // 触发执行 fn
    cancel: throttled.cancel, // 取消当前节流
    flush: throttled.flush, // 当前节流立即调用
  }
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useThrottleFn/index.ts)

## useThrottleEffect

为 useEffect 增加节流的能力。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-throttle-effect)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usethrottleeffect-demo1/)

```ts
import React, { useState } from 'react'
import { useThrottleEffect } from 'ahooks'

export default () => {
  const [value, setValue] = useState('hello')
  const [records, setRecords] = useState<string[]>([])
  useThrottleEffect(
    () => {
      setRecords((val) => [...val, value])
    },
    [value],
    {
      wait: 1000,
    }
  )
  return (
    <div>
      <input
        value={value}
        onChange={(e) => setValue(e.target.value)}
        placeholder="Typed value"
        style={{ width: 280 }}
      />
      <p style={{ marginTop: 16 }}>
        <ul>
          {records.map((record, index) => (
            <li key={index}>{record}</li>
          ))}
        </ul>
      </p>
    </div>
  )
}
```

### 核心实现

它的实现依赖于 `useThrottleFn` hook，具体实现逻辑同 useDebounceEffect，只是把 `useDebounceFn` 换成 `useThrottleFn`

```ts
function useThrottleEffect(
  // 执行函数
  effect: EffectCallback,
  // 依赖数组
  deps?: DependencyList,
  // 配置节流的行为
  options?: ThrottleOptions
) {
  const [flag, setFlag] = useState({})

  const { run } = useThrottleFn(() => {
    setFlag({}) // 设置新的空对象，强制触发更新
  }, options)

  useEffect(() => {
    return run()
  }, deps)

  // 只在 flag 依赖更新时执行，但是会忽略首次执行
  useUpdateEffect(effect, [flag])
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useThrottleEffect/index.ts)

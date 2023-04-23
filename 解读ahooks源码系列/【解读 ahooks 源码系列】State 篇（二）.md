# 【解读 ahooks 源码系列】State 篇（二）

本文是 ahooks 源码（[v3.7.4](https://github.com/alibaba/hooks/tree/v3.7.4)）系列的第九篇——State 篇（二）

往期文章：

- [【解读 ahooks 源码系列】（开篇）如何获取和监听 DOM 元素](https://juejin.cn/post/7201889983170592824)：useEffectWithTarget
- [【解读 ahooks 源码系列】DOM 篇（一）](https://juejin.cn/post/7202254039215800375)：useEventListener、useClickAway、useDocumentVisibility、useDrop、useDrag
- [【解读 ahooks 源码系列】DOM 篇（二）](https://juejin.cn/post/7202633255043465271)：useEventTarget、useExternal、useTitle、useFavicon、useFullscreen、useHover
- [【解读 ahooks 源码系列】DOM 篇（三）](https://juejin.cn/post/7202996870251380791)：useMutationObserver、useInViewport、useKeyPress、useLongPress
- [【解读 ahooks 源码系列】DOM 篇（四）](https://juejin.cn/post/7203397626527891515)：useMouse、useResponsive、useScroll、useSize、useFocusWithin
- [【解读 ahooks 源码系列】Dev 篇——useTrackedEffect 和 useWhyDidYouUpdate](https://juejin.cn/post/7204683121374232635)
- [【解读 ahooks 源码系列】Advanced 篇](https://juejin.cn/post/7207810396420669477)：useControllableValue、useCreation、useIsomorphicLayoutEffect、useEventEmitter、useLatest、useMemoizedFn、useReactive
- [【解读 ahooks 源码系列】State 篇（一）](https://juejin.cn/post/7210786286570324025)：useSetState、useToggle、useBoolean、useCookieState、useLocalStorageState、useSessionStorageState、useDebounce、useThrottle

本文主要解读 `useMap`、`useSet`、`usePrevious`、`useRafState`、`useSafeState`、`useGetState`、`useResetState` 的源码实现

## useMap

管理 Map 类型状态的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-map)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usemap-demo1/)

```ts
import React from 'react'
import { useMap } from 'ahooks'

export default () => {
  const [map, { set, setAll, remove, reset, get }] = useMap<string | number, string>([
    ['msg', 'hello world'],
    [123, 'number type'],
  ])

  return (
    <div>
      <button type="button" onClick={() => set(String(Date.now()), new Date().toJSON())}>
        Add
      </button>
      <button type="button" onClick={() => setAll([['text', 'this is a new Map']])} style={{ margin: '0 8px' }}>
        Set new Map
      </button>
      <button type="button" onClick={() => remove('msg')} disabled={!get('msg')}>
        Remove 'msg'
      </button>
      <button type="button" onClick={() => reset()} style={{ margin: '0 8px' }}>
        Reset
      </button>
      <div style={{ marginTop: 16 }}>
        <pre>{JSON.stringify(Array.from(map), null, 2)}</pre>
      </div>
    </div>
  )
}
```

### API

```ts
const [
  map, // Map 对象
  {
    set, // 添加元素
    setAll, // 生成一个新的 Map 对象
    remove, // remove
    reset, // 重置为默认值
    get // 获取元素
  }
] = useMap(initialValue?: Iterable<[any, any]>);
```

### Map

Map 对象保存键值对，并且能够记住键的原始插入顺序。任何值（对象或者基本类型）都可以作为一个键或一个值。详情可以看[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Map)

### 核心实现

- 由于 React state 是不可变数据，所以需要每次更改都需要创建一个新的 Map 对象。

```ts
function useMap<K, T>(initialValue?: Iterable<readonly [K, T]>) {
  // 获取默认的 Map 参数
  const getInitValue = () => {
    return initialValue === undefined ? new Map() : new Map(initialValue)
  }

  const [map, setMap] = useState<Map<K, T>>(() => getInitValue())

  // 添加元素
  const set = (key: K, entry: T) => {
    setMap((prev) => {
      const temp = new Map(prev)
      temp.set(key, entry)
      return temp
    })
  }

  // 生成一个新的 Map 对象
  const setAll = (newMap: Iterable<readonly [K, T]>) => {
    setMap(new Map(newMap))
  }

  // 移除元素
  const remove = (key: K) => {
    setMap((prev) => {
      const temp = new Map(prev)
      temp.delete(key)
      return temp
    })
  }

  // 重置为默认值
  const reset = () => setMap(getInitValue())

  // 获取元素
  const get = (key: K) => map.get(key)

  return [
    map,
    {
      // useMemoizedFn 持久化导出函数
      set: useMemoizedFn(set),
      setAll: useMemoizedFn(setAll),
      remove: useMemoizedFn(remove),
      reset: useMemoizedFn(reset),
      get: useMemoizedFn(get),
    },
  ] as const
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useMap/index.ts)

## useSet

管理 Set 类型状态的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-set)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/useset-demo1/)

```ts
import React from 'react'
import { useSet } from 'ahooks'

export default () => {
  const [set, { add, remove, reset }] = useSet(['Hello'])

  return (
    <div>
      <button type="button" onClick={() => add(String(Date.now()))}>
        Add Timestamp
      </button>
      <button type="button" onClick={() => remove('Hello')} disabled={!set.has('Hello')} style={{ margin: '0 8px' }}>
        Remove Hello
      </button>
      <button type="button" onClick={() => reset()}>
        Reset
      </button>
      <div style={{ marginTop: 16 }}>
        <pre>{JSON.stringify(Array.from(set), null, 2)}</pre>
      </div>
    </div>
  )
}
```

### API

```ts
const [
  set, // Set 对象
  {
    add, // 添加元素
    remove, // 移除元素
    reset // 重置为默认值
  }
] = useSet(initialValue?: Iterable<K>);
```

### Set

Set 对象允许你存储任何类型的唯一值，无论是原始值或者是对象引用。详情可以看[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Set)

### 核心实现

- 由于 React state 是不可变数据，所以需要每次更改都需要创建一个新的 Set 对象。

```ts
function useSet<K>(initialValue?: Iterable<K>) {
  // 获取默认值
  const getInitValue = () => {
    // 通过 new Set() 构造函数，创建一个新的 Set 对象
    return initialValue === undefined ? new Set<K>() : new Set(initialValue)
  }

  const [set, setSet] = useState<Set<K>>(() => getInitValue())

  // 添加元素
  const add = (key: K) => {
    if (set.has(key)) {
      return
    }
    setSet((prevSet) => {
      const temp = new Set(prevSet)
      temp.add(key) // 在 Set 对象尾部添加一个元素。返回该 Set 对象。
      return temp
    })
  }

  // 移除元素
  const remove = (key: K) => {
    if (!set.has(key)) {
      return
    }
    setSet((prevSet) => {
      const temp = new Set(prevSet)
      temp.delete(key)
      return temp
    })
  }

  // 重置为默认值
  const reset = () => setSet(getInitValue())

  return [
    set,
    {
      add: useMemoizedFn(add),
      remove: useMemoizedFn(remove),
      reset: useMemoizedFn(reset),
    },
  ] as const
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useSet/index.ts)

## usePrevious

保存上一次状态的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-previous)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/useprevious-demo1/)

记录上次的 count 值

```ts
import { usePrevious } from 'ahooks'
import React, { useState } from 'react'

export default () => {
  const [count, setCount] = useState(0)
  const previous = usePrevious(count)
  return (
    <>
      <div>counter current value: {count}</div>
      <div style={{ marginBottom: 8 }}>counter previous value: {previous}</div>
      <button type="button" onClick={() => setCount((c) => c + 1)}>
        increase
      </button>
      <button type="button" style={{ marginLeft: 8 }} onClick={() => setCount((c) => c - 1)}>
        decrease
      </button>
    </>
  )
}
```

### 使用场景

实现新旧值的对比来处理一些逻辑

### 实现思路

每次状态变更的时候来比较值有没有发生变化：

1. 需要维护两个状态： prevRef（保存上一次状态值）和 curRef（当前状态值）
2. state 状态变更的时候，使用 shouldUpdate 参数判断是否发生变化。如果发生变化，先更新 prevRef 的值为上一个 curRef，将 curRef 的值更新为当前最新 state 值
3. shouldUpdate 支持自定义，由开发结合自身场景判断值是否变化，来更新上一次状态

### 核心实现

```ts
// 默认判断是否需要更新的函数
const defaultShouldUpdate = <T>(a?: T, b?: T) => !Object.is(a, b)

function usePrevious<T>(
  // 需要记录变化的值
  state: T,
  // 自定义判断值是否变化
  shouldUpdate: ShouldUpdateFunc<T> = defaultShouldUpdate
): T | undefined {
  const prevRef = useRef<T>() // 保存上一次状态值
  const curRef = useRef<T>() // 当前状态值

  // 自定义 shouldUpdate 函数，判断值是否变化
  if (shouldUpdate(curRef.current, state)) {
    prevRef.current = curRef.current
    curRef.current = state
  }

  return prevRef.current
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/usePrevious/index.ts)

## useRafState

只在 [requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame) callback 时更新 state，一般用于性能优化。用法与 React.useState 一致

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-raf-state)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/zh-CN/hooks/use-raf-state)

```ts
import { useRafState } from 'ahooks'
import React, { useEffect } from 'react'

export default () => {
  const [state, setState] = useRafState({
    width: 0,
    height: 0,
  })

  useEffect(() => {
    const onResize = () => {
      setState({
        width: document.documentElement.clientWidth,
        height: document.documentElement.clientHeight,
      })
    }
    onResize()

    window.addEventListener('resize', onResize)

    return () => {
      window.removeEventListener('resize', onResize)
    }
  }, [])

  return (
    <div>
      <p>Try to resize the window </p>
      current: {JSON.stringify(state)}
    </div>
  )
}
```

### requestAnimationFrame

> window.requestAnimationFrame()：告诉浏览器——你希望执行一个动画，并且要求浏览器在下次重绘之前调用指定的回调函数更新动画。该方法需要传入一个回调函数作为参数，该回调函数会在浏览器下一次重绘之前执行

与 setTimeout 相比，requestAnimationFrame 最大的优势是由系统来决定回调函数的执行时机，它能保证回调函数在屏幕每一次的刷新间隔中只被执行一次，这样就不会引起丢帧现象，也不会导致动画出现卡顿的问题。

> window.cancelAnimationFrame：取消一个先前通过调用 window.requestAnimationFrame()方法添加到计划中的动画帧请求。

### 使用场景

- state 操作是比较频繁的
- 实现频繁的动画效果

### 核心实现

主要是实现 setRafState 方法，在外部调用 setRafState 方法时，会取消上一次的 setState 回调函数，并执行 requestAnimationFrame 来控制 setState 的执行时机

```ts
function useRafState<S>(initialState?: S | (() => S)) {
  const ref = useRef(0)
  const [state, setState] = useState(initialState)

  const setRafState = useCallback((value: S | ((prevState: S) => S)) => {
    // 先取消上一次的 setRafState 操作
    cancelAnimationFrame(ref.current)

    ref.current = requestAnimationFrame(() => {
      // 在回调执行真正的 setState
      setState(value)
    })
  }, [])

  // 页面卸载时取消回调函数
  useUnmount(() => {
    cancelAnimationFrame(ref.current)
  })

  return [state, setRafState] as const
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useRafState/index.ts)

## useSafeState

用法与 React.useState 完全一样，但是在组件卸载后异步回调内的 setState 不再执行，避免因组件卸载后更新状态而导致的内存泄漏。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-safe-state)

警告内容如下：

> Warning: Can't perform a React state update on an unmounted component. This is a no-op, but it indicates a memory leak in your application. To fix, cancel all subscriptions and asynchronous tasks in a useEffect cleanup function.

升级 React 18 后官方已经移除了该警告，所以后续无需考虑该告警了，也不再需要这个 useSafeState Hook 了，详情可见该文章：[React 18 对 Hooks 的影响：一](https://juejin.cn/post/7081171944799731720)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usesafestate-demo1/)

```ts
import { useSafeState } from 'ahooks'
import React, { useEffect, useState } from 'react'

const Child = () => {
  const [value, setValue] = useSafeState<string>()

  useEffect(() => {
    setTimeout(() => {
      setValue('data loaded from server')
    }, 5000)
  }, [])

  const text = value || 'Loading...'

  return <div>{text}</div>
}

export default () => {
  const [visible, setVisible] = useState(true)

  return (
    <div>
      <button onClick={() => setVisible(false)}>Unmount</button>
      {visible && <Child />}
    </div>
  )
}
```

### 核心实现

内部使用了[useUnmountedRef](https://ahooks.js.org/zh-CN/hooks/use-unmounted-ref) 这个 Hook 来获取当前组件是否已卸载，该 Hook 原理是通过判断有无执行 useEffect 的卸载函数，在其标识为已卸载。

```ts
useEffect(() => {
  return () => {
    // 可设置卸载标识
  }
}, [])
```

useSafeState 实现原理则是依据 unmountedRef 标识，在外部执行 `setCurrentState` 的时候，判断如果标识为 true（已卸载），则 return 停止更新

```ts
function useSafeState<S>(initialState?: S | (() => S)) {
  // useUnmountedRef：获取当前组件是否已卸载
  const unmountedRef = useUnmountedRef()
  const [state, setState] = useState(initialState)
  const setCurrentState = useCallback((currentState) => {
    // 如果组件卸载了则停止更新
    if (unmountedRef.current) return
    setState(currentState)
  }, [])

  return [state, setCurrentState] as const
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useSafeState/index.ts)

## useGetState

给 React.useState 增加了一个 getter 方法，以获取当前最新值。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-get-state)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usegetstate-demo1/)

计数器每 3 秒打印一次值

```ts
import React, { useEffect } from 'react'
import { useGetState } from 'ahooks'

export default () => {
  const [count, setCount, getCount] = useGetState<number>(0)

  useEffect(() => {
    const interval = setInterval(() => {
      // 在这里使用 count 无法获取到最新值
      console.log('interval count', getCount())
    }, 3000)

    return () => {
      clearInterval(interval)
    }
  }, [])

  return <button onClick={() => setCount((count) => count + 1)}>count: {count}</button>
}
```

### 核心实现

实现原理是使用 useRef 来保存最新的 state 值，暴露一个 getState 直接返回 stateRef.current 即可

```ts
function useGetState<S>(initialState?: S) {
  const [state, setState] = useState(initialState)
  // 使用 useRef 保存最新 state
  const stateRef = useRef(state)
  stateRef.current = state
  // 获取当前最新值
  const getState = useCallback(() => stateRef.current, [])

  return [state, setState, getState]
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useGetState/index.ts)

## useResetState

提供重置 state 方法的 Hooks，用法与 React.useState 基本一致。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-reset-state)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/useresetstate-demo1/)

```ts
import React from 'react'
import { useResetState } from 'ahooks'

interface State {
  hello: string
  count: number
}

export default () => {
  const [state, setState, resetState] = useResetState<State>({
    hello: '',
    count: 0,
  })

  return (
    <div>
      <pre>{JSON.stringify(state, null, 2)}</pre>
      <p>
        <button type="button" style={{ marginRight: '8px' }} onClick={() => setState({ hello: 'world', count: 1 })}>
          set hello and count
        </button>

        <button type="button" onClick={resetState}>
          resetState
        </button>
      </p>
    </div>
  )
}
```

### 核心实现

实现原理是直接使用初始值作为 setState 的参数。说白了就是语义化（提供 reset 开头命名的函数）和偷懒（少传了个初始值参数）的写法。

```ts
const useResetState = <S>(initialState: S | (() => S)): [S, Dispatch<SetStateAction<S>>, ResetState] => {
  const [state, setState] = useState(initialState)

  // 重置 state
  // useMemoizedFn：持久化函数 Hook
  const resetState = useMemoizedFn(() => {
    setState(initialState)
  })

  return [state, setState, resetState]
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useResetState/index.ts)

# 【解读 ahooks 源码系列】State 篇（一）

## 前言

本文是 ahooks 源码（[v3.7.4](https://github.com/alibaba/hooks/tree/v3.7.4)）系列的第八篇——State 篇（一）

往期文章：

- [【解读 ahooks 源码系列】（开篇）如何获取和监听 DOM 元素](https://juejin.cn/post/7201889983170592824)：useEffectWithTarget
- [【解读 ahooks 源码系列】DOM 篇（一）](https://juejin.cn/post/7202254039215800375)：useEventListener、useClickAway、useDocumentVisibility、useDrop、useDrag
- [【解读 ahooks 源码系列】DOM 篇（二）](https://juejin.cn/post/7202633255043465271)：useEventTarget、useExternal、useTitle、useFavicon、useFullscreen、useHover
- [【解读 ahooks 源码系列】DOM 篇（三）](https://juejin.cn/post/7202996870251380791)：useMutationObserver、useInViewport、useKeyPress、useLongPress
- [【解读 ahooks 源码系列】DOM 篇（四）](https://juejin.cn/post/7203397626527891515)：useMouse、useResponsive、useScroll、useSize、useFocusWithin
- [【解读 ahooks 源码系列】Dev 篇——useTrackedEffect 和 useWhyDidYouUpdate](https://juejin.cn/post/7204683121374232635)
- [【解读 ahooks 源码系列】Advanced 篇](https://juejin.cn/post/7207810396420669477)：useControllableValue、useCreation、useIsomorphicLayoutEffect、useEventEmitter、useLatest、useMemoizedFn、useReactive

本文主要解读 `useSetState`、`useToggle`、`useBoolean`、`useCookieState`、`useLocalStorageState`、`useSessionStorageState`、`useDebounce`、`useThrottle` 的源码实现

## useSetState

管理 object 类型 state 的 Hooks，用法与 class 组件的 this.setState 基本一致。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-set-state/)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usesetstate-demo1/)

```ts
import React from 'react'
import { useSetState } from 'ahooks'

interface State {
  hello: string
  count: number
  [key: string]: any
}

export default () => {
  const [state, setState] = useSetState<State>({
    hello: '',
    count: 0,
  })

  return (
    <div>
      <pre>{JSON.stringify(state, null, 2)}</pre>
      <p>
        <button type="button" onClick={() => setState({ hello: 'world' })}>
          set hello
        </button>
        <button type="button" onClick={() => setState({ foo: 'bar' })} style={{ margin: '0 8px' }}>
          set foo
        </button>
        <button type="button" onClick={() => setState((prev) => ({ count: prev.count + 1 }))}>
          count + 1
        </button>
      </p>
    </div>
  )
}
```

### 使用场景

setState 对象时想省略合并运算符，保证每一次设置值都是自动合并

### 实现思路

该 Hook 主要就是内部做了自动合并操作处理

可见 [官网 useState 解释](https://zh-hans.reactjs.org/docs/hooks-reference.html#usestate)

> 主要原因是因为 useState 不会自动合并更新对象，大部分情况下需要我们自己手动合并，因此提供了 useSetState hooks 来解决这个问题

```ts
const [state, setState] = useState({})
setState((prevState) => {
  // 也可以使用 Object.assign
  return { ...prevState, ...updatedValues }
})
```

### 核心实现

```ts
const useSetState = <S extends Record<string, any>>(initialState: S | (() => S)): [S, SetState<S>] => {
  const [state, setState] = useState<S>(initialState)

  // 自定义新的 setState 函数，返回自动合并的值
  const setMergeState = useCallback((patch) => {
    setState((prevState) => {
      // 传入的 patch 值是否为函数：如果是函数则执行，表示旧的状态。否则直接作为新的状态值
      const newState = isFunction(patch) ? patch(prevState) : patch
      // 拓展运算符合并返回新的对象
      return newState ? { ...prevState, ...newState } : prevState
    })
  }, [])

  return [state, setMergeState]
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useSetState/index.ts)

## useToggle

用于在两个状态值间切换的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-toggle)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usesetstate-demo2/)

接受两个可选参数，在它们之间进行切换。

```ts
import React from 'react'
import { useToggle } from 'ahooks'

export default () => {
  // Hello 表示左值（默认值）， World 表示右值（取反的状态值）
  const [state, { toggle, set, setLeft, setRight }] = useToggle('Hello', 'World')

  return (
    <div>
      <p>Effects：{state}</p>
      <p>
        <button type="button" onClick={toggle}>
          Toggle
        </button>
        <button type="button" onClick={() => set('Hello')} style={{ margin: '0 8px' }}>
          Set Hello
        </button>
        <button type="button" onClick={() => set('World')}>
          Set World
        </button>
        <button type="button" onClick={setLeft} style={{ margin: '0 8px' }}>
          Set Left
        </button>
        <button type="button" onClick={setRight}>
          Set Right
        </button>
      </p>
    </div>
  )
}
```

**相关字段解释**

- state：状态值
- defaultValue：传入默认的状态值
- reverseValue：传入取反的状态值
- toggle：切换 state
- set：修改 state
- setLeft：设置为 defaultValue
- setRight：如果传入了 reverseValue, 则设置为 reverseValue。 否则设置为 defaultValue 的反值

先来看看它的类型定义

- 函数重载：针对不同参数个数和类型，推断返回值类型

```ts
const [state, { toggle, set, setLeft, setRight }] = useToggle(defaultValue?: boolean);
const [state, { toggle, set, setLeft, setRight }] = useToggle<T>(defaultValue: T);
const [state, { toggle, set, setLeft, setRight }] = useToggle<T, U>(defaultValue: T, reverseValue: U);
```

### 核心实现

实现比较简单：

1. 入参可传有两个值，第一个参数是默认值（左值）；第二个是取反之后的值（右值），可以不传；当不传的时候，为 defaultValue 的反值
2. 根据这两个值，实现函数 toggle、set、setLeft、setRight（该 Hook 忽略 defaultValue、reverseValue 这两个值的变更，也就是说无需监听；在使用中也需要注意，这两个值是固定值才可使用该 Hook）

```ts
function useToggle<D, R>(defaultValue: D = false as unknown as D, reverseValue?: R) {
  const [state, setState] = useState<D | R>(defaultValue)

  const actions = useMemo(() => {
    // 取反的状态值
    const reverseValueOrigin = (reverseValue === undefined ? !defaultValue : reverseValue) as D | R

    // 切换值（左值与右值）
    const toggle = () => setState((s) => (s === defaultValue ? reverseValueOrigin : defaultValue))
    // 修改 state
    const set = (value: D | R) => setState(value)
    // 修改 state
    const setLeft = () => setState(defaultValue)
    // setRight：如果传入了 reverseValue, 则设置为 reverseValue。 否则设置为 defaultValue 的反值
    const setRight = () => setState(reverseValueOrigin)

    return {
      toggle,
      set,
      setLeft,
      setRight,
    }
    // useToggle ignore value change
    // }, [defaultValue, reverseValue]);
  }, [])

  return [state, actions]
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useToggle/index.ts)

## useBoolean

优雅的管理 boolean 状态的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-boolean)

### 基本用法

上面讲了 `useToggle`，而 `useBoolean` 是 useToggle 的其中一种使用场景，下面是 useToggle 的其中一种函数类型定义：

```ts
const [state, { toggle, set, setLeft, setRight }] = useToggle(defaultValue?: boolean);
```

[官方在线 Demo](https://ahooks.js.org/~demos/useboolean-demo1/)

切换 boolean，可以接收默认值。

```ts
import React from 'react'
import { useBoolean } from 'ahooks'

export default () => {
  const [state, { toggle, setTrue, setFalse }] = useBoolean(true)

  return (
    <div>
      <p>Effects：{JSON.stringify(state)}</p>
      <p>
        <button type="button" onClick={toggle}>
          Toggle
        </button>
        <button type="button" onClick={setFalse} style={{ margin: '0 16px' }}>
          Set false
        </button>
        <button type="button" onClick={setTrue}>
          Set true
        </button>
      </p>
    </div>
  )
}
```

**相关字段解释**

- toggle：切换 state
- set：设置 state
- setTrue：设置为 true
- setFalse：设置为 false

### 核心实现

有了 useToggle 的基础，实现比较简单，直接看代码：

```ts
export default function useBoolean(defaultValue = false): [boolean, Actions] {
  const [state, { toggle, set }] = useToggle(defaultValue)

  const actions: Actions = useMemo(() => {
    const setTrue = () => set(true)
    const setFalse = () => set(false)
    return {
      toggle,
      set: (v) => set(!!v),
      setTrue,
      setFalse,
    }
  }, [])

  return [state, actions]
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useBoolean/index.ts)

## useCookieState

一个可以将状态存储在 Cookie 中的 Hook 。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-cookie-state)

### 基本用法

将 state 存储在 Cookie 中

[官方在线 Demo](https://ahooks.js.org/~demos/usecookiestate-demo1/)

刷新页面后，可以看到输入框中的内容被从 Cookie 中恢复了。

```ts
import React from 'react'
import { useCookieState } from 'ahooks'

export default () => {
  // useCookieStateString 表示 Cookie 的 key 值
  const [message, setMessage] = useCookieState('useCookieStateString')
  return (
    <input
      value={message}
      placeholder="Please enter some words..."
      onChange={(e) => setMessage(e.target.value)}
      style={{ width: 300 }}
    />
  )
}
```

### Cookie 相关

- [Document.cookie](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/cookie)：获取并设置与当前文档相关联的 cookie。可以把它当成一个 getter and setter

JS 操作 cookie 常用的库是 [js-cookie](https://www.npmjs.com/package/js-cookie)，js-cookie 是一个上手简单，轻量的，处理 cookies 的库。它的优点是：

1. **简单易用**：直接通过 `js-cookie` 的 API 可以很容易操作 cookie
2. **轻量级**：`js-cookie` 压缩后小于 800 字节
3. **支持所有浏览器**
4. **安全性高**：`js-cookie` 带有防止 XSS 攻击的处理机制。它保证了 Cookie 的安全性，可以预防网络劫持或脚本注入等攻击方式。
5. **支持 ESM/AMD/CommonJs**

### 核心实现

useCookieState 返回的是`[state, setState]`格式

默认值的实现：

1. （优先级最高）如果本地 cookie 中已有该值，则直接读取。
2. 外部设置的默认值是函数则执行。否则直接返回（options.defaultValue）
3. 需要注意的是 options.defaultValue 定义的 Cookie 默认值，但不同步到本地 Cookie

```ts
function useCookieState(cookieKey: string, options: Options = {}) {
  const [state, setState] = useState<State>(() => {
    const cookieValue = Cookies.get(cookieKey)

    // 如果本地 cookie 中已有该值，则直接读取
    if (isString(cookieValue)) return cookieValue
    // 外部设置的默认值是函数则执行
    if (isFunction(options.defaultValue)) {
      return options.defaultValue()
    }
    // 返回外部传入的默认值
    return options.defaultValue
  })

  // 设置 Cookie 值
  const updateState = useMemoizedFn(
    (newValue: State | ((prevState: State) => State), newOptions: Cookies.CookieAttributes = {}) => {
      // setState 可以更新 cookie options，会与 useCookieState 设置的 options 进行 merge 操作。
      const { defaultValue, ...restOptions } = { ...options, ...newOptions }
      setState((prevState) => {
        const value = isFunction(newValue) ? newValue(prevState) : newValue
        // 值为 undefined 则清除 cookie
        if (value === undefined) {
          Cookies.remove(cookieKey)
        } else {
          // 设置 cookie
          Cookies.set(cookieKey, value, restOptions)
        }
        return value
      })
    }
  )

  return [state, updateState] as const
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useCookieState/index.ts)

## useLocalStorageState

将状态存储在 localStorage 中的 Hook 。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-local-storage-state)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/uselocalstoragestate-demo1)

将 state 存储在 localStorage 中

```ts
import React from 'react'
import { useLocalStorageState } from 'ahooks'

export default function () {
  const [message, setMessage] = useLocalStorageState<string | undefined>('use-local-storage-state-demo1', {
    defaultValue: 'Hello~',
  })

  return (
    <>
      <input
        value={message || ''}
        placeholder="Please enter some words..."
        onChange={(e) => setMessage(e.target.value)}
      />
      <button style={{ margin: '0 8px' }} type="button" onClick={() => setMessage('Hello~')}>
        Reset
      </button>
      <button type="button" onClick={() => setMessage(undefined)}>
        Clear
      </button>
    </>
  )
}
```

### 核心实现

实际上是实现了 createUseStorageState 方法，useLocalStorageState 是调用了 createUseStorageState 返回的结果。

useLocalStorageState 在往 localStorage 写入数据前，会先调用一次 serializer，在读取数据之后，会先调用一次 deserializer

```ts
// 判断是否为浏览器环境
const useLocalStorageState = createUseStorageState(() => (isBrowser ? localStorage : undefined))
```

- serializer：序列化方法（存入 storage 使用）
- deserializer：反序列化方法（从 storage 取出）
- getStoredValue：获取 storage 的值
- updateState： 更新 storage 状态值

```ts
// 序列化
const serializer = (value: T) => {
  if (options?.serializer) {
    // 支持自定义序列化
    return options?.serializer(value)
  }
  return JSON.stringify(value)
}

// 反序列化
const deserializer = (value: string) => {
  if (options?.deserializer) {
    // 支持自定义反序列化
    return options?.deserializer(value)
  }
  return JSON.parse(value)
}

// 获取 storage 的值
function getStoredValue() {
  try {
    const raw = storage?.getItem(key)
    if (raw) {
      // 反序列化取出值
      return deserializer(raw)
    }
  } catch (e) {
    console.error(e)
  }
  // raw 没值，则使用默认值
  if (isFunction(options?.defaultValue)) {
    return options?.defaultValue()
  }
  return options?.defaultValue
}
```

对于普通的字符串，可能不需要默认的 JSON.stringify/JSON.parse 来序列化。

```ts
serializer: (v) => v ?? '',
deserializer: (v) => v,
```

再来看下 updateState 方法：

- 如果传入函数，优先取值函数执行后的结果
- 传入 undefined，则表示删除这条数据
- 否则直接设置值

```ts
// 定义 state 状态同步拿到 storage 值
const [state, setState] = useState<T>(() => getStoredValue())

// 当 key 更新的时候执行
// useUpdateEffect：忽略首次执行，只在依赖更新时执行
useUpdateEffect(() => {
  setState(getStoredValue())
}, [key])

// 更新 storage 状态值
const updateState = (value: T | IFuncUpdater<T>) => {
  // 传入函数优先取函数执行后的结果
  const currentState = isFunction(value) ? value(state) : value
  setState(currentState)

  // 值为 undefined，表示移除该 storage
  if (isUndef(currentState)) {
    storage?.removeItem(key)
  } else {
    // 否则直接设置值
    try {
      storage?.setItem(key, serializer(currentState))
    } catch (e) {
      console.error(e)
    }
  }
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useLocalStorageState/index.ts)

## useSessionStorageState

将状态存储在 sessionStorage 中的 Hook。

同样调用了 createUseStorageState 方法，只需把 localStorage 改为 sessionStorage，其它一致；
这里就不展开写了

```ts
const useSessionStorageState = createUseStorageState(() => (isBrowser ? sessionStorage : undefined))
```

## useDebounce

用来处理防抖值的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-debounce)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usedebounce-demo1/)

DebouncedValue 只会在输入结束 500ms 后变化。

```ts
import React, { useState } from 'react'
import { useDebounce } from 'ahooks'

export default () => {
  const [value, setValue] = useState<string>()
  const debouncedValue = useDebounce(value, { wait: 500 })

  return (
    <div>
      <input
        value={value}
        onChange={(e) => setValue(e.target.value)}
        placeholder="Typed value"
        style={{ width: 280 }}
      />
      <p style={{ marginTop: 16 }}>DebouncedValue: {debouncedValue}</p>
    </div>
  )
}
```

### 核心实现

来看看支持的选项，都是 [lodash.debounce](https://lodash.com/docs/4.17.15#debounce) 里面的参数：

```ts
interface DebounceOptions {
  wait?: number // 等待时间，单位为毫秒
  leading?: boolean // 是否在延迟开始前调用函数
  trailing?: boolean // 是否在延迟开始后调用函数
  maxWait?: number // 最大等待时间，单位为毫秒
}
```

看代码实现主要是依赖 `useDebounceFn` 这个 Hook，这个 Hook 内部使用的是 lodash 的 debounce 方法。

```ts
function useDebounce<T>(value: T, options?: DebounceOptions) {
  const [debounced, setDebounced] = useState(value)

  const { run } = useDebounceFn(() => {
    setDebounced(value)
  }, options)

  // 监听需要防抖的值变化
  useEffect(() => {
    run() // 变化就执行 debounced 函数
  }, [value])

  return debounced
}
```

useDebounceFn 的实现：

```ts
/** 用来处理防抖函数的 Hook。 */
function useDebounceFn<T extends noop>(fn: T, options?: DebounceOptions) {
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

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useDebounce/index.ts)

## useThrottle

用来处理节流值的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-throttle)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usethrottle-demo1/)

ThrottledValue 每隔 500ms 变化一次。

```ts
import React, { useState } from 'react'
import { useThrottle } from 'ahooks'

export default () => {
  const [value, setValue] = useState<string>()
  const throttledValue = useThrottle(value, { wait: 500 })

  return (
    <div>
      <input
        value={value}
        onChange={(e) => setValue(e.target.value)}
        placeholder="Typed value"
        style={{ width: 280 }}
      />
      <p style={{ marginTop: 16 }}>throttledValue: {throttledValue}</p>
    </div>
  )
}
```

### 核心实现

来看看支持的选项，都是 [lodash.throttle](https://lodash.com/docs/4.17.15#throttle) 里面的参数：

```ts
interface ThrottleOptions {
  wait?: number // 等待时间，单位为毫秒
  leading?: boolean // 是否在延迟开始前调用函数
  trailing?: boolean // 是否在延迟开始后调用函数
}
```

看代码实现主要是依赖 `useThrottleFn` 这个 Hook，这个 Hook 内部使用的是 lodash 的 throttle 方法。

```ts
function useThrottle<T>(value: T, options?: ThrottleOptions) {
  const [throttled, setThrottled] = useState(value)

  const { run } = useThrottleFn(() => {
    setThrottled(value)
  }, options)

  useEffect(() => {
    run()
  }, [value])

  return throttled
}
```

useThrottleFn 的实现：

```ts
function useThrottleFn<T extends noop>(fn: T, options?: ThrottleOptions) {
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

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useThrottle/index.ts)

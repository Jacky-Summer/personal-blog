# 【解读 ahooks 源码系列】Advanced 篇

## 前言

本文是 ahooks 源码（[v3.7.4](https://github.com/alibaba/hooks/tree/v3.7.4)）系列的第七篇——Advanced 篇

往期文章：

- [【解读 ahooks 源码系列】（开篇）如何获取和监听 DOM 元素](https://juejin.cn/post/7201889983170592824)：useEffectWithTarget
- [【解读 ahooks 源码系列】DOM 篇（一）](https://juejin.cn/post/7202254039215800375)：useEventListener、useClickAway、useDocumentVisibility、useDrop、useDrag
- [【解读 ahooks 源码系列】DOM 篇（二）](https://juejin.cn/post/7202633255043465271)：useEventTarget、useExternal、useTitle、useFavicon、useFullscreen、useHover
- [【解读 ahooks 源码系列】DOM 篇（三）](https://juejin.cn/post/7202996870251380791)：useMutationObserver、useInViewport、useKeyPress、useLongPress
- [【解读 ahooks 源码系列】DOM 篇（四）](https://juejin.cn/post/7203397626527891515)：useMouse、useResponsive、useScroll、useSize、useFocusWithin
- [解读 ahooks 源码系列】Dev 篇——useTrackedEffect 和 useWhyDidYouUpdate](https://juejin.cn/post/7204683121374232635)

本文主要解读 `useControllableValue`、`useCreation`、`useEventEmitter`、`useIsomorphicLayoutEffect`、`useLatest`、`useMemoizedFn`、`useReactive` 的源码实现

## useControllableValue

在某些组件开发时，我们需要组件的状态既可以自己管理，也可以被外部控制，useControllableValue 就是帮你管理这种状态的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/https://ahooks.js.org/zh-CN/hooks/use-controllable-value)

### 基本用法

- 非受控组件：如果 props 中没有 value，则组件内部自己管理 state
- 受控组件：如果 props 有 value 字段，则由父级接管控制 state
- 无 value，有 onChange 的组件：只要 props 中有 onChange 字段，则在 state 变化时，就会触发 onChange 函数

[官方在线 Demo](https://ahooks.js.org/~demos/usecontrollablevalue-demo2/)

如果 props 有 value 字段，则由父级接管控制 state：

```ts
import React, { useState } from 'react'
import { useControllableValue } from 'ahooks'

const ControllableComponent = (props: any) => {
  const [state, setState] = useControllableValue<string>(props)

  return <input value={state} onChange={(e) => setState(e.target.value)} style={{ width: 300 }} />
}

const Parent = () => {
  const [state, setState] = useState<string>('')
  const clear = () => {
    setState('')
  }

  return (
    <>
      <ControllableComponent value={state} onChange={setState} />
      <button type="button" onClick={clear} style={{ marginLeft: 8 }}>
        Clear
      </button>
    </>
  )
}
```

### 受控组件与非受控组件

React 官方对[受控组件](https://zh-hans.reactjs.org/docs/forms.html#controlled-components)和[非受控组件](https://zh-hans.reactjs.org/docs/uncontrolled-components.html)的解释：

> - 受控组件：在 HTML 中，表单元素（如`<input>`、 `<textarea>` 和 `<select>`）通常自己维护 state，并根据用户输入进行更新。而在 React 中，可变状态（mutable state）通常保存在组件的 state 属性中，并且只能通过使用 setState()来更新。
> - 非受控组件：表单数据将交由 DOM 节点来处理。

但是受控组件/非受控组件又是一个相对的概念，子组件相对父组件来说才有受控/非受控的说法。当组件中有数据受父级组件的控制（比如数据的来源和修改的方式由父级组件的 props 提供），就是受控组件；而当组件的数据完全由组件自身维护，这样的组件是非受控组件。从这个角度看，antd 的 Input 组件既可以是受控也可以非受控，这取决于我们如何使用。

### 使用场景

- 表单组件既支持受控又要支持非受控的场景，目前很多 UI 库目前都基本支持这两种场景

### 实现思路

根据 props 是否有`[valuePropName]`属性来判断是否受控。如受控：则值由父级接管；否则组件内部状态维护；初始值的设置也遵循该逻辑。

### 核心实现

```ts
function useControllableValue<T = any>(props: StandardProps<T>): [T, (v: SetStateAction<T>) => void]
function useControllableValue<T = any>(
  props?: Props,
  options?: Options<T>
): [T, (v: SetStateAction<T>, ...args: any[]) => void]
function useControllableValue<T = any>(props: Props = {}, options: Options<T> = {}) {
  const {
    defaultValue, // 默认值，会被 props.defaultValue 和 props.value 覆盖
    defaultValuePropName = 'defaultValue', // 默认值的属性名
    valuePropName = 'value', // 值的属性名
    trigger = 'onChange', // 修改值时，触发的函数
  } = options
  // 外部（父级）传递进来的 props 值
  const value = props[valuePropName] as T
  // 是否受控：判断 valuePropName（默认即表示value属性），有该属性代表受控
  const isControlled = props.hasOwnProperty(valuePropName)

  // 首次默认值
  const initialValue = useMemo(() => {
    // 受控：则由外部的props接管控制 state
    if (isControlled) {
      return value
    }
    // 外部有传递 defaultValue，则优先取外部的默认值
    if (props.hasOwnProperty(defaultValuePropName)) {
      return props[defaultValuePropName]
    }
    // 优先级最低，组件内部的默认值
    return defaultValue
  }, [])

  const stateRef = useRef(initialValue)
  // 受控组件：如果 props 有 value 字段，则由父级接管控制 state
  if (isControlled) {
    stateRef.current = value
  }

  // update：调用该函数会强制组件重新渲染
  const update = useUpdate()

  function setState(v: SetStateAction<T>, ...args: any[]) {
    const r = isFunction(v) ? v(stateRef.current) : v

    // 非受控
    if (!isControlled) {
      stateRef.current = r
      update() // 更新状态
    }
    // 只要 props 中有 onChange（trigger 默认值未 onChange）字段，则在 state 变化时，就会触发 onChange 函数
    if (props[trigger]) {
      props[trigger](r, ...args)
    }
  }

  // 返回 [状态值, 修改 state 的函数]
  return [stateRef.current, useMemoizedFn(setState)] as const
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useControllableValue/index.ts)

## useCreation

useCreation 是 useMemo 或 useRef 的替代品。

因为 useMemo 不能保证被 memo 的值一定不会被重计算，而 useCreation 可以保证这一点。以下为 [React 官方文档](https://reactjs.org/docs/hooks-reference.html#usememo)中的介绍：

> You may rely on useMemo as a performance optimization, not as a semantic guarantee. In the future, React may choose to “forget” some previously memoized values and recalculate them on next render, e.g. to free memory for offscreen components. Write your code so that it still works without useMemo — and then add it to optimize performance.

而相比于 useRef，你可以使用 useCreation 创建一些常量，这些常量和 useRef 创建出来的 ref 有很多使用场景上的相似，但对于复杂常量的创建，useRef 却容易出现潜在的性能隐患。

```ts
const a = useRef(new Subject()) // 每次重渲染，都会执行实例化 Subject 的过程，即便这个实例立刻就被扔掉了
const b = useCreation(() => new Subject(), []) // 通过 factory 函数，可以避免性能隐患
```

[官方文档](https://ahooks.js.org/zh-CN/hooks/https://ahooks.js.org/zh-CN/hooks/use-creation)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usecreation-demo1/)

确保实例不会被重复创建。点击 "Rerender" 按钮，触发组件的更新，但 Foo 的实例会保持不变

```ts
import React, { useState } from 'react'
import { useCreation } from 'ahooks'

class Foo {
  constructor() {
    this.data = Math.random()
  }

  data: number
}

export default function () {
  const foo = useCreation(() => new Foo(), [])
  const [, setFlag] = useState({})
  return (
    <>
      <p>{foo.data}</p>
      <button
        type="button"
        onClick={() => {
          setFlag({})
        }}
      >
        Rerender
      </button>
    </>
  )
}
```

我们发现看下来用法与 useMemo 完全一致，算是 useMemo 的再优化版本

### 实现思路

useCreation 的核心依赖是 useRef

1. 把相关值（依赖，具体值，是否初始化）保存在 useRef 中，将值进行缓存
2. 初始化或依赖变更（检测 useRef 的旧值依赖与当前依赖用 `Object.is()`比对）时，不一致则进行更新。

### 核心实现

```ts
// 通过 Object.is 比较依赖数组的值是否相等
function depsAreSame(oldDeps: DependencyList, deps: DependencyList): boolean {
  if (oldDeps === deps) return true
  for (let i = 0; i < oldDeps.length; i++) {
    if (!Object.is(oldDeps[i], deps[i])) return false
  }
  return true
}

function useCreation<T>(factory: () => T, deps: DependencyList) {
  const { current } = useRef({
    deps,
    obj: undefined as undefined | T,
    initialized: false,
  })
  // 初始化或依赖变更时，重新初始化
  if (current.initialized === false || !depsAreSame(current.deps, deps)) {
    current.deps = deps // 更新依赖
    current.obj = factory() // 执行创建所需对象的函数
    current.initialized = true // 初始化标识为 true
  }
  return current.obj as T
}
```

这个 Hooks 属于是精益求精了，本来使用 `useMemo` 这个优化型的 Hook 就要考量场景，暂时还不知道哪些精细化场景用 `useCreation` 比较好

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useCreation/index.ts)

## useIsomorphicLayoutEffect

在 SSR 模式下，使用 `useLayoutEffect` 时，会出现警告，为了避免该警告,可以使用 `useIsomorphicLayoutEffect` 代替 `useLayoutEffect`。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-isomorphic-layout-effect)

### 核心实现

```ts
const isBrowser = !!(typeof window !== 'undefined' && window.document && window.document.createElement)

const useIsomorphicLayoutEffect = isBrowser ? useLayoutEffect : useEffect
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useIsomorphicLayoutEffect/index.ts)

## useEventEmitter

在多个组件之间进行事件通知有时会让人非常头疼，借助 EventEmitter ，可以让这一过程变得更加
简单。

```ts
const event$ = useEventEmitter()
```

通过 props 或者 Context ，可以将 `event$` 共享给其他组件。然后在其他组件中，可以调用 EventEmitter 的方法

[官方文档](https://ahooks.js.org/zh-CN/hooks/https://ahooks.js.org/zh-CN/hooks/use-event-emitter)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/useeventemitter-demo1/)

父组件向子组件共享事件

父组件创建了一个 `focus$` 事件，并且将它传递给了两个子组件。在 MessageBox 中调用 `focus$.emit` ，InputBox 组件就可以收到通知。

```ts
import React, { useRef, FC } from 'react'
import { useEventEmitter } from 'ahooks'
import { EventEmitter } from 'ahooks/lib/useEventEmitter'

const MessageBox: FC<{
  focus$: EventEmitter<void>
}> = function (props) {
  return (
    <div style={{ paddingBottom: 24 }}>
      <p>You received a message</p>
      <button
        type="button"
        onClick={() => {
          props.focus$.emit()
        }}
      >
        Reply
      </button>
    </div>
  )
}

const InputBox: FC<{
  focus$: EventEmitter<void>
}> = function (props) {
  const inputRef = useRef<any>()
  props.focus$.useSubscription(() => {
    inputRef.current.focus()
  })
  return <input ref={inputRef} placeholder="Enter reply" style={{ width: '100%', padding: '4px' }} />
}

export default function () {
  const focus$ = useEventEmitter()
  return (
    <>
      <MessageBox focus$={focus$} />
      <InputBox focus$={focus$} />
    </>
  )
}
```

### 使用场景

- 同级跨组件（距离较远的组件）通信

> 对于子组件通知父组件的情况，我们仍然推荐直接使用 props 传递一个 onEvent 函数。而对于父组件通知子组件的情况，可以使用 forwardRef 获取子组件的 ref ，再进行子组件的方法调用。 useEventEmitter 适合的是在距离较远的组件之间进行事件通知，或是在多个组件之间共享事件通知。

### 实现思路

通过发布订阅模式实现。主要实现两个方法`useSubscription`和`emit`

1. 首先要保证每次渲染调用 `useEventEmitter` 得到的返回值会保持不变： `useRef`来判断
2. `useSubscription` 会在组件创建时自动注册订阅，并在组件销毁时自动取消订阅：Set 数据结构记录订阅的事件列表，在`useEffect`里面实现监听和自动取消订阅操作
3. `emit` 方法，推送一个事件：循环 Set 事件列表取出时间执行

### 核心实现

主函数

```ts
function useEventEmitter<T = void>() {
  const ref = useRef<EventEmitter<T>>()
  if (!ref.current) {
    // 在组件多次渲染时，每次渲染调用 useEventEmitter 得到的返回值会保持不变，不会重复创建 EventEmitter 的实例。
    ref.current = new EventEmitter()
  }
  return ref.current
}
```

核心其实就是实现 `EventEmitter` 类

```ts
class EventEmitter<T> {
  // Set 结构存放订阅的事件列表
  private subscriptions = new Set<Subscription<T>>()

  // 发送一个事件通知来触发事件
  emit = (val: T) => {
    // 触发订阅列表的所有事件
    for (const subscription of this.subscriptions) {
      subscription(val)
    }
  }

  // 订阅事件
  useSubscription = (callback: Subscription<T>) => {
    // eslint-disable-next-line react-hooks/rules-of-hooks
    const callbackRef = useRef<Subscription<T>>()
    callbackRef.current = callback // 保证拿到最新引用
    // eslint-disable-next-line react-hooks/rules-of-hooks
    useEffect(() => {
      function subscription(val: T) {
        if (callbackRef.current) {
          callbackRef.current(val)
        }
      }
      // 添加到订阅事件列表
      this.subscriptions.add(subscription)
      return () => {
        // 卸载的时候自动删除（取消订阅）
        this.subscriptions.delete(subscription)
      }
    }, [])
  }
}
```

个人建议如果是单独父子通信的话，没必要使用这个 Hook，直接传递参数就行了；而大量的事件管理建议使用全局状态库管理

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useEventEmitter/index.ts)

## useLatest

返回当前最新值的 Hook，可以避免闭包问题。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-latest)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/uselatest-demo1/)

```ts
import React, { useState, useEffect } from 'react'
import { useLatest } from 'ahooks'

export default () => {
  const [count, setCount] = useState(0)

  const latestCountRef = useLatest(count)

  useEffect(() => {
    const interval = setInterval(() => {
      setCount(latestCountRef.current + 1)
    }, 1000)
    return () => clearInterval(interval)
  }, [])

  return (
    <>
      <p>count: {count}</p>
    </>
  )
}
```

### 核心实现

通过 `useRef`，保持最新的引用值

```ts
function useLatest<T>(value: T) {
  const ref = useRef(value)
  // useRef 保存能保证每次获取到的都是最新的值
  ref.current = value

  return ref
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useLatest/index.ts)

这个 Hook 还算比较实用点

## useMemoizedFn

持久化 function 的 Hook，理论上，可以使用 useMemoizedFn 完全代替 useCallback。

在某些场景中，我们需要使用 useCallback 来记住一个函数，但是在第二个参数 deps 变化时，会重新生成函数，导致函数地址变化。

使用 useMemoizedFn，可以省略第二个参数 deps，同时保证函数地址永远不会变化。
它的功能和 useCallback 类似，不过使用更简单，不需要提供 dep 数组。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-memoized-fn)

### 基本用法

useMemoizedFn 与 useCallback 可以实现同样的效果。但 useMemoizedFn 函数地址不会变化，可以用于性能优化。

示例中 memoizedFn 是不会变化的，callbackFn 在 count 变化时变化。

[官方在线 Demo](https://ahooks.js.org/~demos/usememoizedfn-demo2/)

```ts
import { useMemoizedFn } from 'ahooks'
import { message } from 'antd'
import React, { useCallback, useRef, useState } from 'react'

export default () => {
  const [count, setCount] = useState(0)

  const callbackFn = useCallback(() => {
    message.info(`Current count is ${count}`)
  }, [count])

  const memoizedFn = useMemoizedFn(() => {
    message.info(`Current count is ${count}`)
  })

  return (
    <>
      <p>count: {count}</p>
      <button
        type="button"
        onClick={() => {
          setCount((c) => c + 1)
        }}
      >
        Add Count
      </button>

      <p>You can click the button to see the number of sub-component renderings</p>

      <div style={{ marginTop: 32 }}>
        <h3>Component with useCallback function:</h3>
        {/* use callback function, ExpensiveTree component will re-render on state change */}
        <ExpensiveTree showCount={callbackFn} />
      </div>

      <div style={{ marginTop: 32 }}>
        <h3>Component with useMemoizedFn function:</h3>
        {/* use memoized function, ExpensiveTree component will only render once */}
        <ExpensiveTree showCount={memoizedFn} />
      </div>
    </>
  )
}

// some expensive component with React.memo
const ExpensiveTree = React.memo<{ [key: string]: any }>(({ showCount }) => {
  const renderCountRef = useRef(0)
  renderCountRef.current += 1

  return (
    <div>
      <p>Render Count: {renderCountRef.current}</p>
      <button type="button" onClick={showCount}>
        showParentCount
      </button>
    </div>
  )
})
```

### 使用场景

useMemoizedFn 能保证每次运行过程中保持最新的函数地址引用，适用于用于较为复杂的场景/组件，在属性传递过程减少不必要的 re-render

### 实现思路

针对上面 Demo 来说，如果我们要自己实现 `useMemoizedFn`，就是解决 useCallback 在 Demo 存在的缺陷。

1. callbackFn 的引用地址不能随 render 而改变，并在需要 count 值的时候能实时拿到（ref 保持引用地址不变）
2. 无需添加 dep 依赖（在 render 期间 ref 就要维持赋值最新的引用）

### 核心实现

1. 一个 `useRef` 保持外部传入的 fn 的引用 fnRef
2. 另一个 `useRef` 负责返回持久化函数 memoizedFn（内部获取并执行 fnRef），当实例后引用地址永远保持不变

源码不难，但确实是巧妙的实现。

```ts
/**
 * 持久化 function 的 Hook
 * @param fn 需要持久化的函数
 * @returns 引用地址永远不会变化的 fn
 */
function useMemoizedFn<T extends noop>(fn: T) {
  if (isDev) {
    if (!isFunction(fn)) {
      console.error(`useMemoizedFn expected parameter is a function, got ${typeof fn}`)
    }
  }
  // 通过 useRef 保证持有最新的引用
  const fnRef = useRef<T>(fn)

  // why not write `fnRef.current = fn`?
  // https://github.com/alibaba/hooks/issues/728
  // useMemo 在这里并不是核心，只是避免在 devtool 模式下的异常行为；
  fnRef.current = useMemo(() => fn, [fn]) // 在 render 期间实时拿到最新的fn，直观看就是：fnRef.current = fn

  const memoizedFn = useRef<PickFunction<T>>()
  if (!memoizedFn.current) {
    // 内部定义方法，赋值给 memoizedFn.current，这样返回出去的方法实例化后引用地址永远保持不变
    memoizedFn.current = function (this, ...args) {
      return fnRef.current.apply(this, args) // 调用的时候再去取 fnRef（存有最新的 fn 引用）
    }
  }

  // 返回的持久化函数
  return memoizedFn.current as T
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useMemoizedFn/index.ts)

## useReactive

提供一种数据响应式的操作体验，定义数据状态不需要写 useState，直接修改属性即可刷新视图。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-reactive)

### 基本用法

这种用法跟 Vue 一样，实现了响应式修改数据

[官方在线 Demo](https://ahooks.js.org/~demos/usereactive-demo1/)

```ts
import React from 'react'
import { useReactive } from 'ahooks'

export default () => {
  const state = useReactive({
    count: 0,
    inputVal: '',
    obj: {
      value: '',
    },
  })

  return (
    <div>
      <p> state.count：{state.count}</p>

      <button style={{ marginRight: 8 }} onClick={() => state.count++}>
        state.count++
      </button>
      <button onClick={() => state.count--}>state.count--</button>

      <p style={{ marginTop: 20 }}> state.inputVal: {state.inputVal}</p>
      <input onChange={(e) => (state.inputVal = e.target.value)} />

      <p style={{ marginTop: 20 }}> state.obj.value: {state.obj.value}</p>
      <input onChange={(e) => (state.obj.value = e.target.value)} />
    </div>
  )
}
```

### 核心实现

本质是使用 [Proxy](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy) 代理进行数据劫持和修改。

- [WeakMap](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)：WeakMap 对象是一组键/值对的集合，其中的键是弱引用的。其键必须是对象，而值可以是任意的。WeakMap 的作用就是可以更有效的垃圾回收、释放内存
- [Reflect.get](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect/get)：Reflect.get()方法与从对象 (target[key]) 中读取属性类似，但它是通过一个函数执行来操作的。

tip：由于 React 中更新页面状态值需要重新渲染，故 Proxy 代理改变/删除值后需要手动强制渲染更新

```ts
/**
 * target：需要取值的目标对象
 * propertyKey：需要获取的值的键值
 * receiver：如果target对象中指定了getter，receiver则为getter调用时的this值。
*/
Reflect.get(target, propertyKey[, receiver])
```

```ts
function useReactive<S extends Record<string, any>>(initialState: S): S {
  const update = useUpdate()
  const stateRef = useRef<S>(initialState)

  // useCreation 是 useMemo 或 useRef 的替代品。对于 useMemo 来说，useCreation能保证被 memo 的值一定不会被重计算
  const state = useCreation(() => {
    return observer(stateRef.current, () => {
      update() // 强制组件重新渲染
    })
  }, [])

  return state
}
```

核心是 `observer` 函数的实现：

```ts
function observer<T extends Record<string, any>>(initialVal: T, cb: () => void): T {
  const existingProxy = proxyMap.get(initialVal)

  // 添加缓存 防止重新构建proxy
  if (existingProxy) {
    return existingProxy
  }

  // 防止代理已经代理过的对象
  // https://github.com/alibaba/hooks/issues/839
  if (rawMap.has(initialVal)) {
    return initialVal
  }
  // 使用 Proxy 代理进行拦截和更新
  const proxy = new Proxy<T>(initialVal, {
    // 拦截对象的读取属性操作
    get(target, key, receiver) {
      const res = Reflect.get(target, key, receiver)
      // 如果值是对象，继续递归代理；否则直接返回属性值
      return isObject(res) ? observer(res, cb) : res
    },
    // 设置属性值操作的捕获器
    set(target, key, val) {
      const ret = Reflect.set(target, key, val)
      cb() // 属性赋值时触发回调
      return ret
    },
    // 拦截对对象属性的删除操作
    deleteProperty(target, key) {
      const ret = Reflect.deleteProperty(target, key)
      cb() // 删除属性时触发回调
      return ret
    },
  })

  proxyMap.set(initialVal, proxy)
  rawMap.set(proxy, initialVal)

  return proxy
}
```

结合上边代码，可以看到 `observer`函数的第二个参数回调函数的值是固定执行`update()`，故对属性进行`set`和`delete`操作都会重新渲染。

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useReactive/index.ts)

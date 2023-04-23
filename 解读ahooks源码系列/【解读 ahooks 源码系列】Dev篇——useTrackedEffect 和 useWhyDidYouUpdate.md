# 【解读 ahooks 源码系列】Dev 篇——useTrackedEffect 和 useWhyDidYouUpdate

## 前言

本文是 ahooks 源码（[v3.7.4](https://github.com/alibaba/hooks/tree/v3.7.4)）系列的第六篇——Dev 篇，该篇主要是协助开发调优的 Hook，只有两个

往期文章：

- [【解读 ahooks 源码系列】（开篇）如何获取和监听 DOM 元素](https://juejin.cn/post/7201889983170592824)：useEffectWithTarget
- [【解读 ahooks 源码系列】DOM 篇（一）](https://juejin.cn/post/7202254039215800375)：useEventListener、useClickAway、useDocumentVisibility、useDrop、useDrag
- [【解读 ahooks 源码系列】DOM 篇（二）](https://juejin.cn/post/7202633255043465271)：useEventTarget、useExternal、useTitle、useFavicon、useFullscreen、useHover
- [【解读 ahooks 源码系列】DOM 篇（三）](https://juejin.cn/post/7202996870251380791)：useMutationObserver、useInViewport、useKeyPress、useLongPress
- [【解读 ahooks 源码系列】DOM 篇（四）](https://juejin.cn/post/7203397626527891515)：useMouse、useResponsive、useScroll、useSize、useFocusWithin

本文主要解读 `useTrackedEffect`、`useWhyDidYouUpdate` 的源码实现

## useTrackedEffect

追踪是哪个依赖变化触发了 useEffect 的执行。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-tracked-effect)

### 基本用法

查看每次 effect 执行时发生变化的依赖项

[官方在线 Demo](https://ahooks.js.org/~demos/usetrackedeffect-demo1/)

```ts
import React, { useState } from 'react'
import { useTrackedEffect } from 'ahooks'

export default () => {
  const [count, setCount] = useState(0)
  const [count2, setCount2] = useState(0)

  useTrackedEffect(
    (changes) => {
      console.log('Index of changed dependencies: ', changes)
    },
    [count, count2]
  )

  return (
    <div>
      <p>Please open the browser console to view the output!</p>
      <div>
        <p>Count: {count}</p>
        <button onClick={() => setCount((c) => c + 1)}>count + 1</button>
      </div>
      <div style={{ marginTop: 16 }}>
        <p>Count2: {count2}</p>
        <button onClick={() => setCount2((c) => c + 1)}>count + 1</button>
      </div>
    </div>
  )
}
```

### 核心实现

实现原理：通过 uesRef 记录上一次依赖的值，在当前执行的时候，判断当前依赖值和上次依赖值之间有无变化

- changes：变化的依赖 index 数组
- previousDeps：上一个依赖
- currentDeps：当前依赖

```ts
useTrackedEffect(
  effect: (changes: [], previousDeps: [], currentDeps: []) => (void | (() => void | undefined)),
  deps?: deps,
)
```

源码实现

```ts
const useTrackedEffect = (effect: Effect, deps?: DependencyList) => {
  const previousDepsRef = useRef<DependencyList>() // 记录上次依赖

  useEffect(() => {
    // 判断依赖前后的 changes
    const changes = diffTwoDeps(previousDepsRef.current, deps)
    const previousDeps = previousDepsRef.current // 赋值上次依赖
    previousDepsRef.current = deps
    return effect(changes, previousDeps, deps)
  }, deps)
}
```

diffTwoDeps 方法实现：

1. 对前后两个 deps 依赖项列表使用 Object.is 进行严格相等性检查
2. 如果定义了 deps1，则遍历 deps1 并将每个元素与来自 deps2 的对应索引元素进行比较（因为这个函数只在这个钩子中使用，所以假设两个 deps 列表的长度总是相同的）
   - 相等返回 -1
   - 不相等返回索引值
   - 过滤小于 0 的值（即校验结果相等的）`.filter((ele) => ele >= 0)`，最终只返回变化的数组索引值

```ts
const diffTwoDeps = (deps1?: DependencyList, deps2?: DependencyList) => {
  // 对前后两个 deps 依赖项列表使用 Object.is 进行严格相等性检查
  return deps1
    ? deps1.map((_ele, idx) => (!Object.is(deps1[idx], deps2?.[idx]) ? idx : -1)).filter((ele) => ele >= 0) // 过滤相等值
    : deps2
    ? deps2.map((_ele, idx) => idx)
    : []
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useTrackedEffect/index.ts)

## useWhyDidYouUpdate

帮助开发者排查是那个属性改变导致了组件的 rerender。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-why-did-you-update)

### 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usewhydidyouupdate-demo1/)

打开控制台，可以看到改变的属性。

```ts
import { useWhyDidYouUpdate } from 'ahooks'
import React, { useState } from 'react'

const Demo: React.FC<{ count: number }> = (props) => {
  const [randomNum, setRandomNum] = useState(Math.random())

  useWhyDidYouUpdate('useWhyDidYouUpdateComponent', { ...props, randomNum })

  return (
    <div>
      <div>
        <span>number: {props.count}</span>
      </div>
      <div>
        randomNum: {randomNum}
        <button onClick={() => setRandomNum(Math.random)} style={{ marginLeft: 8 }}>
          🎲
        </button>
      </div>
    </div>
  )
}

export default () => {
  const [count, setCount] = useState(0)

  return (
    <div>
      <Demo count={count} />
      <div>
        <button onClick={() => setCount((prevCount) => prevCount - 1)}>count -</button>
        <button onClick={() => setCount((prevCount) => prevCount + 1)} style={{ marginLeft: 8 }}>
          count +
        </button>
      </div>
      <p style={{ marginTop: 8 }}>Please open the browser console to view the output!</p>
    </div>
  )
}
```

### 使用场景

- 检查哪些 props 发生改变
- 协助找出无效渲染：`useWhyDidYouUpdate` 会告诉我们监听数据中所有变化的数据，不管它是不是无效的更新，但还需要我们自己来区分识别哪些是无效更新的属性，从而进行优化。

### 实现思路

1. 使用 useRef 声明 prevProps 变量（确保拿到最新值），用来保存上一次的 props
2. 每次 useEffect 更新都置空 changedProps 对象，并将新旧 props 对象的属性提取出来，生成属性数组 allKeys
3. 遍历 allKeys 数组，去对比新旧属性值。如果不同，则记录到 changedProps 对象中
4. 如果 changedProps 有长度，则输出改变的内容，并更新 prevProps

### 核心实现

实现原理：通过 useEffect 拿到上一次 props 值 和当前 props 值 进行遍历比较，如果值发送改变则输出

```ts
// componentName：观测组件的名称
// props：需要观测的数据（当前组件 state 或者传入的 props 等可能导致 rerender 的数据）
export default function useWhyDidYouUpdate(componentName: string, props: IProps) {
  const prevProps = useRef<IProps>({})

  useEffect(() => {
    if (prevProps.current) {
      // 获取所有的需要观测的数据
      const allKeys = Object.keys({ ...prevProps.current, ...props })
      const changedProps: IProps = {} // 发生改变的属性值

      allKeys.forEach((key) => {
        // 通过 Object.is 判断是否进行更新
        if (!Object.is(prevProps.current[key], props[key])) {
          changedProps[key] = {
            from: prevProps.current[key],
            to: props[key],
          }
        }
      })

      // 遍历改变的属性，有值则输出日志
      if (Object.keys(changedProps).length) {
        console.log('[why-did-you-update]', componentName, changedProps)
      }
    }

    prevProps.current = props
  })
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useWhyDidYouUpdate/index.ts)

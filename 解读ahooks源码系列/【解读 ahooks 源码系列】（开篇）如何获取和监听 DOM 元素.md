# 【解读 ahooks 源码系列】（开篇）如何获取和监听 DOM 元素

## 前言

由于在工作中自定义 Hook 场景写的较多，当实现某个通用场景功能时，可能没想过有已实现好的 Hook 封装或者压根没想去从 Hooks 库里面找，但是社区好的实现使用起来是可以提高开发效率和减少 bug 率的。

公司项目中有依赖库 [ahooks](https://ahooks.js.org/zh-CN/hooks/use-request/index)，但我用的次数不多，于是有了想详细了解 ahooks 的打算，更主要是为了更加熟练抽离与实现一些场景 Hook，学习如何更好的自定义 Hook，便有开始阅读 ahooks 源码的打算了。

## 学习 ahooks 源码的好处

在我看来，学习 ahooks 常见 Hooks 封装有以下好处：

- 熟悉如何根据需求去提炼相应的 Hooks，将通用逻辑进行封装
- 讲解源码实现思路，提炼核心实现，通过学习源码学习自定义 Hooks 最佳实践
- 深入学习特定的场景 Hooks，项目开发中一点通，使用时更得心应手

## 关于源码系列

本系列文章基于 ahooks 版本 [v3.7.4](https://github.com/alibaba/hooks/tree/v3.7.4)，后续会相继输出 ahooks 源码解读的系列文章。

按照 ahooks 官网的分类，我目前先从 DOM 篇开始看起，DOM 篇包括的 Hooks 如下：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e1b098277ba845c4b82636847d39aa47~tplv-k3u1fbpfcp-watermark.image?)

- useEventListener：优雅的使用 addEventListener。
- useClickAway：监听目标元素外的点击事件。
- useDocumentVisibility：优雅的使用 addEventListener。
- useDrop & useDrag：处理元素拖拽的 Hook。
- useEventTarget：常见表单控件(通过 e.target.value 获取表单值) 的 onChange 跟 value 逻辑封装，支持自定义值转换和重置功能。
- useExternal：动态注入 JS 或 CSS 资源，useExternal 可以保证资源全局唯一。
- useTitle：用于设置页面标题。
- useFavicon：设置页面的 favicon。
- useFullscreen：管理 DOM 全屏的 Hook。
- useHover：监听 DOM 元素是否有鼠标悬停。
- useMutationObserver：一个监听指定的 DOM 树发生变化的 Hook。
- useInViewport：观察元素是否在可见区域，以及元素可见比例。
- useKeyPress：监听键盘按键，支持组合键，支持按键别名。
- useLongPress：监听目标元素的长按事件。
- useMouse：监听鼠标位置。
- useResponsive：获取响应式信息。
- useScroll：监听元素的滚动位置。
- useSize：监听 DOM 节点尺寸变化的 Hook。
- useFocusWithin：监听当前焦点是否在某个区域之内，同 css 属性: focus-within。

由于内容较多，DOM 篇会分成几篇文章输出，这样每篇读起来既不太耗时也能快速过一遍。文章会在解读源码的基础上，也会把涉及到的 JS 基础知识拎出来，在学源码的过程也能查漏补缺基础。

回到本文正题，在看 DOM 篇分类下的 Hooks 时，我发现 `getTargetElement` 方法和 `useEffectWithTarget` 内部 Hook 使用较多，所以在讲源码之前先来了解这两个 Hook。

## 如何获取 DOM 元素

### 三种类型的 target

在 [DOM 类 Hooks 使用规范](https://ahooks.js.org/zh-CN/guide/dom/)中提到：

ahooks 大部分 DOM 类 Hooks 都会接收 target 参数，表示要处理的元素。

target 支持三种类型 `React.MutableRefObject`、`HTMLElement`、`() => HTMLElement`。

1. React.MutableRefObject

```js
export default () => {
  const ref = useRef(null)
  const isHovering = useHover(ref)
  return <div ref={ref}>{isHovering ? 'hover' : 'leaveHover'}</div>
}
```

2. HTMLElement

```js
export default () => {
  const isHovering = useHover(document.getElementById('test'))
  return <div id="test">{isHovering ? 'hover' : 'leaveHover'}</div>
}
```

3. 支持 () => HTMLElement，一般适用在 SSR 场景

```js
export default () => {
  const isHovering = useHover(() => document.getElementById('test'))
  return <div id="test">{isHovering ? 'hover' : 'leaveHover'}</div>
}
```

### getTargetElement

为了兼容以上三种类型入参，ahooks 封装了 [getTargetElement - 获取目标 DOM 元素](https://github.com/alibaba/hooks/blob/master/packages/hooks/src/utils/domTarget.ts) 方法。我们来看看代码做了什么：

1. 判断是否为浏览器环境，不是则返回 `undefined`
2. 判断目标元素是否为空，为空则返回函数参数指定的默认元素
3. 核心：
   - 如果是函数，则返回函数执行后的结果
   - 如果有 `current` 属性，则返回 `.current`属性的值，兼容 `React.MutableRefObject` 类型
   - 以上都不是，则代表普通 DOM 元素，直接返回

```js
export function getTargetElement<T extends TargetType>(target: BasicTarget<T>, defaultElement?: T) {
  // 判断是否为浏览器环境
  if (!isBrowser) {
    return undefined;
  }

  // 目标元素为空则返回函数参数指定的默认元素
  if (!target) {
    return defaultElement;
  }

  let targetElement: TargetValue<T>;

  // 支持函数执行返回
  if (isFunction(target)) {
    targetElement = target();
  } else if ('current' in target) {
    // 兼容 React.MutableRefObject 类型，返回 .current 属性的值
    targetElement = target.current;
  } else {
    // 普通 DOM 元素
    targetElement = target;
  }

  return targetElement;
}
```

对应的 TS 类型：

```ts
type TargetValue<T> = T | undefined | null

type TargetType = HTMLElement | Element | Window | Document

export type BasicTarget<T extends TargetType = Element> =
  | (() => TargetValue<T>)
  | TargetValue<T>
  | MutableRefObject<TargetValue<T>>
```

## 监听 DOM 元素

### target 支持动态变化

ahooks 的 [DOM 类 Hooks 使用规范](https://ahooks.js.org/zh-CN/guide/dom/)第二条点指出：

DOM 类 Hooks 的 target 是支持动态变化的，如下：

```js
export default () => {
  const [boolean, { toggle }] = useBoolean()

  const ref = useRef(null)
  const ref2 = useRef(null)

  const isHovering = useHover(boolean ? ref : ref2)
  return (
    <>
      <div ref={ref}>{isHovering ? 'hover' : 'leaveHover'}</div>
      <div ref={ref2}>{isHovering ? 'hover' : 'leaveHover'}</div>
    </>
  )
}
```

### useEffectWithTarget

为了满足上述条件， ahooks 内部则封装 `useEffectWithTarget`（`packages/hooks/src/utils/useEffectWithTarget.ts`），来看这个文件的代码：

```js
import { useEffect } from 'react'
import createEffectWithTarget from './createEffectWithTarget'

const useEffectWithTarget = createEffectWithTarget(useEffect)

export default useEffectWithTarget
```

看到它实际用了 `createEffectWithTarget`方法，传入的参数是 `useEffect`（`packages/hooks/src/utils/createEffectWithTarget.ts`）

> - createEffectWithTarget 接受参数 useEffect 或 useLayoutEffect，返回 useEffectWithTarget 函数
> - useEffectWithTarget 函数接收三个参数：前两个参数是 effect 和 deps（与 useEffect 参数一致），第三个参数则兼容了 DOM 元素的三种类型，可传 普通 DOM/ref 类型/函数类型

useEffectWithTarget 实现思路：

1. 使用 useEffect/useLayoutEffect 监听，内部不传第二个参数依赖项，每次更新都会执行该副作用函数
2. 通过 hasInitRef 判断是否是第一次执行，是则初始化：记录最后一次目标元素列表和依赖项，执行 effect 函数
3. 由于该 useEffectType 函数体每次更新都会执行，所以每次都拿到最新的 targets 和 deps，所以后续执行可与第 2 点记录的最后一次的 ref 值进行比对
4. 非首次执行：则判断元素列表长度或目标元素或者依赖发生变化，变化了则执行更新流程：执行上一次返回的卸载函数，更新最新值，重新执行 effect
5. 组件卸载：执行 unLoadRef.current?.() 卸载函数，重置 hasInitRef

```ts
const createEffectWithTarget = (useEffectType: typeof useEffect | typeof useLayoutEffect) => {
  /**
   *
   * @param effect
   * @param deps
   * @param target target should compare ref.current vs ref.current, dom vs dom, ()=>dom vs ()=>dom
   */
  const useEffectWithTarget = (
    effect: EffectCallback,
    deps: DependencyList,
    target: BasicTarget<any> | BasicTarget<any>[]
  ) => {
    // 判断是否已初始化
    const hasInitRef = useRef(false)

    const lastElementRef = useRef<(Element | null)[]>([]) // 最后一次
    const lastDepsRef = useRef<DependencyList>([])

    const unLoadRef = useRef<any>()

    // useEffectType：代表 useEffect 或 useLayoutEffect，每次更新都会执行该函数
    useEffectType(() => {
      const targets = Array.isArray(target) ? target : [target]
      const els = targets.map((item) => getTargetElement(item)) // 获取 DOM 元素列表

      // 首次执行：初始化
      if (!hasInitRef.current) {
        hasInitRef.current = true
        lastElementRef.current = els // 最后一次执行的相应的 target 元素
        lastDepsRef.current = deps // 最后一次执行的相应的依赖

        unLoadRef.current = effect() // 执行外部传入的 effect 函数，返回卸载函数
        return
      }

      // 非首次执行：判断元素列表长度或目标元素或者依赖发生变化
      if (
        els.length !== lastElementRef.current.length ||
        !depsAreSame(els, lastElementRef.current) ||
        !depsAreSame(deps, lastDepsRef.current)
      ) {
        // 依赖发生变更了，相当于走 useEffect 更新流程
        unLoadRef.current?.()
        lastElementRef.current = els
        lastDepsRef.current = deps
        unLoadRef.current = effect() // 再次执行 effect，赋值卸载函数给 unLoadRef
      }
    }) // 没有传第二个参数，则每次都会执行

    // 卸载操作 Hook
    useUnmount(() => {
      unLoadRef.current?.() // 执行卸载操作
      // for react-refresh
      hasInitRef.current = false
    })
  }

  return useEffectWithTarget
}
```

depsAreSame 实现：

```ts
import type { DependencyList } from 'react'

export default function depsAreSame(oldDeps: DependencyList, deps: DependencyList): boolean {
  if (oldDeps === deps) return true // 浅比较
  for (let i = 0; i < oldDeps.length; i++) {
    if (!Object.is(oldDeps[i], deps[i])) return false
  }
  return true
}
```

这样使用起来跟 useEffect 的区别就是有第三个参数——监听的 DOM 元素

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2278689ef1164697ad931b6342be7921~tplv-k3u1fbpfcp-watermark.image?)

## 参考文章

- [ahooks 是怎么处理 DOM 的？](https://juejin.cn/post/7111860051362447390)

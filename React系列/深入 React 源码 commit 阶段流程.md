---
theme: channing-cyan
---

## 前言

本文源码基于 React v17.0.2，阅读本文前需要先了解 React 渲染相关流程原理与重点概念。前面已经写了 React render 阶段的两篇文章：

- [深入 React 源码 render 阶段的 beginWork 流程](https://juejin.cn/post/7218961155900555321)
- [深入 React 源码 render 阶段的 completeWork 流程](https://juejin.cn/post/7221186557729374264)

React 渲染流程可分为 render 阶段和 commit 阶段。

- render 阶段就是 Reconciler（协调器） 工作的阶段，主要构建 Fiber 树和生成 effectList
- commit 阶段就是 Renderer（渲染器）工作的阶段，这个阶段会把 render 阶段计算出的变更应用到 DOM 上

## commit 阶段流程

### commit 阶段概览

```js
function commitRoot(root) {
  const renderPriorityLevel = getCurrentPriorityLevel()
  // 调度 commitRootImpl
  runWithPriority(ImmediateSchedulerPriority, commitRootImpl.bind(null, root, renderPriorityLevel))
  return null
}
```

commit 阶段首先会处理上次还未被完成的 effect 异步函数，其次针对 root 上收集的 effectList 进行处理，其余工作可分为三个子阶段：

1. Before Mutation 阶段（执行 DOM 操作前）
2. Mutation 阶段（执行 DOM 操作）
3. Layout 阶段（执行 DOM 操作后）

```js
function commitRootImpl(root, renderPriorityLevel) {
  do {
    // 执行未执行的 useEffect
    flushPassiveEffects() // 底层会调用 `flushPassiveEffectsImpl` 遍历执行 useEffect 的回调和销毁函数
  } while (rootWithPendingPassiveEffects !== null) // 判断有无未执行的 useEffect，有的话先执行

  // ...

  // 获取 effectList
  let firstEffect
  if (finishedWork.flags > PerformedWork) {
    // 把 rootFiber 节点 加入到链表中
    if (finishedWork.lastEffect !== null) {
      finishedWork.lastEffect.nextEffect = finishedWork
      firstEffect = finishedWork.firstEffect
    } else {
      firstEffect = finishedWork
    }
  } else {
    // There is no effect on the root.
    firstEffect = finishedWork.firstEffect
  }

  // ... 省略其它代码，下文说

  // Before Mutation 阶段
  commitBeforeMutationEffects()
  // Mutation 阶段
  commitMutationEffects(root, renderPriorityLevel)
  // Layout 阶段
  commitLayoutEffects(root, lanes)
}
```

`commitRootImpl` 函数的代码很长，上面省略为大致流程，该函数最主要做的事情：

1. 针对 rootFiber 节点进行处理，将 rootFiber 节点加入到 effectList 中
2. 调用 `commitBeforeMutationEffects`函数（Before Mutation 阶段）
3. 调用 `commitMutationEffects` 函数（Mutation 阶段）
4. 调用 `commitLayoutEffects` 函数（Layout 阶段）

### Before Mutation 阶段

Before Mutation 阶段的入口函数为 `commitBeforeMutationEffects`，主要作用是：

1. 如果是类组件，判断调用 getSnapshotBeforeUpdate，在 commit 前获取 DOM 相关信息
2. 如果是函数组件且使用了 useEffect，则异步调度 useEffect（安排回调）

```js
function commitBeforeMutationEffects() {
  while (nextEffect !== null) {
    const current = nextEffect.alternate
    // 处理 blur 与 focus 逻辑
    if (!shouldFireAfterActiveInstanceBlur && focusedInstanceHandle !== null) {
      // ...
    }

    const flags = nextEffect.flags
    if ((flags & Snapshot) !== NoFlags) {
      // 在该函数里会调用 getSnapshotBeforeUpdate
      commitBeforeMutationEffectOnFiber(current, nextEffect)
    }
    if ((flags & Passive) !== NoFlags) {
      // 异步调度 useEffect
      if (!rootDoesHavePassiveEffects) {
        rootDoesHavePassiveEffects = true
        scheduleCallback(NormalSchedulerPriority, () => {
          flushPassiveEffects()
          return null
        })
      }
    }
    nextEffect = nextEffect.nextEffect
  }
}
```

`commitBeforeMutationEffectOnFiber` 函数：里面会根据不同的 `finishedWork.tag` 进行处理，如果是 `ClassComponent`，则会执行 getSnapshotBeforeUpdate 生命周期，并将返回值赋值给 fiber 对象的 `__reactInternalSnapshotBeforeUpdate` 属性

```js
function commitBeforeMutationEffectOnFiber(current, finishedWork) {
  switch (finishedWork.tag) {
    // ...
    case ClassComponent: {
      if (finishedWork.flags & Snapshot) {
        if (current !== null) {
          const prevProps = current.memoizedProps
          const prevState = current.memoizedState
          const instance = finishedWork.stateNode
          // 执行 getSnapshotBeforeUpdate 生命周期
          const snapshot = instance.getSnapshotBeforeUpdate(
            finishedWork.elementType === finishedWork.type
              ? prevProps
              : resolveDefaultProps(finishedWork.type, prevProps),
            prevState
          )
          instance.__reactInternalSnapshotBeforeUpdate = snapshot
        }
      }
      return
    }
  }
}
```

### Mutation 阶段

Mutation 阶段的入口函数为 `commitMutationEffects`，主要作用是：

- 遍历 effectList，根据 effectTag 分别处理操作 DOM 节点（如 `Placement | Update | Deletion | Hydrating`）

```js
function commitMutationEffects(root, renderPriorityLevel) {
  // 遍历 effectList
  while (nextEffect !== null) {
    const flags = nextEffect.flags

    // ...

    const primaryFlags = flags & (Placement | Update | Deletion | Hydrating)
    switch (primaryFlags) {
      // 插入 DOM
      case Placement: {
        commitPlacement(nextEffect)
        nextEffect.flags &= ~Placement
        break
      }
      // 更新组件和 DOM
      case PlacementAndUpdate: {
        // 将变化的 DOM 都插入到页面上
        commitPlacement(nextEffect)
        nextEffect.flags &= ~Placement

        // 更新
        const current = nextEffect.alternate
        commitWork(current, nextEffect)
        break
      }
      // ...
      // 更新组件
      case Update: {
        const current = nextEffect.alternate
        commitWork(current, nextEffect)
        break
      }
      // 卸载
      case Deletion: {
        commitDeletion(root, nextEffect, renderPriorityLevel)
        break
      }
    }

    nextEffect = nextEffect.nextEffect
  }
}
```

**Placement（插入）**

调用了 `commitPlacement` 方法：获取当前 Fiber 节点的父 Fiber 对应的 DOM 节点和插入位置，根据父节点对应的 DOM 是否为 container，执行 `insertOrAppendPlacementNodeIntoContainer` 或 `insertOrAppendPlacementNode` 进行插入操作

```js
function commitPlacement(finishedWork) {
  // 找到最近的 Fiber 节点
  const parentFiber = getHostParentFiber(finishedWork)
  // 获取父级 DOM 节点
  const parentStateNode = parentFiber.stateNode
  // 获取兄弟 DOM 节点
  const before = getHostSibling(finishedWork)
  // 是否 container
  if (isContainer) {
    insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent)
  } else {
    insertOrAppendPlacementNode(finishedWork, before, parent)
  }
}
```

**Update（更新）**

调用 `commitWork`，处理由 useLayoutEffect 创建的 effect

```js
function commitWork(current, finishedWork) {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case MemoComponent:
    case SimpleMemoComponent:
    case Block: {
      // 调用 useLayoutEffect 的销毁函数
      commitHookEffectListUnmount(HookLayout | HookHasEffect, finishedWork)
      return
    }
  }
}

function commitHookEffectListUnmount(tag, finishedWork) {
  const updateQueue = finishedWork.updateQueue
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next
    let effect = firstEffect
    do {
      if ((effect.tag & tag) === tag) {
        // 如果有定义 effect 的销毁函数，则执行
        const destroy = effect.destroy
        effect.destroy = undefined
        if (destroy !== undefined) {
          destroy()
        }
      }
      effect = effect.next
    } while (effect !== firstEffect)
  }
}
```

`HookLayout | HookHasEffect` 是一个位运算表达式，表示同时具备两种标识，即表示是 useLayoutEffect 类型的 effect

**Deletion（删除）**

调用 `commitDeletion` 方法，当 fiber.tag 为 ClassComponent 时，执行 componentWillUnmount 生命周期钩子，从页面移除 Fiber 节对应 DOM 节点

```js
function commitDeletion(finishedRoot, current, renderPriorityLevel) {
  if (supportsMutation) {
    unmountHostComponents(finishedRoot, current, renderPriorityLevel)
  } else {
    // Detach refs and call componentWillUnmount() on the whole subtree.
    commitNestedUnmounts(finishedRoot, current, renderPriorityLevel)
  }
  const alternate = current.alternate
  detachFiberMutation(current)
  if (alternate !== null) {
    detachFiberMutation(alternate)
  }
}
```

### Layout 阶段

Layout 阶段的入口函数为 `commitLayoutEffects`，作用：

- 针对类组件：调用生命周期 componentDidMount 和 componentDidUpdate，执行 setState 的第二个参数回调函数
- 针对函数组件：执行 useLayoutEffect 的回调函数

```js
function commitLayoutEffects(root, committedLanes) {
  // 遍历 effectList
  while (nextEffect !== null) {
    const flags = nextEffect.flags
    // 调用生命周期钩子和 hook
    if (flags & (Update | Callback)) {
      const current = nextEffect.alternate
      commitLayoutEffectOnFiber(root, current, nextEffect, committedLanes)
    }

    // 赋值 ref
    if (flags & Ref) {
      commitAttachRef(nextEffect)
    }

    nextEffect = nextEffect.nextEffect
  }
}
```

```js
function commitLayoutEffectOnFiber(finishedRoot, current, finishedWork, committedLanes) {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent:
    case Block: {
      // 循环 FunctionComponent 上的 effect 链

      // 执行 useLayoutEffect 的回调
      commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork)
      // 收集 useEffect 需要执行回调和销毁函数
      schedulePassiveEffects(finishedWork)
      return
    }
    case ClassComponent: {
      const instance = finishedWork.stateNode
      if (finishedWork.flags & Update) {
        if (current === null) {
          // 节点首次挂载，执行 componentDidMount
          instance.componentDidMount()
        } else {
          const prevProps =
            finishedWork.elementType === finishedWork.type
              ? current.memoizedProps
              : resolveDefaultProps(finishedWork.type, current.memoizedProps)
          const prevState = current.memoizedState
          // 更新
          instance.componentDidUpdate(prevProps, prevState, instance.__reactInternalSnapshotBeforeUpdate)
        }
      }

      const updateQueue = finishedWork.updateQueue
      if (updateQueue !== null) {
        // 执行 setState 的第二个参数回调
        commitUpdateQueue(finishedWork, updateQueue, instance)
      }
      return
    }
    // case ...
  }
}
```

针对函数组件，会调用 `commitHookEffectListMount` 和 `schedulePassiveEffects` 方法

```js
// 只处理由 useLayoutEffect 创建的 effect
function commitHookEffectListMount(tag, finishedWork) {
  const updateQueue = finishedWork.updateQueue
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next
    let effect = firstEffect
    do {
      if ((effect.tag & tag) === tag) {
        // 执行副作用 effect 创建函数，并将返回值（cleanup 函数）存储到 `destroy` 属性上
        // 在 Mutation 阶段已经执行了 useLayoutEffect 的销毁函数，在此执行创建函数，确保每次 useLayoutEffect 的执行的卸载与创建顺序正确
        const create = effect.create
        effect.destroy = create()
      }
      effect = effect.next
    } while (effect !== firstEffect)
  }
}

// 收集 useEffect 待执行的回调和销毁函数
function schedulePassiveEffects(finishedWork) {
  const updateQueue = finishedWork.updateQueue
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next
    let effect = firstEffect
    do {
      const { next, tag } = effect
      if (
        // HookPassive 标识 useEffect
        (tag & HookPassive) !== NoHookEffect &&
        // 依赖数组没有发生变化
        (tag & HookHasEffect) !== NoHookEffect
      ) {
        // 收集待执行的销毁函数到数组
        enqueuePendingPassiveHookEffectUnmount(finishedWork, effect)
        // 收集待执行的回调函数到数组
        enqueuePendingPassiveHookEffectMount(finishedWork, effect)
      }
      effect = next
    } while (effect !== firstEffect)
  }
}
```

## 总结

commit 阶段可分为三个子阶段：

1. Before Mutation 阶段：读取组件变更前的状态。其中针对类组件会调用 getSnapshotBeforeUpdate 生命周期函数；调度 useEffect
2. Mutation 阶段：更新 DOM 节点的阶段。如插入、更新以及删除操作。针对类组件会调用 componentWillUnmount 生命周期函数；针对函数组件，则会执行 useLayoutEffect 的销毁函数
3. Layout 阶段：调用类组件的 componentDidMount、componentDidUpdate 生命周期函数，执行 setState 的第二个回调参数；针对函数组件会执行 useLayoutEffect 的 effect 回调，并处理 useLayoutEffect

以上就是各个 commit 阶段的大致工作，当然描述的内容肯定不全，只是挑出一些重点，具体各位可以细看源码。

## 参考

- [React 技术揭秘](https://react.iamkasong.com/renderer/prepare.html#before-mutation%E4%B9%8B%E5%89%8D)

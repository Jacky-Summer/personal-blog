# 深入 React 源码 render 阶段的 beginWork 流程

## 前言

本文源码基于 Reactv17.0.2，阅读本文前需要先有了解 React 渲染相关流程与重点概念。

## React 渲染的两个阶段

React 渲染流程可分为 render 阶段和 commit 阶段。

### render 阶段

render 阶段就是 Reconciler（协调器） 工作的阶段，主要构建 Fiber 树和生成 EffectList，在该阶段会调用组件的 render 方法，收集了需要应用到 DOM 上的变更

### commit 阶段

commit 阶段就是 Renderer（渲染器）工作的阶段，这个阶段会把 render 阶段计算出的变更应用到 DOM 上，该阶段又可分为三个子阶段：

1. before mutation 阶段（执行 DOM 操作前）
2. mutation 阶段（执行 DOM 操作）
3. layout 阶段（执行 DOM 操作后）

## React 初始化

在讲 beginWork 方法之前，让我们跟随源码过下 React 初始化阶段做的事情

### legacyRenderSubtreeIntoContainer

在使用 Reactv17 及以下版本初始化的时候，我们会使用 render 方法将页面挂载到根节点：

```js
import { render } from 'react-dom'

render(<div>页面组件</div>, document.getElementById('root'))
```

通过源码可以看出 `ReactDOM.render` 实际上返回了函数 `legacyRenderSubtreeIntoContainer` 的调用结果，主要作用是初始化容器。

> packages/react-dom/src/client/ReactDOMLegacy.js

```js
export function render(element, container, callback) {
  return legacyRenderSubtreeIntoContainer(null, element, container, false, callback)
}

function legacyRenderSubtreeIntoContainer(parentComponent, children, container, forceHydrate, callback) {
  let root = container._reactRootContainer
  let fiberRoot
  // 没有根组件则需创建
  if (!root) {
    // 初始化容器并创建 FiberRoot
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(container, forceHydrate)
    fiberRoot = root._internalRoot
    // ...
    // 初始化不应该批量更新
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback)
    })
  } else {
    fiberRoot = root._internalRoot
    // ...
    // 将传入的组件进行调度更新，更新 container
    updateContainer(children, fiberRoot, parentComponent, callback)
  }
  return getPublicRootInstance(fiberRoot) // 获取 root 实例
}
```

`legacyRenderSubtreeIntoContainer` 函数主要做的事情是初始化容器

接下来看看内部执行的 `updateContainer` 方法做的事情：

1. 获取当前 Fiber 节点的 lane（任务优先级）
2. 根据优先级创建当前 Fiber 节点的 update 对象，并将其加入更新队列
3. 调度更新任务

### scheduleUpdateOnFiber

```js
function updateContainer(element, container, parentComponent, callback) {
  const current = container.current // 获取当前时间戳
  const eventTime = requestEventTime() // 获取更新触发时间
  const lane = requestUpdateLane(current) // 获取任务优先级

  if (enableSchedulingProfiler) {
    markRenderScheduled(lane)
  }

  const context = getContextForSubtree(parentComponent)
  if (container.context === null) {
    container.context = context
  } else {
    container.pendingContext = context
  }

  // 创建 update 对象
  const update = createUpdate(eventTime, lane)

  callback = callback === undefined ? null : callback
  if (callback !== null) {
    update.callback = callback
  }
  // 将 update 加入 updateQueue
  enqueueUpdate(current, update)
  // 调度更新任务
  scheduleUpdateOnFiber(current, lane, eventTime)

  return lane
}
```

上面代码中，比较重要的是 `scheduleUpdateOnFiber` 方法，它是整个更新任务的开始，也是 render 阶段的入口，通过该方法来调度任务；具体做的事情：

1. 通过当前的更新优先级 lane，把当前 fiber 到 rootFiber 的父级链表上的所有优先级都给更新了。
2. 如果当前 fiber 确定更新，则调用 `ensureRootIsScheduled`。（无论是首次渲染还是后续更新，legacy 下模式最终都会执行 `performSyncWorkOnRoot`，下文会讲到）

> packages/react-reconciler/src/ReactFiberWorkLoop.new.js

```js
function scheduleUpdateOnFiber(fiber, lane, eventTime) {
  // 从触发更新的节点开始向上遍历到 rootFiber，遍历的过程会更新父级链表上所有 Fiber 的任务优先级
  const root = markUpdateLaneFromFiberToRoot(fiber, lane)
  if (root === null) {
    return null
  }

  // Mark that the root has a pending update.
  markRootUpdated(root, lane, eventTime)

  if (
    // Check if we're inside unbatchedUpdates
    (executionContext & LegacyUnbatchedContext) !== NoContext &&
    // Check if we're not already rendering
    (executionContext & (RenderContext | CommitContext)) === NoContext
  ) {
    // 首次渲染
    performSyncWorkOnRoot(root)
  } else {
    // 根据任务的类型安排调度任务
    ensureRootIsScheduled(root, eventTime)
    if (executionContext === NoContext) {
      flushSyncCallbackQueue()
    }
  }
}

// 向上递归标记父级链上的优先级 childLanes
function markUpdateLaneFromFiberToRoot(sourceFiber, lane) {
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane) // 更新当前 Fiber
  let alternate = sourceFiber.alternate
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane) // 更新缓冲树上的 lane
  }

  let node = sourceFiber // 当前 fiber
  let parent = sourceFiber.return // 当前 fiber 的父级
  // 向上递归父级链上的 childLanes
  while (parent !== null) {
    parent.childLanes = mergeLanes(parent.childLanes, lane)
    alternate = parent.alternate
    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane)
    }
    node = parent
    parent = parent.return
  }
  if (node.tag === HostRoot) {
    const root = node.stateNode
    return root
  } else {
    return null
  }
}
```

`markUpdateLaneFromFiberToRoot` 会更新当前 fiber 的优先级 lane, 同时向上递归父级链上的任务优先级 childLanes，在更新过程中确保更新按照正确的顺序和优先级执行。

`ensureRootIsScheduled` 的调用链路（本文只关注同步模式）：

```js
function ensureRootIsScheduled(root, currentTime) {
  // ...
  let newCallbackNode
  // 同步模式
  if (newCallbackPriority === SyncLanePriority) {
    newCallbackNode = scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root))
  } else if (newCallbackPriority === SyncBatchedLanePriority) {
    newCallbackNode = scheduleCallback(ImmediateSchedulerPriority, performSyncWorkOnRoot.bind(null, root))
  } else {
    // Concurrent 模式
    newCallbackNode = scheduleCallback(schedulerPriorityLevel, performConcurrentWorkOnRoot.bind(null, root))
  }
}

// 开始执行同步任务
function performSyncWorkOnRoot(root) {
  // render 阶段
  let exitStatus = renderRootSync(root, lanes)
  // commit 阶段
  commitRoot(root)
  // 退出之前确认有其他的等待中的任务，有则继续更新
  ensureRootIsScheduled(root, now())
}
```

## beginWork 流程

### performUnitOfWork

上面源码最后说到了函数 `performSyncWorkOnRoot`，render 阶段真正开始于这个函数，来看看 `renderRootSync` 这个函数：

```ts
function renderRootSync(root, lanes) {
  do {
    try {
      // 同步循环更新
      workLoopSync()
      break
    } catch (thrownValue) {
      handleError(root, thrownValue)
    }
  } while (true)

  // workLoopSync 执行完，重置状态，将进入 commit 阶段
  workInProgressRoot = null
  workInProgressRootRenderLanes = NoLanes
}

function workLoopSync() {
  // Already timed out, so perform work without checking if we need to yield.
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress)
  }
}

// 执行当前工作单元
function performUnitOfWork(unitOfWork) {
  const current = unitOfWork.alternate
  // 处理 current 并返回 child
  let next = beginWork(current, unitOfWork, subtreeRenderLanes)
  unitOfWork.memoizedProps = unitOfWork.pendingProps
  if (next === null) {
    // 没有子节点
    completeUnitOfWork(unitOfWork)
  } else {
    // 处理下一个节点
    workInProgress = next
  }
}
```

- `workInProgress `：指向当前正在处理的 fiber 节点
- `renderRootSync`：执行 workLoopSync 方法，执行完毕后重置状态进入 commit 阶段
- `workLoopSync`：通过循环反复判断 workInProgress 是否为 null，不为 null 就执行 `performUnitOfWork` 函数，直到 workInProgress 为空。
- `performUnitOfWork` 方法，它通过 workInProgress fiber 跟已创建的 fiber 连接起来形成 Fiber 树，这个过程为深度优先遍历，而 beginWork 也在这个方法里被调用。

`beginWork` 是向下调和，`completeUnitOfWork` 是向上归并。

### beginWork 方法概览

```js
function beginWork(current, workInProgress, renderLanes) {
  const updateLanes = workInProgress.lanes
  // update（当满足条件时可复用 current）
  if (current !== null) {
    const oldProps = current.memoizedProps // 上一次渲染的 props
    const newProps = workInProgress.pendingProps // 当前 props

    // props 前后改变 或 有老版本 context 使用并发生变化
    if (oldProps !== newProps || hasLegacyContextChanged()) {
      didReceiveUpdate = true // 标记组件接收到了新的更新
      // 当前 Fiber 节点更新的优先级不够，则子孙节点不需要被调和
      // includesSomeLane 用于判断是否是高优先级的任务
    } else if (!includesSomeLane(renderLanes, updateLanes)) {
      didReceiveUpdate = false

      switch (workInProgress.tag) {
        case HostRoot:
          pushHostRootContext(workInProgress)
          resetHydrationState()
          break
        case HostComponent:
          pushHostContext(workInProgress)
          break
        case ClassComponent: {
          const Component = workInProgress.type
          if (isLegacyContextProvider(Component)) {
            pushLegacyContextProvider(workInProgress)
          }
          break
        }
        // ...
      }
      // 复用 current
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes)
    } else {
      if ((current.flags & ForceUpdateForLegacySuspense) !== NoFlags) {
        didReceiveUpdate = true
      } else {
        didReceiveUpdate = false
      }
    }
  } else {
    // 当前 Fiber 节点未创建
    didReceiveUpdate = false
  }

  workInProgress.lanes = NoLanes

  // 根据 workInProgress 的 tag 属性执行 React 不同类型的节点函数
  // 这些返回的方法内部都会调用 reconcileChildren
  switch (workInProgress.tag) {
    // 函数组件
    case FunctionComponent: {
      return updateFunctionComponent(current, workInProgress, Component, resolvedProps, renderLanes)
    }
    // 类组件
    case ClassComponent: {
      return updateClassComponent(current, workInProgress, Component, resolvedProps, renderLanes)
    }
    // ...省略其它 tag
  }
}
```

`beginWork` 方法整体做的事情：

1. 如果当前 Fiber 为 null 表示首次挂载，则直接进入下面第 2 点进行更新；否则 Fiber 走更新流程（`current !== null`），则需先判断更新情况。
2. 更新 Fiber，根据 Fiber 节点 tag 属性的不同，调用不同节点方法，但底层都会调用 `reconcileChildren` 方法。

由于 `workInProgress.tag` 下的节点类型太多，就不一一举例了，下面就以 `updateClassComponent` 和 `updateFunctionComponent` 为例子简单过下：

### 类组件更新函数：updateClassComponent

主要工作是对未初始化的 ClassComponent 进行初始化，对已经初始化的组件进行更新复用

```js
function updateClassComponent(current, workInProgress, Component, nextProps, renderLanes) {
  // 省略 context 处理逻辑...

  const instance = workInProgress.stateNode
  let shouldUpdate
  // 组件实例未创建
  if (instance === null) {
    if (current !== null) {
      current.alternate = null
      workInProgress.alternate = null
      workInProgress.flags |= Placement
    }
    // 执行构造函数，得到实例 instance
    constructClassInstance(workInProgress, Component, nextProps)
    // 挂载类组件
    mountClassInstance(workInProgress, Component, nextProps, renderLanes)
    // 标记组件需要更新渲染
    shouldUpdate = true
  } else if (current === null) {
    // 组件实例已创建，但 current 为空，即类组件是首次渲染
    // 渲染中断，复用类组件实例
    shouldUpdate = resumeMountClassInstance(workInProgress, Component, nextProps, renderLanes)
  } else {
    // 组件实例已创建，不是首次渲染
    // 更新
    shouldUpdate = updateClassInstance(current, workInProgress, Component, nextProps, renderLanes)
  }
  // 完成类组件更新
  const nextUnitOfWork = finishClassComponent(current, workInProgress, Component, shouldUpdate, hasContext, renderLanes)
  return nextUnitOfWork
}

function finishClassComponent(current, workInProgress, Component, shouldUpdate, hasContext, renderLanes) {
  // ...
  // 没更新且没有错误捕获
  if (!shouldUpdate && !didCaptureError) {
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes)
  }

  const instance = workInProgress.stateNode

  // Rerender
  ReactCurrentOwner.current = workInProgress
  let nextChildren
  // 有错误捕获
  if (didCaptureError && typeof Component.getDerivedStateFromError !== 'function') {
    // 该类组件没有 getDerivedStateFromError 函数
    nextChildren = null

    if (enableProfilerTimer) {
      stopProfilerTimerIfRunning(workInProgress)
    }
  } else {
    // 该类组件有 getDerivedStateFromError 函数
    nextChildren = instance.render()
  }

  // 有错误捕获强行调和子节点
  if (current !== null && didCaptureError) {
    forceUnmountCurrentAndReconcile(current, workInProgress, nextChildren, renderLanes)
  } else {
    // 调和子节点
    reconcileChildren(current, workInProgress, nextChildren, renderLanes)
  }
  workInProgress.memoizedState = instance.state

  return workInProgress.child
}
```

### 函数更新函数：updateFunctionComponent

如当节点是 `FunctionComponent` 的时候，走函数组件的更新流程。

```js
function updateFunctionComponent(current, workInProgress, Component, nextProps, renderLanes) {
  let nextChildren
  // 调用 renderWithHooks 执行函数组件更新，返回的即是函数组件 return 的 jsx
  nextChildren = renderWithHooks(current, workInProgress, Component, nextProps, context, renderLanes)
  // 函数组件能否复用 Fiber，取决于 didReceiveUpdate 这个变量
  if (current !== null && !didReceiveUpdate) {
    bailoutHooks(current, workInProgress, renderLanes)
    // 判断 fiber 的子树是否需要更新
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes)
  }

  // React DevTools reads this flag.
  workInProgress.flags |= PerformedWork
  // 调用 reconcileChildren
  reconcileChildren(current, workInProgress, nextChildren, renderLanes)
  return workInProgress.child
}
```

### reconcileChildren

`reconcileChildren` 方法是 Reconciler（协调器） 模块的核心，作用是 diff，向下生成子节点。

- 如果是 mount 的组件，会创建新的子 Fiber 节点
- 如果是 update 的组件，则会继续遍历子节点，通过 diff 算法进行比较，来判断是否可以复用还是生成新的 Fiber 节点

```js
function reconcileChildren(current, workInProgress, nextChildren, renderLanes) {
  if (current === null) {
    // 对于 mount 的组件
    workInProgress.child = mountChildFibers(workInProgress, null, nextChildren, renderLanes)
  } else {
    // 对于 update 的组件，则继续深度遍历子节点
    workInProgress.child = reconcileChildFibers(workInProgress, current.child, nextChildren, renderLanes)
  }
}
```

`mountChildFibers` 和 `reconcileChildFibers` 这两个方法的逻辑基本一致，区别是 `reconcileChildFibers` 会为生成的 Fiber 节点带上`effectTag` 属性。

可以看出，代码层面区别就是函数的传参不同，该参数 `shouldTrackSideEffects` 表示是否追踪副作用。

```js
export const reconcileChildFibers = ChildReconciler(true)
export const mountChildFibers = ChildReconciler(false)
```

`ChildReconciler` 函数内部定义了大量函数，从函数名(`delete`删除、`place`插入、`update`修改、`create`创建等等)就可以看出这些都是对 Fiber 节点的操作

```js
function ChildReconciler(shouldTrackSideEffects) {
  function deleteChild(returnFiber, childToDelete) {
    if (!shouldTrackSideEffects) {
      // Noop.
      return;
    }
    const deletions = returnFiber.deletions;
    if (deletions === null) {
      returnFiber.deletions = [childToDelete];
      returnFiber.flags |= Deletion;
    } else {
      deletions.push(childToDelete);
    }
  }
  function deleteRemainingChildren(returnFiber, currentFirstChild) {...}
  function placeChild(newFiber, lastPlacedIndex, newIndex) {...}
  function updateElement(returnFiber, current, element, lanes) {...}
  function createChild(returnFiber, newChild, lanes) {...}
  function reconcileChildrenArray(returnFiber, currentFirstChild, newChildren, lanes) {...}
  // ...
}
```

### bailoutOnAlreadyFinishedWork

该函数主要用于检测子节点是否需要更新

```js
function bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes) {
  if (current !== null) {
    // Reuse previous dependencies
    workInProgress.dependencies = current.dependencies
  }

  if (enableProfilerTimer) {
    stopProfilerTimerIfRunning(workInProgress)
  }

  markSkippedUpdateLanes(workInProgress.lanes)

  if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
    // 子节点无需更新
    return null
  } else {
    // 子节点需要更新，clone 并返回
    cloneChildFibers(current, workInProgress)
    return workInProgress.child
  }
}
```

## beginWork 流程图

最后通过一张流程图来总结 beginWork 流程：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b5096f8274b49c3b102d6ac781cf8b3~tplv-k3u1fbpfcp-watermark.image?)

这样 render 阶段的大致源码就算过完了，有些内容相对比较粗略，后续会补上 `completeUnitOfWork` 和 commit 阶段源码等内容的文章。

## 参考文章

- [React 技术揭秘](https://react.iamkasong.com/process/beginWork.html)

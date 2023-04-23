# 深入 React 源码 render 阶段的 completeWork 流程

## 前言

本文源码基于 Reactv17.0.2，阅读本文前需要先了解 React 渲染相关流程原理与重点概念。

上篇文章：[深入 React 源码 render 阶段的 beginWork 流程](https://juejin.cn/post/7218961155900555321)

## completeWork 阶段流程

在上一篇文章 [深入 React 源码 render 阶段的 beginWork 流程](https://juejin.cn/post/7218961155900555321#heading-8) 讲到了函数 `performUnitOfWork`，该函数通过 `beginWork` 来创建当前的子节点直到子节点为空，深度遍历的"递"阶段完成，进入"归"阶段，而"归"阶段就始于 `completeUnitOfWork(unitOfWork)` 函数，这个函数会因上层函数 `workLoopSync` 被循环调用。

```js
function workLoopSync() {
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

由此可以看出，当 `next === null` 时，即当前 Fiber 节点没有子节点时，将调用 `completeUnitOfWork` 函数

`completeUnitOfWork` 做的事情：

1. 从当前节点由下而上循环执行 `completeWork`，遍历兄弟节点和父节点；当存在兄弟节点时，会结束当前调用，触发兄弟节点的 `performUnitOfWork` 循环；当遍历到父节点时，则进入下一轮循环
2. 收集 EffectList（EffectList 是一条用于收集存在 EffectTag 的 Fiber 节点的单向链表，该 EffectList 收集完成后会在 commit 阶段使用。）

```js
function completeUnitOfWork(unitOfWork) {
  // 完成当前节点的 work，然后移动到兄弟节点，当没有兄弟节点时，返回到父节点
  let completedWork = unitOfWork
  do {
    // 记录当前节点
    const current = completedWork.alternate
    // 获取父节点 Fiber
    const returnFiber = completedWork.return

    // workInProgress 节点没有错误抛出，走正常的 complete 流程
    if ((completedWork.flags & Incomplete) === NoFlags) {
      let next
      // ...
      // 执行 completeWork，并把返回值赋值给 next
      next = completeWork(current, completedWork, subtreeRenderLanes)

      if (next !== null) {
        // Completing this fiber spawned new work. Work on that next.
        workInProgress = next
        return
      }
      // 重置子节点的优先级
      resetChildLanes(completedWork)

      if (returnFiber !== null && (returnFiber.flags & Incomplete) === NoFlags) {
        // 父节点没有挂载 firstEffect
        if (returnFiber.firstEffect === null) {
          // 将当前节点的 effectList 合并到到父节点的 effectList
          returnFiber.firstEffect = completedWork.firstEffect
        }
        // 父节点的 lastEffect 有值
        if (completedWork.lastEffect !== null) {
          if (returnFiber.lastEffect !== null) {
            // 串联 EffectList 链
            returnFiber.lastEffect.nextEffect = completedWork.firstEffect
          }
          returnFiber.lastEffect = completedWork.lastEffect
        }

        const flags = completedWork.flags

        if (flags > PerformedWork) {
          if (returnFiber.lastEffect !== null) {
            returnFiber.lastEffect.nextEffect = completedWork
          } else {
            returnFiber.firstEffect = completedWork
          }
          returnFiber.lastEffect = completedWork
        }
      }
    } else {
      // 执行到 else 这里说明之前的更新有错误
      const next = unwindWork(completedWork, subtreeRenderLanes)
      // ...
    }
    // 获取兄弟节点
    const siblingFiber = completedWork.sibling
    // 存在兄弟节点，当前节点结束 completeWork，return 终止循环
    if (siblingFiber !== null) {
      workInProgress = siblingFiber
      return
    }
    // 不存在兄弟节点，则向上回到父节点
    completedWork = returnFiber
    // 将 workInProgress 节点指向父节点
    workInProgress = completedWork
  } while (completedWork !== null)

  // 到达根节点，完成整个树的工作
  if (workInProgressRootExitStatus === RootIncomplete) {
    // 标记完成
    workInProgressRootExitStatus = RootCompleted
  }
}
```

- `fiber.firstEffect`表示挂载到当前 Fiber 节点的 EffectList 的第一个 Fiber 节点，同理 `fiber.lastEffect` 则表示最后一个。

## completeWork

`completeWork` 主要做的事情：

1. mount 阶段：创建 DOM 节点，并把 Fiber 子节点的第一层子节点插入到刚创建的 DOM 节点后面，最后给 DOM 节点设置属性和初始化事件监听器
2. update 阶段：调用 `updateHostComponent` 处理 props，主要为更新旧的 DOM 节点，计算出需要更新的 DOM 节点属性，并给当前节点打上 Update 的 EffectTag。

```js
function completeWork(current, workInProgress, renderLanes) {
  const newProps = workInProgress.pendingProps
  // 根据 workInProgress.tag 进入不同节点处理逻辑函数
  switch (workInProgress.tag) {
    case IndeterminateComponent:
    case LazyComponent:
    case SimpleMemoComponent:
    case FunctionComponent:
    case ForwardRef:
    case Fragment:
    case Mode:
    case ContextConsumer:
    case MemoComponent:
      bubbleProperties(workInProgress)
      return null
    case ClassComponent: {
      const Component = workInProgress.type
      if (isLegacyContextProvider(Component)) {
        popLegacyContext(workInProgress)
      }
      bubbleProperties(workInProgress)
      return null
    }
    // 原生 DOM 组件
    case HostComponent: {
      popHostContext(workInProgress)
      const rootContainerInstance = getRootHostContainer()
      const type = workInProgress.type
      // 更新（当 current 存在且 workInProgress 节点对应的 DOM 实例存在时走更新逻辑）
      if (current !== null && workInProgress.stateNode != null) {
        updateHostComponent(current, workInProgress, type, newProps, rootContainerInstance)

        if (current.ref !== workInProgress.ref) {
          markRef(workInProgress)
        }
      } else {
        // 挂载
        if (!newProps) {
          // This can happen when we abort work.
          bubbleProperties(workInProgress)
          return null
        }
        // 为 DOM 节点的创建做准备
        const currentHostContext = getHostContext()
        // 1. 创建 DOM 节点
        const instance = createInstance(type, newProps, rootContainerInstance, currentHostContext, workInProgress)
        // 2. 把 Fiber 子节点的第一层子节点插入到当前 DOM 后面
        appendAllChildren(instance, workInProgress, false, false)

        workInProgress.stateNode = instance

        // 3. 初始化 DOM 属性和事件监听器
        if (finalizeInitialChildren(instance, type, newProps, rootContainerInstance, currentHostContext)) {
          markUpdate(workInProgress)
        }

        if (workInProgress.ref !== null) {
          markRef(workInProgress)
        }
      }
      // bubbleProperties 数在这个过程中负责将子树的事件属性聚合到父节点，以便在根元素上进行统一处理。
      bubbleProperties(workInProgress)
      return null
    }
    // case ...
    // case ...
  }
}
```

### mount 阶段

#### createInstance

作用：使用 createElement 创建 DOM 节点

```js
export function createInstance(type, props, rootContainerInstance, hostContext, internalInstanceHandle) {
  let parentNamespace
  parentNamespace = hostContext
  // 创建 DOM 元素
  const domElement = createElement(type, props, rootContainerInstance, parentNamespace)
  // 在 DOM 对象上创建指向 fiber 节点对象的属性（指针）
  precacheFiberNode(internalInstanceHandle, domElement)
  // 在 DOM 对象上创建指向 props 的属性（指针）
  updateFiberProps(domElement, props)
  return domElement
}
```

#### appendAllChildren

作用：把 Fiber 子节点的第一层子节点插入到当前 DOM 后面

```js
appendAllChildren = function (parent, workInProgress, needsVisibilityToggle, isHidden) {
  // 找到当前节点的子 Fiber 节点
  let node = workInProgress.child
  // 存在子节点则向下遍历
  while (node !== null) {
    // 子节点是原生 DOM 节点，则直接执行插入操作
    if (node.tag === HostComponent || node.tag === HostText) {
      appendInitialChild(parent, node.stateNode)
    } else if (enableFundamentalAPI && node.tag === FundamentalComponent) {
      appendInitialChild(parent, node.stateNode.instance)
    } else if (node.tag === HostPortal) {
      // do nothing
    } else if (node.child !== null) {
      // 继续查找子节点
      node.child.return = node
      node = node.child
      continue
    }
    if (node === workInProgress) {
      return
    }
    // 不存在兄弟节点，向上回溯
    while (node.sibling === null) {
      if (node.return === null || node.return === workInProgress) {
        return
      }
      node = node.return
    }
    node.sibling.return = node.return
    node = node.sibling
  }
}

// 调用原生的 appendChild 方法
export function appendInitialChild(parentInstance, child) {
  parentInstance.appendChild(child)
}
```

#### finalizeInitialChildren

作用：初始化 DOM 属性和事件监听器

```js
export function finalizeInitialChildren(domElement, type, props, rootContainerInstance, hostContext) {
  setInitialProperties(domElement, type, props, rootContainerInstance)
  return shouldAutoFocusHostComponent(type, props)
}

function setInitialDOMProperties(tag, domElement, rootContainerElement, nextProps, isCustomComponentTag) {
  for (const propKey in nextProps) {
    if (!nextProps.hasOwnProperty(propKey)) {
      continue
    }
    const nextProp = nextProps[propKey]
    if (propKey === STYLE) {
      // 设置行内样式
      setValueForStyles(domElement, nextProp)
    } else if (propKey === DANGEROUSLY_SET_INNER_HTML) {
      // 设置 innerHTML
      const nextHtml = nextProp ? nextProp[HTML] : undefined
      if (nextHtml != null) {
        setInnerHTML(domElement, nextHtml)
      }
    } else if (propKey === CHILDREN) {
      if (typeof nextProp === 'string') {
        const canSetTextContent = tag !== 'textarea' || nextProp !== ''
        if (canSetTextContent) {
          setTextContent(domElement, nextProp)
        }
      } else if (typeof nextProp === 'number') {
        setTextContent(domElement, '' + nextProp)
      }
    } else if (propKey === SUPPRESS_CONTENT_EDITABLE_WARNING || propKey === SUPPRESS_HYDRATION_WARNING) {
      // Noop
    } else if (propKey === AUTOFOCUS) {
      // do nothing
    } else if (registrationNameDependencies.hasOwnProperty(propKey)) {
      // 绑定事件
      if (nextProp != null) {
        if (!enableEagerRootListeners) {
          ensureListeningTo(rootContainerElement, propKey, domElement)
        } else if (propKey === 'onScroll') {
          listenToNonDelegatedEvent('scroll', domElement)
        }
      }
    } else if (nextProp != null) {
      // 设置属性
      setValueForProperty(domElement, propKey, nextProp, isCustomComponentTag)
    }
  }
}
```

### update 阶段

#### updateHostComponent

我们以 `HostComponent` 为例，讲讲它的处理逻辑。

`updateHostComponent` 函数作用是更新旧的 DOM 节点，计算出需要更新的 DOM 节点属性，并给当前节点打上 Update 的 EffectTag。

```js
updateHostComponent = function (current, workInProgress, type, newProps, rootContainerInstance) {
  const oldProps = current.memoizedProps
  // props 前后没有发生变化，则直接返回
  if (oldProps === newProps) {
    return
  }

  const instance = workInProgress.stateNode
  const currentHostContext = getHostContext()

  // 计算出需要更新的 DOM 节点属性
  const updatePayload = prepareUpdate(instance, type, oldProps, newProps, rootContainerInstance, currentHostContext)
  // 新属性被挂载到 workInProgress.updateQueue 中，以便在 commit 阶段进行统一处理
  workInProgress.updateQueue = updatePayload

  if (updatePayload) {
    // 标记 workInProgress 节点有更新（标记上 Update 的 EffectTag）
    markUpdate(workInProgress)
  }
}

function markUpdate(workInProgress) {
  // Tag the fiber with an update effect. This turns a Placement into
  // a PlacementAndUpdate.
  workInProgress.flags |= Update
}
```

## completeWork 流程图

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc83d5486dc549eb97472e975efd7e03~tplv-k3u1fbpfcp-watermark.image?)

当所有节点都完成 `completeWork` 时，更新工作就完成了，然后 `workInProgress` 树会进入 `commit` 阶段，这个后续会继续写文补充这个流程。

## 参考文章

- [深挖 React 的 completeWork](https://juejin.cn/post/6919629280012042254)
- [React 源码解析系列 - React 的 render 阶段（三）：completeUnitOfWork](https://juejin.cn/post/7017703203525361695)

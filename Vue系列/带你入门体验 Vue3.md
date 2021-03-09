# 带你入门体验 Vue3

## 前言

一直都是 React 进行开发，虽然 Vue 是接触最早的，但现在实际工作也不怎么有机会用，Vue3 都出了。其中新写法有点像 React Hook，于是，这段时间迅速对 Vue3 进行了基本知识入门，体验下 Vue3。

## Vue2.x 有哪些痛点

首先 Vue3 出现之前，让我们来理理 Vue2.x 有什么痛点，需要 Vue3 补上的。

- Vue2.x 对数组对象的深层监听无法实现。因为组件每次渲染都是将 data 里的数据通过 defineProperty 进行响应式或者双向绑定上，之前没有后加的属性是不会被绑定上，也就不会触发更新渲染。
- 代码的可读性随着组件变大而变差
- 每一种代码复用的方式，都存在缺点
- vue2.x 对 typescript 支持不太友好，需要使用一堆装饰器语法

## Options API 到 Composition API 的转变

Options API 即是通过定义 methods，computed，watch，data 等属性与方法，共同处理页面逻辑。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9c1026434744a1bae136ac31e1bcac4~tplv-k3u1fbpfcp-watermark.image)

可以看到 Options 代码编写方式，如果是组件状态，则写在 data 属性上，如果是方法，则写在 methods 属性上

当组件变得复杂时，会导致对应属性的列表也会增长，这可能会导致组件难以阅读和理解。

下面表示组件是一个大型组件（不同颜色表示不同逻辑），会看到同个功能逻辑的代码被分散到各处，这种碎片化使得理解和维护复杂组件变得困难，需要不断"跳"到相关代码处。
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/de12120d2468456fb75465ecae3b9ee0~tplv-k3u1fbpfcp-watermark.image)

而 Composition API 是根据逻辑功能来组织的，一个功能所定义的所有 API 会放在一起（更加的高内聚，低耦合）

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f21b64d574254722b9a13d3b491aeedb~tplv-k3u1fbpfcp-watermark.image)

用 Composition API 来表示一个大型组件与上面 Options API 的对比，可以直观地感受到 Composition API 在逻辑组织方面的优势，以后修改一个属性功能的时候，只需要跳到控制该属性的方法中即可，在逻辑组织和逻辑复用方面更好。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4cd55d676c034223a99adc64936d39ef~tplv-k3u1fbpfcp-watermark.image)

Options API 是面向对象的思想，Composition API 是函数式编程的思想。

Vue3 将 Vue2.x 里面的 Options API，细粒度拆分成细小的函数，然后统一放到 setup 这一个入口函数中调用。

这样做的好处：

- 保留了 Vue2.x 中各个函数的功能，做到兼容
- 同时以小的函数形式被调用，更加灵活，更好维护，更好 tree-shaking

## vue3 有哪些优化

- 打包大小减少 41%，初次渲染快 55%，更新快 133%，内存使用减少 54%（官方数据）
- 采用 Typescript 开发，更好支持 Typescript
- 使用 Proxy 代替 vue2.x 中的 defineProperty，能够深层监听数组对象的变化。
- Options API 到 Composition API，使复杂组件逻辑复用变得更容易和维护。

## Composition API

字面意思就是组合式 API，也就是将原来的很多底层的方法拆分开，暴露出来让大家去使用。

### setup

`setup` 是 Composition API 的核心，它在`props`、`data`、`computed`、`methods`、`生命周期函数`之前运行的。

它返回一个对象，该对象上的属性将合并到组件模板的渲染上下文中，还可以返回一个 render 函数。

```javascript
<template>
  <div>setup</div>
</template>

<script>
export default {
  setup() {
    // Composition API 逻辑组织的入口
  },
}
</script>
```

它将 Vue2.x 中的 `beforeCreate` 和 `created` 代替了，以一个 `setup` 函数的形式灵活组织代码；还可以 return 数据或者 template，相当于把 data 和 render 也一并代替了。

```javascript
<template>
  <div>{{ foo }}</div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'

export default defineComponent({
  setup() {
    const foo = 2 // 普通变量

    return {
      foo, // 必须 return 出去才能在模板使用
    }
  },
})
</script>
```

> 顺带一提，通常为了在 TypeScript 下能让组件有正确的参数类型推断、IDE 有更好的提示效果，会在`export default`处再包一层函数`defineComponent`（该函数只返回传递给它的对象）

```javascript
<script lang="ts">
import { defineComponent } from 'vue'

export default defineComponent({
  setup() {
    // Composition API 逻辑组织的入口
  },
})
</script>
```

**`setup`函数使用需要注意的点：**

- 由于在执行`setup`函数的时候，还没有执行`created` 生命周期方法，所以在`setup` 函数中，无法使用 `data`和 `methods` 的变量和方法
- `setup`函数只能是同步的不能是异步的

由于 `setup` 是一个入口函数，本质是面向函数编程，而 `this` 是面向对象的一种体现，这里其实相当于取消`this`了，在 Vue3`setup` 函数里的`this`为 `undefined`。

而取消了`this`，取而代之的是`setup`增加了 2 个参数：

```
props：组件参数
context：上下文信息（如attrs、slots、emit）
```

```javascript
export default defineComponent({
  props: {
    title: String,
  },
  setup(props, context) {
    const foo = 2
    console.log(props.title, context)
    return {
      foo,
    }
  },
})
</script>
```

### ref

接受一个内部值并返回一个响应式且可变的 ref 对象（响应式对象），`ref()`创建的数据会触发模版更新。ref 对象具有指向内部值的单个 property `.value`。

```javascript
<template>
  <div>
    {{ count }}
    <button @click="handleClick">加1</button>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    // 被 ref 方法包裹后的元素就变成了一个代理对象
    const count = ref(0)

    const handleClick = () => {
      count.value++ // 在setup里面使用ref代理对象需要 `.value`，模板中使用时由于vue帮我们做了自动解析所以不用 `.value`
    }

    return {
      count,
      handleClick,
    }
  },
})
</script>
```

### reactive

reactive 也是实现响应式的一种方法，它接收一个普通对象然后返回该普通对象的响应式代理，相当于 Vue2.x 中的 `Vue.observable()`

一般约定 reactive 的参数是一个对象，而`ref`的参数通常是原始数据类型，虽然反过来也可以。

`ref`的本质还是 reactive 系统会自动根据`ref()`函数的入参将其转换成`ref(x)`即`reactive({ value:x })`

```javascript
<template>
  <div>
    {{ obj1.count1 }}
    {{ count2 }}
  </div>
</template>

<script lang="ts">
import { defineComponent, reactive, toRefs } from 'vue'

export default defineComponent({
  setup() {
    const obj1 = reactive({
      count1: 10,
    })
    const obj2 = reactive({
      count2: 20,
    })

    return {
      obj1,
      ...toRefs(obj2),
    }
  },
})
</script>
```

上面代码可以看到，直接 return obj1，在模板使用时需要`obj.count1`，那为什么不直接`...obj1`呢，那是因为 obj1 是`proxy`代理对象，整个对象都是响应式，所以不能使用剩余运算符。

如果觉得麻烦，可以使用` ...toRefs(obj2)`，这样就可以在模板直接使用定义的数据了。

> `toRefs` 函数是将`reactive`创建的响应式对象，转化成为普通的对象，并且这个对象上的每个节点，都是`ref`类型的响应式数据

### toRef

可以用来为源响应式对象上的某个 property 新创建一个 ref。然后，ref 可以被传递，它会保持对其源 property 的响应式连接。

```javascript
export default defineComponent({
  setup() {
    const state = reactive({
      foo: 1,
      bar: 2,
    })

    const fooRef = toRef(state, 'foo')

    fooRef.value++
    console.log(state.foo) // 2

    state.foo++
    console.log(fooRef.value) // 3
  },
})
```

`toRef()`接收两个参数，第一个为对象，第二个为对象中的某个属性。它是对原数据的一个引用，当值改变时会影响到原始值；创建的响应式数据并不会触发 vue 模版更新。

### toRefs

将响应式对象转换为普通对象，其中结果对象的每个 property 都是指向原始对象相应 property 的 ref。

`toRefs()`接收一个对象作为参数，并遍历对象身上的所有属性，然后逐个调用`toRef()`执行。以此，将响应式对象转化为普通对象，便于在模版中可以直接使用属性。

通常用来将一组的响应式对象拆成单个的响应式对象，如将一个 reactive 代理对象打平，转换为 ref 代理对象，使得对象的属性可以直接在 template 上使用。

```javascript
<template>
  <div>
    {{ foo }}
    {{ bar }}
  </div>
</template>

<script lang="ts">
import { defineComponent, reactive, toRefs } from 'vue'

export default defineComponent({
  setup() {
    // 创建一个响应式对象state
    const state = reactive({
      foo: 1,
      bar: 2,
    })

    const stateAsRefs = toRefs(state) // 将响应式的对象变为普通对象结构, 且能使用ES6的扩展运算符，也仍具有响应性

    // ref 和原始 property 已经“链接”起来了
    state.foo++
    console.log(stateAsRefs.foo.value) // 2

    stateAsRefs.foo.value++
    console.log(state.foo) // 3

    return {
      ...stateAsRefs,
    }
  },
})
</script>
```

### computed

接受一个 getter 函数，并为从 getter 返回的值返回一个不变的响应式 ref 对象。

```javascript
setup() {
  const count = ref(1)
  const plusOne = computed(() => count.value + 1) // 结果是一个 ref 代理对象，取值需要 .value

  console.log(plusOne.value) // 2
},
```

与 Vue2.x 中的作用类似，获取一个计算结果。当 computed 参数使用 object 对象书写时，不仅支持取值 get（默认），还支持赋值 set。

```javascript
setup() {
  const count = ref(1)
  const plusOne = computed({
    get: () => count.value + 1,
    set: (val) => {
      count.value = val - 1
    },
  })

  plusOne.value = 1
  console.log(count.value) // 0
},
```

### watch

watch API 与选项式 API `this.$watch` (以及相应的 watch 选项) 完全等效。watch 需要侦听特定的数据源，并在单独的回调函数中执行副作用。默认情况下，它也是惰性的——即回调仅在侦听源发生更改时被调用。

```javascript
<template>
  <div>
    {{ count }}
    <button @click="changeCount">count加1</button>
  </div>
</template>

<script lang="ts">
import { defineComponent, watch, ref } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)
    const changeCount = () => {
      count.value++
    }
    watch(count, (newCount, prevCount) => {
      console.log('newCount:', newCount, ' oldCount:', prevCount)
    })

    return {
      count,
      changeCount,
    }
  },
})
</script>
```

也可以使用数组同时侦听多个源。

### watchEffect

在响应式地跟踪其依赖项时立即执行传入的一个函数，并在更改依赖项时重新运行它。

当组件的 setup()或者生命周期钩子被调用时，watchEffect 会被链接到该组件的生命周期，并在组件卸载时自动停止。

```javascript
<template>
  <div>
    {{ count }}
    <button @click="changeCount">count加1</button>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, watchEffect } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)
    const changeCount = () => {
      count.value++
    }

    watchEffect(() => console.log(count.value))
    // -> logs 0

    setTimeout(() => {
      count.value++
      // -> logs 1
    }, 100)

    return {
      count,
      changeCount,
    }
  },
})
</script>
```

**watch 与 watchEffect 比较，watch 允许我们**

- 惰性地执行副作用；
- 更具体地说明应触发侦听器重新运行的状态；
- 访问被侦听状态的先前值和当前值。

### readonly

接受一个对象 (响应式或纯对象) 或 ref 并返回原始对象的只读代理。只读代理是深层的：任何被访问的嵌套 property 也是只读的。

```javascript
setup() {
  const original = reactive({ count: 0 })

  // 返回的 readonly 对象，一旦修改就会在 console 有 warning 警告。程序还是会照常运行，不会报错。
  const copy = readonly(original)

  watchEffect(() => {
    // 只要有数据变化，这个函数都会执行
    console.log(copy.count)
  })

  // 这里会触发 watchEffect
  original.count++

  // 这里不会触发上方的 watchEffect，因为是 readonly。
  copy.count++ // warning!
},
```

### Fragments Template

Vue3.x 中，可以不用唯一根节点。

```
<template>
  <div></div>
  <div></div>
</template>
```

### Teleport 组件

Teleport 在国内翻译成了瞬间移动组件或者独立组件，它可以把你写的组件挂载到任何你想挂载的 DOM 上。

```javascript
<template>
  <div class="user-card">
    <b> {{ name }} </b>
    <button @click="isModalOpen = true">删除用户</button>

    <!-- 注意这一块代码 -->
    <Teleport to="#modal">
      <div v-show="isModalOpen">
        <p>确定删除?</p>
        <button @click="removeUser">确定</button>
        <button @click="isModalOpen = false">取消</button>
      </div>
    </Teleport>
  </div>
</template>

<script lang="ts">
import { defineComponent, reactive, ref, toRefs } from 'vue'

export default defineComponent({
  setup() {
    const isModalOpen = ref(false)
    const user = reactive({
      name: 'jacky',
    })

    const removeUser = () => {
      console.log('removeUser')
    }

    return {
      isModalOpen,
      removeUser,
      ...toRefs(user),
    }
  },
})
</script>
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b44366dd9a654ef9b540d89132d43936~tplv-k3u1fbpfcp-watermark.image)

运行这段代码之前，需要去`./public/index.html`加上个 id 为`modal`元素供挂载，当然也可以到组件里面添加。

```html
<div id="app"></div>
<div id="modal"></div>
```

Teleport 提供了一种干净的方法，允许我们控制在 DOM 中哪个父节点下渲染了 HTML，而不必求助于全局状态或将其拆分为两个组件。

### Suspense 组件

Suspense 提供两个 template 的位置，一个是没有请求未完成或失败显示的内容，一个是全部请求完毕的内容。这样进行异步内容的渲染就会非常简单。

下面 Demo 我就模拟一下用法，不写真实请求了。

新建`AsyncShow.vue`组件

```javascript
<template>
  <div>
    <h1>{{ result }}</h1>
  </div>
</template>

<script>
import { defineComponent } from 'vue'

export default defineComponent({
  name: 'AsyncShow',
  setup() {
    return new Promise((resolve) => {
      setTimeout(() => {
        return resolve({
          result: '这是请求结果',
        })
      }, 3000)
    })
  },
})
</script>
```

新建`Suspense.vue`组件

```javascript
<template>
  <div>
    <Suspense>
      <template #default>
        <!-- 请求成功展示 -->
        <async-show />
      </template>
      <template #fallback>
        <!-- 请求中和失败展示 -->
        <h2>Promise Loading...（请求中）</h2>
      </template>
    </Suspense>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'
import AsyncShow from './AsyncShow.vue'

export default defineComponent({
  components: {
    AsyncShow,
  },
  setup() {
    //
  },
})
</script>
```

## Vue3.x 生命周期变化

### 被替换

- beforeCreate -> setup()
- created -> setup()

### 重命名

生命周期方法前面都统一加了`on`；`destroy`被改名成`Unmount`更贴切也对应`Mount`

- beforeMount -> onBeforeMount
- mounted -> onMounted
- beforeUpdate -> onBeforeUpdate
- updated -> onUpdated
- beforeDestroy -> onBeforeUnmount
- destroyed -> onUnmounted
- errorCaptured -> onErrorCaptured

### 新增

新增的以下 2 个方便调试 debug 的回调钩子：

- onRenderTracked：状态跟踪，它会跟踪页面上所有响应式变量和方法的状态，也就是我们用 return 返回去的值，他都会跟踪。只要页面有 update 的情况，他就会跟踪，然后生成一个 event 对象，我们通过 event 对象来查找程序的问题所在。
- onRenderTriggered：状态触发，它不会跟踪每一个值，而是给你变化值的信息，并且新值和旧值都会给你明确的展示出来。

### 新旧生命周期谁先运行？

- 在 Vue2.x 中通过补丁形式引入 Composition API，进行 Vue2.x 和 Vue3.x 的回调函数混用时：Vue2.x 的回调函数会相对先执行，比如：mounted 优先于 onMounted。
- 在 Vue3.x 中，为了兼容 Vue2.x 的语法，所有旧的生命周期函数得到保留（除了 beforeDestroy 和 destroyed）。当生命周期混合使用时：Vue3.x 的生命周期相对优先于 Vue2.x 的执行，比如：onMounted 比 mounted 先执行。

## 结语

暂时就写这么多了，并不完全，但也能大概对 Vue3 有个初步了解了。

本文案例代码：[vue3-tutorial](https://github.com/Jacky-Summer/vue-tutorial)

## 参考

- [vue2.x/React/vue3.x 简单横评（4）](https://juejin.cn/post/6844904096265142279)
- [vue3 composition api](https://juejin.cn/post/6899702587977646088)
- [Vue3.0 所采用的 Composition Api 与 Vue2.x 使用的 Options Api 有什么不同？](https://www.cnblogs.com/houxianzhou/p/14368919.html)

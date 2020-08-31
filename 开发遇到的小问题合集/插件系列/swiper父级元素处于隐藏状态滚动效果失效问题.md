# swiper 父级元素处于隐藏状态滚动效果失效问题

在使用 vue-awesome-swiper 的时候，实现这样一个功能, 点击页面的 banner 图（Banner.vue），进入一个图片画廊（Gallary.vue），可以查看更多相关的照片，但发现轮播图滚动效果失效
![](https://user-gold-cdn.xitu.io/2019/12/25/16f3aa7f03be6109?w=432&h=764&f=gif&s=1825248)

原因是：一开始 Gallay.vue 的所有用 swiper 控制的图片处于隐藏状态，直到点击才进入图片画廊显示图片。swiper 其父级元素处于隐藏状态(display:none), 会导致 swiper 初始化失败, 页面中的滚动效果就会有问题。

Gallary.vue 的结构

```html
<template>
  <div class="container" @click="handleGallaryClick">
    <div class="wrapper">
      <swiper :options="swiperOption">
        <swiper-slide v-for="(item, index) of imgs" :key="index">
          <img class="swiper-img" :src="item" />
        </swiper-slide>
        <div class="swiper-pagination" slot="pagination"></div>
      </swiper>
    </div>
  </div>
</template>
```

一开始 Gallary 组件都是通过父组件 Banner 的 v-show 值设置处于隐藏状态

解决办法：
通过 swiper 的配置项：

> observer
> 启动动态检查器(OB/观众/观看者)，当改变 swiper 的样式（例如隐藏/显示）或者修改 swiper 的子元素时

> observeParents
> 将 observe 应用于 Swiper 的父元素。当 Swiper 的父元素变化时，例如 window.resize，Swiper 更新。

```javascript
data () {
    return {
        swiperOption: {
            pagination : {
                el: '.swiper-pagination',
                type: 'fraction'
            },
            observeParents: true, //修改swiper的父元素时，自动初始化swiper
            observer: true   //修改swiper自己或子元素时，自动初始化swiper
        }
    }
},
```

![](https://user-gold-cdn.xitu.io/2019/12/25/16f3ab29259088a7?w=431&h=759&f=gif&s=974520)

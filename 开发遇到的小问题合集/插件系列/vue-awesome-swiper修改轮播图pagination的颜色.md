# vue-awsome-swiper 修改轮播图 pagination 的颜色

在使用 vue-awsome-swiper 轮播图的时候，pagination 处于当前选中状态默认是蓝色，但通过修改这个类并无法改变其背景颜色

```html
<div class="wrapper">
  <swiper :options="swiperOption">
    <swiper-slide v-for="(item, index) of swiperList" :key="index">
      <img class="swiper-img" :src="item.imgUrl" />
    </swiper-slide>
    <div class="swiper-pagination" slot="pagination"></div>
  </swiper>
</div>
```

![](https://user-gold-cdn.xitu.io/2019/12/22/16f2bbbf5fbc739b?w=433&h=140&f=png&s=141401)

原因是为 style 设置了 scoped 以后，swiper 分页样式就失效了。分页是在 mounted 里创建的，此时创建的 DOM，vue 不会帮 swiper 的 pagination 加上 scoped 自定义属性。

解决办法：

```css
.wrapper >>> .swiper-pagination-bullet-active
    background: #ff0
```

    .wrapper >>> .swiper-pagination-bullet-active
    表示的是在wrapper下所有出现.swiper-pagination-bullet-active

效果如下：
![](https://user-gold-cdn.xitu.io/2019/12/22/16f2bc67d4dbfa0f?w=426&h=136&f=png&s=151132)

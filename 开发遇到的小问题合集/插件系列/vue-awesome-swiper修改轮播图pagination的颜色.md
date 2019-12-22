# vue-awsome-swiper修改轮播图pagination的颜色

在使用vue-awsome-swiper轮播图的时候，pagination处于当前选中状态默认是蓝色，但通过修改这个类并无法改变其背景颜色
```html
<div class="wrapper">
    <swiper :options="swiperOption">
        <swiper-slide v-for="(item, index) of swiperList" :key="index">
            <img class="swiper-img" :src="item.imgUrl">
        </swiper-slide>
        <div class="swiper-pagination" slot="pagination"></div>
    </swiper>
</div>
```
![](https://user-gold-cdn.xitu.io/2019/12/22/16f2bbbf5fbc739b?w=433&h=140&f=png&s=141401)

原因是为style设置了scoped以后，swiper分页样式就失效了。分页是在mounted里创建的，此时创建的DOM，vue不会帮swiper的pagination加上scoped自定义属性。

解决办法：
```css
.wrapper >>> .swiper-pagination-bullet-active
    background: #ff0
```
    .wrapper >>> .swiper-pagination-bullet-active
    表示的是在wrapper下所有出现.swiper-pagination-bullet-active
    
效果如下：
![](https://user-gold-cdn.xitu.io/2019/12/22/16f2bc67d4dbfa0f?w=426&h=136&f=png&s=151132)
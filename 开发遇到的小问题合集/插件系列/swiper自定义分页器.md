### 问题:不想使用swiper(用的是4.5.1版本)的自带的圆钮式的分页器，需要自定义样式同时分页器需要摆在图片外面

解决办法:利用swiper提供的renderCustom()方法

html文件
```html
<div class="swiper-box">
    <div class="swiper-container">
        <div class="swiper-wrapper">
            <div class="swiper-slide"><img src="./img/IMG_20191109_141728.jpg" alt=""></div>
            <div class="swiper-slide"><img src="./img/IMG_20191109_154050.jpg" alt=""></div>
            <!--分页器要放图片外显示的话，需要移出来-->
            <!--<div class="swiper-pagination"></div>-->
        </div> 
    </div>
    <div class="swiper-pagination"></div>
</div>
```
js文件
```javascript
var mySwiper = new Swiper ('.swiper-container', {
        autoplay: {
            delay: 3000,
            disableOnInteraction: false
        },
        pagination: {
            el: '.swiper-pagination',
            type: 'custom',
            renderCustom: function (swiper, current, total) {
                var customPaginationHtml = "";
                for(var i = 0; i < total; i++) {
                    // 判断哪个分页器此刻应该被激活
                    if(i == (current - 1)) {
                        customPaginationHtml += '<span class="swiper-pagination-customs swiper-pagination-customs-active"></span>';
                    } else {
                        customPaginationHtml += '<span class="swiper-pagination-customs"></span>';
                    }
                }
                return customPaginationHtml;
            }
        },
    });
```
css文件
```css
.swiper-box{
    position: relative;
    margin: 0 auto;
}
.swiper-container{
    width: 74vw;
    height: 34vw;
    margin-top: 4vw;
}
.swiper-container .swiper-slide img{
    width: 100%;
}
.swiper-pagination{
    text-align: center;
    width: 74vw;
}
.swiper-pagination-bullet{
    margin: 4vw
}
/* 包裹自定义分页器的div的位置的样式 */
.swiper-pagination-custom {
    bottom: -5vw;
}
.swiper-pagination-customs {
    background-image: url(../img/point-grey.png); /* 未轮播到的图片分页样式 */
    display: inline-block;
    background-repeat: no-repeat;
    background-size: contain;
    width: 1.4vw;
    height: 1.4vw;
    margin-left: 2vw;
}
/*自定义分页器激活时的样式表现*/
.swiper-pagination-customs-active {
    background-image: url(../img/point-green.png);
}
```
效果如下：![1573913526\(1\).jpg](/img/bVbAprI)

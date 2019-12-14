# 自定义checkbox样式

由于原生的checkbox样式比较难看，所以我们经常需要改写它的样式，美化复选框，所以今天总结下自定义checkbox样式的方法，上代码：

html部分
```html
<label class="checkbox-inline">
    <input type="checkbox" class="checkbox" name="hobby">
    <span class="hobby">羽毛球</span>
    <span class="checkmark"></span>
</label>
<label class="checkbox-inline">
    <input type="checkbox" class="checkbox" name="hobby">
    <span class="hobby">跑步</span>
    <span class="checkmark"></span>
</label>
```
css部分
```css
.checkbox-inline{
    position: relative;
}
.checkbox{
    position: absolute;
    width: 0;
    height: 0;
}
.checkmark{
    position: absolute;
    top: 2px;
    left: 0;
    height: 15px;
    width: 15px;
    border: 1px solid;
    background-color: #fff;
    border-radius: 10px;
}
.hobby{
    padding-left: 20px;
}
.checkbox-inline input:checked ~ .checkmark {
    background-color: #492c94;
}
.checkbox-inline input:checked ~ .checkmark:after {
    display: block;
}
.checkbox-inline .checkmark:after {
    display: none;
    content: "";
    position: absolute;
    left: 4px;
    top: 0;
    width: 4px;
    height: 10px;
    border: solid white;
    border-width: 0 2px 2px 0;
    -webkit-transform: rotate(45deg);
    -ms-transform: rotate(45deg);
    transform: rotate(45deg);
}
```
效果图（未选中与选中对比）：

![](https://user-gold-cdn.xitu.io/2019/12/14/16f04ab69b849cda?w=199&h=68&f=png&s=3118)

采取的做法是
1. 先将原生的checkbox样式隐藏，设置宽高为0，使其隐藏不占位。
2. 在label标签里面添加额外的span标签，用来做自定义样式
3. 添加伪元素，利用css3的transform属性做矩形旋转，做出打勾的样式，并将其隐藏。
4. 当checkbox被选中时，利用css3的选择器:checked，匹配每个已被选中的 checkbox 元素；再使用兄弟选择符~，匹配到同层级checkmark类，就可以显示和触发自定义checkbox选中状态的样式了

这里还涉及到一个知识点，伪元素属于主元素的一部分，因此点击伪元素触发的是主元素的click事件。
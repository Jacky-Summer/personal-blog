# 完全搞懂 BFC

## 什么是 BFC

BFC 全称是 Block Formatting Context，即块格式化上下文。

除了 BFC，还有：

- IFC（行级格式化上下文）- inline 内联
- GFC（网格布局格式化上下文）- `display: grid`
- FFC（自适应格式化上下文）- `display: flex`或`display: inline-flex`

> 注意：同一个元素不能同时存在于两个 BFC 中

它是 Web 页面的可视 CSS 渲染的一部分，是块盒子的布局过程发生的区域，也是浮动元素与其他元素交互的区域。怎么理解呢？实际就是说 BFC 是一个渲染区域，并且有自己的一套渲染规则，使其内部布局的元素具有一些特性。

**BFC 提供一个独立的布局环境，BFC 内部的元素布局与外部互不影响。**

![](https://user-gold-cdn.xitu.io/2020/7/4/17318faf333483aa?w=798&h=317&f=png&s=23237)

## 块级元素

CSS 属性值 display 为 block，list-item，table 的元素。

块级盒具有以下特性：

- CSS 属性值 display 为 block，list-item，table 时，它就是块级元素
- 布局上，块级盒呈现为竖直排列的块
- 每个块级盒都会参与 BFC 的创建
- 每个块级元素都会至少生成一个块级盒，称为主块级盒；一些元素可能会生成额外的块级盒，比如 `<li>`，用来存放项目符号

## 创建 BFC

以下元素会创建 BFC

- 根元素（`<html>`）
- 浮动元素（`float`不为`none`）
- 绝对定位元素（`position`为`absolute`或`fixed`）
- 表格的标题和单元格（`display` 为 `table-caption`，`table-cell`）
- 匿名表格单元格元素（`display` 为 `table` 或 `inline-table`）
- 行内块元素（`display` 为 `inline-block`）
- `overflow` 的值不为 `visible` 的元素
- 弹性元素（`display` 为 `flex` 或 `inline-flex` 的元素的直接子元素）
- 网格元素（`display` 为 `grid` 或 `inline-grid` 的元素的直接子元素）

以上是 CSS2.1 规范定义的 BFC 触发方式，在最新的 CSS3 规范中，弹性元素和网格元素会创建 F(Flex)FC 和 G(Grid)FC。

## BFC 的特性

- BFC 是页面上的一个独立容器，容器里面的子元素不会影响外面的元素。
- BFC 内部的块级盒会在垂直方向上一个接一个排列
- 同一 BFC 下的相邻块级元素可能发生外边距折叠，创建新的 BFC 可以避免外边距折叠
- 每个元素的外边距盒（margin box）的左边与包含块边框盒（border box）的左边相接触（从右向左的格式的话，则相反），即使存在浮动
- 浮动盒的区域不会和 BFC 重叠
- 计算 BFC 的高度时，浮动元素也会参与计算

如果不太理解的，在下面 BFC 的应用我会提及。

## BFC 的应用

### 自适应两列布局

左列浮动（定宽或不定宽都可以），给右列开启 BFC。

```
/* html 代码 */
<div>
    <div class="left">浮动元素，无固定宽度</div>
    <div class="right">自适应</div>
</div>

/* css 代码 */
* {
    margin: 0;
    padding: 0;
}
.left {
    float: left;
    height: 200px;
    margin-right: 10px;
    background-color: red;
}
.right {
    overflow: hidden;
    height: 200px;
    background-color: yellow;
}
```

![](https://user-gold-cdn.xitu.io/2020/7/4/1731a0544fd4461f?w=1919&h=234&f=png&s=13127)

1. 将左列设为左浮动，将自身高度塌陷，使得其它块级元素可以和它占据同一行的位置。
2. 右列为 div 块级元素，利用其自身的流特性占满整行。
3. 右列设置`overflow: hidden`,触发 BFC 特性，使其自身与左列的浮动元素隔离开，不占满整行。

这即是上面说的 BFC 的特性之一：**浮动盒的区域不会和 BFC 重叠**

### 防止外边距（margin）重叠

#### 兄弟元素之间的外边距重叠

```
/* html 代码 */
<div>
    <div class="child1"></div>
    <div class="child2"></div>
</div>

/* css 代码 */
* {
    margin: 0;
    padding: 0;
}
.child1 {
    width: 100px;
    height: 100px;
    margin-bottom: 10px;
    background-color: red;
}
.child2 {
    width: 100px;
    height: 100px;
    margin-top: 20px;
    background-color: green;
}
```

![](https://user-gold-cdn.xitu.io/2020/7/4/17318cb5761f328e?w=183&h=220&f=png&s=2328)

两个块级元素，红色 div 距离底部 10px，绿色 div 距离顶部 20px，按道理应该两个块级元素相距 30px 才对，但实际却是取距离较大的一个，即 20px。

> 块级元素的上外边距和下外边距有时会合并（或折叠）为一个外边距，其大小取其中的较大者，这种行为称为外边距折叠（重叠），注意这个是发生在属于同一 BFC 下的块级元素之间

根据 BFC 特性，创建一个新的 BFC 就不会发生 margin 折叠了。比如我们在他们两个 div 外层再包裹一层容器，加属性`overflow: hidden`，触发 BFC，那么两个 div 就不属于同个 BFC 了。

```
/* html 代码 */
<div>
    <div class="parent">
        <div class="child1"></div>
    </div>
    <div class="parent">
        <div class="child2"></div>
    </div>
</div>

/* css 代码 */
.parent {
    overflow: hidden;
}
/* ... */
```

这个关于兄弟元素外边距叠加的问题，除了触发 BFC 也有其他方案，比如你统一只用上边距或下边距，就不会有上面的问题。

![](https://user-gold-cdn.xitu.io/2020/7/4/17318dc6636f418f?w=209&h=231&f=png&s=2376)

#### 父子元素的外边距重叠

这种情况存在父元素与其第一个或最后一个子元素之间（嵌套元素）。
如果在父元素与其第一个/最后一个子元素之间不存在边框、内边距、行内内容，也没有创建块格式化上下文、或者清除浮动将两者的外边距 分开，此时子元素的外边距会“溢出”到父元素的外面。

如下代码：

```
/* HTML 代码 */
<div id="parent">
  <div id="child"></div>
</div>

/* CSS 代码 */
* {
    margin: 0;
    padding: 0;
}
#parent {
    width: 200px;
    height: 200px;
    background-color: green;
    margin-top: 20px;
}
#child {
    width: 100px;
    height: 100px;
    background-color: red;
    margin-top: 30px;
}
```

![](https://user-gold-cdn.xitu.io/2020/7/4/1731a48a81bc27d9?w=376&h=236&f=png&s=6696)

如上图，红色的 div 在绿色的 div 内部，且设置了`margin-top`为 30px，但我们发现红色 div 的顶部与绿色 div 顶部重合，并没有距离顶部 30px，而是溢出到父元素的外面计算。即本来父元素距离顶部只有 20px，被子元素溢出影响，外边距重叠，取较大的值，则距离顶部 30px。

解决办法：

1. 给父元素触发 BFC（如添加`overflow: hidden`）
2. 给父元素添加 border
3. 给父元素添加 padding

这样就能实现我们期望的效果了：

![](https://user-gold-cdn.xitu.io/2020/7/4/1731a50faed50d4d?w=249&h=221&f=png&s=2040)

### 清除浮动解决令父元素高度坍塌的问题

当容器内子元素设置浮动时，脱离了文档流，容器中总父元素高度只有边框部分高度

```
/* html 代码 */
<div class="parent">
  <div class="child"></div>
</div>

/* css 代码 */
* {
    margin: 0;
    padding: 0;
}
.parent {
    border: 4px solid red;
}
.child {
    float: left;
    width: 200px;
    height: 200px;
    background-color: blue;
}
```

![](https://user-gold-cdn.xitu.io/2020/7/4/1731a132272ea410?w=1927&h=209&f=png&s=16771)

![](https://user-gold-cdn.xitu.io/2020/7/4/1731a249e4c9c33f?w=338&h=679&f=png&s=38121)

解决办法：给父元素触发 BFC，使其有 BFC 特性：**计算 BFC 的高度时，浮动元素也会参与计算**

```
.parent {
    overflow: hidden;
    border: 4px solid red;
}
```

![](https://user-gold-cdn.xitu.io/2020/7/4/1731a1fd01267b9d?w=1920&h=237&f=png&s=10195)

上面我们都是用的`overflow: hidden`触发 BFC，因为确实常用嘛，但是触发 BFC 也不止是只有这一种方法，如上面写的所示。

比如可以设置`float: left`; `float: right`; `display: inline-block`; `overflow: auto`; `display: flex`; `display:" table`; `position`为`absolute`或`fixed`等等，这些都可以触发，不过父元素宽度表现不一定相同，但父元素高度都被撑出来了。当然实际运用可不是随便挑一个走，还是根据场景选择。

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，鼓励我继续写作吧~

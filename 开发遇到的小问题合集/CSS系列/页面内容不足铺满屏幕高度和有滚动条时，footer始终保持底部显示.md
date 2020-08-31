# 页面内容不足铺满屏幕高度和有滚动条时，footer 始终保持底部显示

记录一个项目中经常会用到的技巧，footer 区（比如版权信息）要始终居于页面底部。如果用 fixed 定位显然不可取，因为要保证页面高度大于屏幕高度时，footer 区要跟随着页面滚动保持在底部，如下图：

![](https://user-gold-cdn.xitu.io/2019/12/7/16ee0e36c5c55fea?w=1737&h=899&f=png&s=23364)

![](https://user-gold-cdn.xitu.io/2019/12/7/16ee0e3a25e302bd?w=1577&h=903&f=png&s=27932)

- tip：以下两个方法仅适用于 footer 区高度固定的情况

## 方法一

页面结构：

```
<div class="wrapper">
    <div class="main-container">
        <div class="box">这是主要内容</div>
    </div>
    <footer>这是底部</footer>
</div>
```

- 关键的 css

```css
html,
body {
  height: 100%;
}
.wrapper {
  height: 100%;
}
.main-container {
  min-height: 100%;
  box-sizing: border-box;
  padding-bottom: 40px;
}
footer {
  margin-top: -40px;
}
```

如上代码，主要内容区和 footer 区必须为并列关系。设置 main-container 为 border-box，即元素的内边距和边框都在已设定的高度内绘制，我们设置了 min-height 为 100%，即最小高度是铺满整屏幕。padding-bottom 在 border-box 盒子中，留出高度 40px 的空白块。footer 的 margin-top 为-40px（如果 footer 有设置高度，padding-bottom 的值应以 footer 高度为准），即向上拉到空白块内，main-container 最小高度为整个屏幕，故页面内容不足也可以显示在底部。

## 方法二

页面结构

```
<div class="wrapper">
    <div class="main-container">
        <div class="box">这是主要内容</div>
    </div>
    <footer>这是底部</footer>
</div>
```

关键的 css

```css
html,
body {
  height: 100%;
}
.wrapper {
  min-height: 100%;
  position: relative;
}

.main-container {
  padding-bottom: 40px;
}

footer {
  position: absolute;
  bottom: 0;
  height: 40px;
}
```

如上代码时用绝对定位，相对于 wrapper 居于底部，跟第一个方法有异曲同工之妙。

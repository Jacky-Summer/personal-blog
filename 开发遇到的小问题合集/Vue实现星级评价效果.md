# Vue 实现星级评价效果

我们把星级评价单独做成一个 Star 组件，抽离出来，其中父组件中引入(传入的是评分的值)

```html
<div class="score">
  <Star :score="poiInfo.wm_poi_score"></Star>
</div>
```

初始 Star.vue

```html
<template>
  <div>
    <div class="star">
      <span class="star-item"></span>
    </div>
    <span>{{ score }}</span>
  </div>
</template>
<script>
  export default {
    name: 'Star',
    props: {
      score: Number,
    },
  }
</script>
```

首先我们把要做星级图片用类名为 star-item 的 span 标签循环出来，星级图片有三张，全星，半星，空星
![](https://user-gold-cdn.xitu.io/2019/12/29/16f51be56d1a5ea8?w=20&h=19&f=png&s=1215)
![](https://user-gold-cdn.xitu.io/2019/12/29/16f51bdf50383e0f?w=20&h=19&f=png&s=1349)
![](https://user-gold-cdn.xitu.io/2019/12/29/16f51be27b49b42d?w=20&h=19&f=png&s=1220)

下面只罗列关键的 css 部分, 通过增加类区分图片

```css
.star-item.on {
  background-image: url(./img/star24_on@2x.png);
}
.star-item.half {
  background-image: url(./img/star24_half@2x.png);
}
.star-item.off {
  background-image: url(./img/star24_off@2x.png);
}
```

接下来修改 Star.vue 的代码

```html
<div class="star">
  <span
    class="star-item"
    v-for="(itemClass, index) in itemClasses"
    :key="index"
    :class="itemClass"
  >
  </span>
</div>
```

itemClasses 值是通过计算属性获取的，思路：

1. 通过 computed 返回一个长度为 5 的数组（显示 5 颗星）
2. 数组的值是上述 css 取的不同星对应的类名，再通过绑定每一个循环添加的 class，从而遍历星级。

比如举例评分：

- 4.7 分对应的数组为['on', 'on', 'on', 'on', 'half']
- 3.4 分对应的数组为['on', 'on', 'on', 'half', 'half']

JS 部分的代码：

```javascript
// 星星长度
const LENGTH = 5

// 星星的状态
const CLS_ON = 'on'
const CLS_HALF = 'half'
const CLS_OFF = 'off'

export default {
  name: 'Star',
  props: {
    score: Number,
  },
  computed: {
    itemClasses() {
      let result = []

      let score = Math.floor(this.score * 2) / 2

      // 半星 (通过跟1取余判断是否为小数)
      let hasDecimal = score % 1 !== 0

      // 全星 （向下取整，获取全星部分）
      let integer = Math.floor(score)

      // 遍历全星
      for (let i = 0; i < integer; i++) {
        result.push(CLS_ON)
      }

      // 处理半星
      if (hasDecimal) {
        result.push(CLS_HALF)
      }

      // 补齐
      while (result.length < LENGTH) {
        // 到这里还不够五颗星，则凑空星
        result.push(CLS_OFF)
      }

      return result
    },
  },
}
```

itemClasses 最终是返回了一个长度为 5 的数组，需要注意的是

```javascript
let score = Math.floor(this.score * 2) / 2
```

半星的划分：只有当小数第一位大于或等于 5 才可以算为半星，否则是空星。该计算是为了小数部分>=0.5 的计算结果带有小数，从而再后面跟 1 取余判断是否为半星。一开始有点蒙，多试几个数想想就懂了。

> 比如 4.3 分没有半星，4.5 有半星出现

结果：
比如传入的值为 4.7，则显示

![](https://user-gold-cdn.xitu.io/2019/12/29/16f51d7db002aa38?w=161&h=28&f=png&s=5690)

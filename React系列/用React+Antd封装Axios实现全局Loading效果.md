# 用 React+Antd 封装 Axios 实现全局 Loading 效果

## 前言

今天在做 react 后台管理的时候要实现一个全局 Loading 效果，通常使用 axios 库与后端进行数据交互。为了更好的用户体验，在每次请求前添加一个加载效果，让用户知道在等待加载。

要实现这个功能，我们可以在每个组件请求手动添加加载效果返回后再将其隐藏，但如果每个请求都这么做，就要做多次重复设置显得很麻烦，但好处是可以设置定制多种请求效果。但考虑到该项目场景为后台管理系统，给管理员使用，花样可以不用搞太多，统一优雅即可，故采取全局设置 loading 效果。

## 开发版本

```
"react": "^16.13.1",
"antd": "^4.0.4",
"axios": "^0.19.2"
```

## 代码说明

1. 通过 axios 提供的请求拦截和响应拦截的接口，控制 loading 的显示或者隐藏。在此我还设置了没有网络和网络超时的提示信息
2. 采用 antd 的 Spin 组件来实现 loading 效果，message 组件来进行消息提示（antd.css 这里没有引入，是因为我设置了按需加载）
3. 定义变量 requestCount 作为计数器，确保同一时刻如果有多个请求的话，不会同时添加多个 loading，而是只有 1 个，并在所有请求结束后才会隐藏 loading。
4. 默认所有请求都会自动有 loading 效果。如果某个请求不需要 loading 效果，可以在请求 headers 中设置 isLoading 为 false。

## 步骤

1. 在 src 目录下新建一个文件 axios.js

```
import axios from 'axios';
import React, { Component } from 'react';
import ReactDOM from 'react-dom';
import { message, Spin } from 'antd';

const Axios = axios.create({
    // baseURL: process.env.BASE_URL, // 设置请求的base url
    timeout: 20000, // 设置超时时长
})

// 设置post请求头
Axios.defaults.headers.post['Content-Type'] = 'application/x-www-form-urlencoded;charset=UTF-8'

// 当前正在请求的数量
let requestCount = 0

// 显示loading
function showLoading () {
    if (requestCount === 0) {
        var dom = document.createElement('div')
        dom.setAttribute('id', 'loading')
        document.body.appendChild(dom)
        ReactDOM.render(<Spin tip="加载中..." size="large"/>, dom)
    }
    requestCount++
}

// 隐藏loading
function hideLoading () {
    requestCount--
    if (requestCount === 0) {
        document.body.removeChild(document.getElementById('loading'))
    }
}

// 请求前拦截
Axios.interceptors.request.use(config => {
   // requestCount为0，才创建loading, 避免重复创建
    if (config.headers.isLoading !== false) {
        showLoading()
    }
    return config
}, err => {
    // 判断当前请求是否设置了不显示Loading
    if (err.config.headers.isLoading !== false) {
        hideLoading()
    }
    return Promise.reject(err)
})

// 返回后拦截
Axios.interceptors.response.use(res => {
    // 判断当前请求是否设置了不显示Loading
    if (res.config.headers.isLoading !== false) {
        hideLoading()
    }
    return res
}, err => {
    if (err.config.headers.isLoading !== false) {
        hideLoading()
    }
    if (err.message === 'Network Error') {
        message.warning('网络连接异常！')
    }
    if (err.code === 'ECONNABORTED') {
        message.warning('请求超时，请重试')
    }
    return Promise.reject(err)
})

// 把组件引入，并定义成原型属性方便使用
Component.prototype.$axios = Axios

export default Axios
```

2. 添加 loading 样式在共用的 css 文件里

```css
#loading {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.75);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
  font-size: 20px;
}
```

3. axios 请求

```
// 1. 引入自定义axios文件路径
// 2. 引入共用css文件（loading样式）
// 3. 在react组件中正常写法请求url即可
componentDidMount () {
    axios({
      url: '/manage/statistic/base_count.do'
    }).then(res => {
      this.setState(res.data)
    })
}
```

不加 loading 效果，这样写

```
axios({
  url: '/manage/statistic/base_count.do',
  headers: {
    'isLoading': false
  }
}).then(res => {
  this.setState(res.data)
})
```

## 效果

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/3/29/17126d31a0fc0571~tplv-t2oaga2asx-watermark.awebp)

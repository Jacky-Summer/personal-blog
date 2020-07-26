## 前言

最近做项目的时候，涉及到一个单点登录，即是项目的登录页面，用的是公司共用的一个登录页面，在该页面统一处理逻辑。最终实现用户只需登录一次，就可以以登录状态访问公司旗下的所有网站。

> 单点登录( Single Sign On ，简称 SSO），是目前比较流行的企业业务整合的解决方案之一，用于多个应用系统间，用户只需要登录一次就可以访问所有相互信任的应用系统。

其中本文讲的是在登录后如何管理`access_token`和`refresh_token`，主要就是封装 axios拦截器，在此记录。

## 需求

- 前置场景

1. 进入该项目某个页面`http://xxxx.project.com/profile`需要登录，未登录就跳转至SSO登录平台，此时的登录网址 url为`http://xxxxx.com/login?app_id=project_name_id&redirect_url=http://xxxx.project.com/profile`，其中`app_id`是后台那边约定定义好的，`redirect_url`是成功授权后指定的回调地址。

2. 输入账号密码且正确后，就会重定向回刚开始进入的页面，并在地址栏带一个参数 `?code=XXXXX`，即是`http://xxxx.project.com/profile?code=XXXXXX`，code的值是使用一次后即无效，且10分钟内过期

3. 立马获取这个code值再去请求一个api `/access_token/authenticate`，携带参数`{ verify_code: code }`，并且该api已经自带`app_id`和`app_secret`两个固定值参数，通过它去请求授权的api，请求成功后得到返回值`{ access_token: "xxxxxxx", refresh_token: "xxxxxxxx", expires_in: xxxxxxxx }`，存下`access_token`和`refresh_token`到cookie中（localStorage也可以），此时用户就算登录成功了。

4. `access_token`为标准JWT格式，是授权令牌，可以理解就是验证用户身份的，是应用在调用api访问和修改用户数据必须传入的参数（放在请求头headers里），2小时后过期。也就是说，做完前三步后，你可以调用需要用户登录才能使用的api；但是假如你什么都不操作，静静过去两个小时后，再去请求这些api，就会报`access_token`过期，调用失败。

5. 那么总不能2小时后就让用户退出登录吧，解决方法就是两小时后拿着过期的`access_token`和`refresh_token`（`refresh_token`过期时间一般长一些，比如一个月或更长）去请求`/refresh` api，返回结果为`{ access_token: "xxxxx", expires_in: xxxxx }`，换取新的`access_token`，新的`access_token`过期时间也是2小时，并重新存到cookie，循环往复继续保持登录调用用户api了。`refresh_token`在限定过期时间内（比如一周或一个月等），下次就可以继续换取新的`access_token`，但过了限定时间，就算真正意义过期了，也就要重新输入账号密码来登录了。

公司网站登录过期时间都只有两小时（token过期时间），但又想让一个月内经常活跃的用户不再次登录，于是才有这样需求，避免了用户再次输入账号密码登录。

为什么要专门用一个 `refresh_token` 去更新 `access_token` 呢？首先`access_token`会关联一定的用户权限，如果用户授权更改了，这个`access_token`也是需要被刷新以关联新的权限的，如果没有 `refresh_token`，也可以刷新 `access_token`，但每次刷新都要用户输入登录用户名与密码，多麻烦。有了 `refresh_ token`，可以减少这个麻烦，客户端直接用 `refresh_token` 去更新 `access_token`，无需用户进行额外的操作。

说了这么多，或许有人会吐槽，一个登录用`access_token`就行了还要加个`refresh_token`搞得这么麻烦，或者有的公司`refresh_token`是后台包办的并不需要前端处理。但是，前置场景在那了，需求都是基于该场景下的。

- 需求

1. 当`access_token`过期的时候，要用`refresh_token`去请求获取新的`access_token`，前端需要做到用户无感知的刷新`access_token`。比如用户发起一个请求时，如果判断`access_token`已经过期，那么就先要去调用刷新token接口拿到新的`access_token`，再重新发起用户请求。

2. 如果同时发起多个用户请求，第一个用户请求去调用刷新token接口，当接口还没返回时，其余的用户请求也依旧发起了刷新token接口请求，就会导致多个请求，这些请求如何处理，就是我们本文的内容了。

## 思路

### 方案一

写在请求拦截器里，在请求前，先利用最初请求返回的字段`expires_in`字段来判断`access_token`是否已经过期，若已过期，则将请求挂起，先刷新`access_token`后再继续请求。

- 优点： 能节省http请求
- 缺点： 因为使用了本地时间判断，若本地时间被篡改，有校验失败的风险

### 方案二

写在响应拦截器里，拦截返回后的数据。先发起用户请求，如果接口返回`access_token`过期，先刷新`access_token`，再进行一次重试。

- 优点：无需判断时间
- 缺点： 会消耗多一次http请求

在此我选择的是方案二。

## 实现

这里使用axios，其中做的是请求后拦截，所以用到的是axios的响应拦截器`axios.interceptors.response.use()`方法

### 方法介绍

- @utils/auth.js
```javascript
import Cookies from 'js-cookie'

const TOKEN_KEY = 'access_token'
const REGRESH_TOKEN_KEY = 'refresh_token'

export const getToken = () => Cookies.get(TOKEN_KEY)

export const setToken = (token, params = {}) => {
  Cookies.set(TOKEN_KEY, token, params)
}

export const setRefreshToken = (token) => {
  Cookies.set(REGRESH_TOKEN_KEY, token)
}
```

- request.js
```javascript
import axios from 'axios'
import { getToken, setToken, getRefreshToken } from '@utils/auth'

// 刷新 access_token 的接口
const refreshToken = () => {
  return instance.post('/auth/refresh', { refresh_token: getRefreshToken() }, true)
}

// 创建 axios 实例
const instance = axios.create({
  baseURL:  process.env.GATSBY_API_URL,
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  }
})

instance.interceptors.response.use(response => {
    return response
}, error => {
    if (!error.response) {
        return Promise.reject(error)
    }
    // token 过期或无效，返回 401 状态码，在此处理逻辑
    return Promise.reject(error)
})

// 给请求头添加 access_token
const setHeaderToken = (isNeedToken) => {
  const accessToken = isNeedToken ? getToken() : null
  if (isNeedToken) { // api 请求需要携带 access_token 
    if (!accessToken) { 
      console.log('不存在 access_token 则跳转回登录页')
    }
    instance.defaults.headers.common.Authorization = `Bearer ${accessToken}`
  }
}

// 有些 api 并不需要用户授权使用，则不携带 access_token；默认不携带，需要传则设置第三个参数为 true
export const get = (url, params = {}, isNeedToken = false) => {
  setHeaderToken(isNeedToken)
  return instance({
    method: 'get',
    url,
    params,
  })
}

export const post = (url, params = {}, isNeedToken = false) => {
  setHeaderToken(isNeedToken)
  return instance({
    method: 'post',
    url,
    data: params,
  })
}
```

接下来改造 request.js中axios的响应拦截器
```
instance.interceptors.response.use(response => {
    return response
}, error => {
    if (!error.response) {
        return Promise.reject(error)
    }
    if (error.response.status === 401) {
        const { config } = error
        return refreshToken().then(res=> {
            const { access_token } = res.data
            setToken(access_token)
            config.headers.Authorization = `Bearer ${access_token}`
            return instance(config)
        }).catch(err => {
            console.log('抱歉，您的登录状态已失效，请重新登录！')
            return Promise.reject(err)
        })
    }
    return Promise.reject(error)
})
```
约定返回401状态码表示`access_token`过期或者无效，如果用户发起一个请求后返回结果是`access_token`过期，则请求刷新`access_token`的接口。请求成功则进入`then`里面，重置配置，并刷新`access_token`并重新发起原来的请求。

但如果`refresh_token`也过期了，则请求也是返回401。此时调试会发现函数进不到`refreshToken()`的`catch`里面，那是因为`refreshToken()`方法内部是也是用了同个`instance`实例，重复响应拦截器401的处理逻辑，但该函数本身就是刷新`access_token`，故需要把该接口排除掉，即：
```javascript
if (error.response.status === 401 && !error.config.url.includes('/auth/refresh')) {}
```
上述代码就已经实现了无感刷新`access_token`了，当`access_token`没过期，正常返回；过期时，则axios内部进行了一次刷新token的操作，再重新发起原来的请求。

## 优化

### 防止多次刷新 token

如果token是过期的，那请求刷新`access_token`的接口返回也是有一定时间间隔，如果此时还有其他请求发过来，就会再执行一次刷新`access_token`的接口，就会导致多次刷新`access_token`。因此，我们需要做一个判断，定义一个标记判断当前是否处于刷新`access_token`的状态，如果处在刷新状态则不再允许其他请求调用该接口。
```javascript
let isRefreshing = false // 标记是否正在刷新 token
instance.interceptors.response.use(response => {
    return response
}, error => {
    if (!error.response) {
        return Promise.reject(error)
    }
    if (error.response.status === 401) {
        const { config } = error
        if (!isRefreshing) {
            isRefreshing = true
            return refreshToken().then(res=> {
                const { access_token } = res.data
                setToken(access_token)
                config.headers.Authorization = `Bearer ${access_token}`
                return instance(config)
            }).catch(err => {
                console.log('抱歉，您的登录状态已失效，请重新登录！')
                return Promise.reject(err)
            }).finally(() => {
                isRefreshing = false
            })
        }
    }
    return Promise.reject(error)
})
```

### 同时发起多个请求的处理

上面做法还不够，因为如果同时发起多个请求，在token过期的情况，第一个请求进入刷新token方法，则其他请求进去没有做任何逻辑处理，单纯返回失败，最终只执行了第一个请求，这显然不合理。

比如同时发起三个请求，第一个请求进入刷新token的流程，第二个和第三个请求需要存起来，等到token更新后再重新发起请求。

在此，我们定义一个数组`requests`，用来保存处于等待的请求，之后返回一个`Promise`，只要不调用`resolve`方法，该请求就会处于等待状态，则可以知道其实数组存的是函数；等到token更新完毕，则通过数组循环执行函数，即逐个执行resolve重发请求。

```javascript
let isRefreshing = false // 标记是否正在刷新 token
let requests = [] // 存储待重发请求的数组

instance.interceptors.response.use(response => {
    return response
}, error => {
    if (!error.response) {
        return Promise.reject(error)
    }
    if (error.response.status === 401) {
        const { config } = error
        if (!isRefreshing) {
            isRefreshing = true
            return refreshToken().then(res=> {
                const { access_token } = res.data
                setToken(access_token)
                config.headers.Authorization = `Bearer ${access_token}`
                // token 刷新后将数组的方法重新执行
                requests.forEach((cb) => cb(access_token))
                requests = [] // 重新请求完清空
                return instance(config)
            }).catch(err => {
                console.log('抱歉，您的登录状态已失效，请重新登录！')
                return Promise.reject(err)
            }).finally(() => {
                isRefreshing = false
            })
        } else {
            // 返回未执行 resolve 的 Promise
            return new Promise(resolve => {
                // 用函数形式将 resolve 存入，等待刷新后再执行
                requests.push(token => {
                    config.headers.Authorization = `Bearer ${token}`
                    resolve(instance(config))
                })  
            })
        }
    }
    return Promise.reject(error)
})
```

最终 request.js 代码
```javascript
import axios from 'axios'
import { getToken, setToken, getRefreshToken } from '@utils/auth'

// 刷新 access_token 的接口
const refreshToken = () => {
  return instance.post('/auth/refresh', { refresh_token: getRefreshToken() }, true)
}

// 创建 axios 实例
const instance = axios.create({
  baseURL:  process.env.GATSBY_API_URL,
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  }
})

let isRefreshing = false // 标记是否正在刷新 token
let requests = [] // 存储待重发请求的数组

instance.interceptors.response.use(response => {
    return response
}, error => {
    if (!error.response) {
        return Promise.reject(error)
    }
    if (error.response.status === 401) {
        const { config } = error
        if (!isRefreshing) {
            isRefreshing = true
            return refreshToken().then(res=> {
                const { access_token } = res.data
                setToken(access_token)
                config.headers.Authorization = `Bearer ${access_token}`
                // token 刷新后将数组的方法重新执行
                requests.forEach((cb) => cb(access_token))
                requests = [] // 重新请求完清空
                return instance(config)
            }).catch(err => {
                console.log('抱歉，您的登录状态已失效，请重新登录！')
                return Promise.reject(err)
            }).finally(() => {
                isRefreshing = false
            })
        } else {
            // 返回未执行 resolve 的 Promise
            return new Promise(resolve => {
                // 用函数形式将 resolve 存入，等待刷新后再执行
                requests.push(token => {
                    config.headers.Authorization = `Bearer ${token}`
                    resolve(instance(config))
                })  
            })
        }
    }
    return Promise.reject(error)
})

// 给请求头添加 access_token
const setHeaderToken = (isNeedToken) => {
  const accessToken = isNeedToken ? getToken() : null
  if (isNeedToken) { // api 请求需要携带 access_token 
    if (!accessToken) { 
      console.log('不存在 access_token 则跳转回登录页')
    }
    instance.defaults.headers.common.Authorization = `Bearer ${accessToken}`
  }
}

// 有些 api 并不需要用户授权使用，则无需携带 access_token；默认不携带，需要传则设置第三个参数为 true
export const get = (url, params = {}, isNeedToken = false) => {
  setHeaderToken(isNeedToken)
  return instance({
    method: 'get',
    url,
    params,
  })
}

export const post = (url, params = {}, isNeedToken = false) => {
  setHeaderToken(isNeedToken)
  return instance({
    method: 'post',
    url,
    data: params,
  })
}
```

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎star，给我一点鼓励继续写作吧~
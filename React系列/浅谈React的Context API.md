# 浅谈React的Context API

## 前言

上下文(Context) 提供了在组件之间共享这些值的方法，而不必在树的每个层级显式传递一个 prop 。

一般情况下，在你没有绝对把握用好和必须的场景下，是不推荐使用的。

但是我们依然间接的使用着它，比如许多官方依赖在使用，如：react-redux, mobx-react，react-router。我们需要它功能的时候，更多是靠第三方依赖库就能实现，而不是自己手动写context。但是，依然需要理解它，对用它的一些依赖库源码理解很有帮助。

```javascript
import React, { Component } from 'react';

class Grandpa extends Component {
    state = {
        info: 'GrandPa组件的信息'
    }
    render() {
        return (
            <div>
                <Father info={this.state.info}/>
            </div>
        );
    }
}

class Father extends Component {
    render() {
        return (
            <div>
                <Son info={this.props.info}/>
            </div>
        );
    }
}

class Son extends Component {
    render() {
        return (
            <div>
                我是Son组件，拿到信息为：{this.props.info}
            </div>
        );
    }
}

export default Grandpa;
```
最后页面输出：
```
我是Son组件，拿到信息为：GrandPa组件的信息
```
我们会发现这样的缺点是一层一层传值，如果有更多层级和更多数据的话，会让代码看起来很不整洁，如果中间哪个组件忘了传值基本就完了；而且Father组件也并没有用到info值，只是将值传给Son组件。

如果使用context，就能帮我们解决这个层级不停传值的问题。

context有旧版和新版之分，以React v16.3.0版本划分。

## 旧版Context API

我们先来说下旧版context API

将代码改成如下：
```javascript
import React, { Component } from 'react';
import PropTypes from 'prop-types';

class Grandpa extends Component {
    state = {
        info: 'GrandPa组件的信息'
    }
    getChildContext () {
        return { info: this.state.info }
    }
    render() {
        return (
            <div>
                <Father/>
            </div>
        );
    }
}
// 声明Context对象属性
Grandpa.childContextTypes  = {
    info: PropTypes.string
}

class Father extends Component {
    render() {
        return (
            <div>
                <Son/>
            </div>
        );
    }
}

class Son extends Component {
    render() {
        return (
            <div>
                我是Son组件，拿到信息为：{this.context.info}
            </div>
        );
    }
}
// 根据约定好的参数类型声明
Son.contextTypes  = {
    info: PropTypes.string
}
export default Grandpa;
```
对PropTypes类型检查，还可以写成下面这种写法
```javascript
class Grandpa extends Component {
    static childContextTypes = {
        info: PropTypes.string
    }
    state = {
        info: 'GrandPa组件的信息'
    }
    getChildContext () {
        return { info: this.state.info }
    }
    render() {
        return (
            <div>
                <Father/>
            </div>
        );
    }
}
```
在需要传出去的context值和需要接收context值都要进行类型检查判断。

正常用这个api可能没什么问题，但如果中间哪个组件用到`shouldComponentUpdate`方法的话，就很可能出现问题。

三个组件的层级关系如下
```javascript
<GrandPa>
    <Father>
        <Son/>
    </Father>
</GrandPa>
```
如果在GrandPa组件设置按钮点击可以更新info的值，即通过`this.setState({info: '改变值'})`方法
更新 Context：
那么

- GrandPa组件通过 setState 设置新的 Context 值同时触发子组件重新render。
- Father组件重新render。
- Son组件重新render，并拿到更新后的Context。

假如在Father组件添加函数
```javascript
shouldComponentUpdate () {
    return false
}
```
由于Father并不依赖任何值，所以我们默认让它无需重新render。但是，这会导致Son组件也不会重新render，即无法获取到最新的 Context 值。

这样的不确定性对于目标组件来说是完全不可控的，也就是说目标组件无法保证自己每一次都可以接收到更新后的 Context 值。这是旧版API存在的一个大问题。

而新版API解决了这个问题，我们来看下新版API怎么写的

## 新版Context API

> 新版Context 新增了 creactContext() 方法用来创建一个context对象。这个对象包含两个组件，一个是 Provider（生产者），另一个是 Consumer（消费者）。

1. Provider 和 Consumer 必须来自同一个Context对象，即一个由 React.createContext()创建的Context对象。
2. React.createContext()方法接收一个参数做默认值，当 Consumer 外层没有对应的 Provider 时就会使用该默认值。
3. Provider 组件使用Object.is方法判断prop值是否发生改变，当prop的值改变时，其内部组件树中对应的 Consumer 组件会接收到新值并重新渲染。此过程不受 shouldComponentUpdete 方法的影响。
4. Consumer 组件接收一个函数作为 children prop 并利用该函数的返回值生成组件树的模式被称为 Render Props 模式。

### 用法

1. 首先创建一个context对象

> React.createContext 方法用于创建一个 Context 对象。该对象包含 Provider 和 Consumer两个属性，分别为两个 React 组件。

```javascript
const InfoContext = React.createContext('');
```
2. 使用Provider组件生成一个context
```javascript
class Grandpa extends Component {
    state = {
        info: 'GrandPa组件的信息'
    }
    render() {
        return (
            <InfoContext.Provider value={{ info: this.state.info }}>
                <Father/>
            </InfoContext.Provider>
        );
    }
}
```
3. 使用Consumer组件获取context对象的值
```javascript
class Son extends Component {
    render() {
        return (
            <InfoContext.Consumer>
            {
                value => {
                    return <div>我是Son组件，拿到信息为：{value.info}</div>
                }
            }
            </InfoContext.Consumer>
        );
    }
}
```

<br>

- ps： [个人技术博文Github仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎star，给我一点鼓励吧~

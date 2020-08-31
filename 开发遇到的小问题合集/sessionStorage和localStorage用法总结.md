# sessionStorage 和 localStorage 用法总结

在工作中使用 sessionStorage 存储数据时，发现 sessionStorage 无法直接存储数组和对象，如存入对象则显示为"[object Object]"，对此作下记录，重新温习 sessionStorage 和 localStorage

html5 中的 web Storage 包括了两种存储方式：sessionStorage 和 localStorage

## 共同点

存储大小为 5MB，都保存在客户端，不与服务器进行交互通信，有相同的 Web API

## sessionStorage、localStorage 区别

localStorage 存储持久数据，浏览器关闭后数据不丢失除非主动删除数据；

sessionStorage 数据在当前浏览器窗口关闭后自动删除。

因此**sessionStorage 和 localStorage 的主要区别在于他们存储数据的生命周期**，sessionStorage 存储的数据的生命周期是一个会话，而 localStorage 存储的数据的生命周期是永久，除非主动删除数据，否则永远不会过期

## Web Storage API

localStorage 和 sessionStorage 有着统一的 API 接口，下面以 sessionStorage 为例介绍一下 API 接口使用方法

### 添加键值对

**setItem(key,value)：为指定 key 值设置一个对应的 value 值**

除了使用 setItem 方法，还可以使用 sessionStorage.key = value 或者 sessionStorage['key'] = value 这两种形式。

```javascript
// 把name值存储到name的键上
sessionStorage.setItem('name', 'jacky') // 法1
// sessionStorage.name = 'jacky'; // 法2
// sessionStorage['name'] = 'jacky'; // 法3
```

**添加数组和对象**

需要注意的是 key 和 value 值必须是**字符串形式**的，如果不是字符串，会调用它们相应的 toString()方法来转换成字符串再存储。

所以要存储数组或对象时，应先转换成字符串格式（如 JSON 格式）再进行存储，使用**JSON.stringify(obj)方法**

```javascript
var obj = {
  name: 'jacky',
  age: 22,
}
sessionStorage['person'] = JSON.stringify(obj)
//sessionStorage['person'] = obj; 不能这样存储，这样存进去结果是"[object Object]"
```

存进去之后则为字符串格式

```javascript
 "{"name":"jacky","age":22}"
```

需要拿出来使用的时候则使用**JSON.parse()方法**将 JSON 字符串转换为对象

```javascript
var person = JSON.parse(sessionStorage['person'])
```

同理，数组也是这个用这个方法进行存储。

### getItem（key）：根据指定的 key 值获取对应的 value 值

和 setItem 一样，getItem 也有两种等效形式,value = sessionStorage.key 和 value = sessionStorage['key']。获取到的 value 值是字符串类型，如果需要其他类型，需要自己做类型转换。

```javascript
// 获取存储到 name 的键上的值
var name = sessionStorage.getItem('name')
// var name = sessionStorage.name;
// var name = sessionStorage['name'];
```

### removeItem（key）：删除指定的 key 值对应的 value 值

注意 localStorage 没有数据过期的概念，所有数据如果失效了，需要开发者手动删除。

```javascript
var name = sessionStorage.getItem('name') // "jacky"
sessionStorage.removeItem('name')
name = sessionStorage.getItem('name') // null
```

### clear()：删除所有存储的内容

它和 removeItem 不同的地方是 removeItem 删除的是某一项，而 clear 是删除所有。

```javascript
// 清除 localStorage
sessionStorage.clear()
var len = sessionStorage.length // 0
//length属性用于获取 sessionStorage 中键值对的数量。
```

### key(index)：在指定的数字位置获取该位置的名字

需要注意的是赋值早的键值对应的索引值大，赋值完的键值对应的索引小,key 方法可用于遍历 sessionStorage 存储的键值。

```javascript
sessionStorage.setItem('name', 'jacky')
var key = sessionStorage.key(0) // 'name'
sessionStorage.setItem('age', 10)
key = sessionStorage.key(0) // 'age'
key = sessionStorage.key(1) // 'name'
```

## 两者应用场景

### sessionStorage 应用场景

进行页面传值

### localStorage 应用场景

1. 适合长期保存在本地的数据
2. 可以用于存储该浏览器对该页面的访问次数

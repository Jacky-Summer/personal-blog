## 前言

JS 中有三个可以对字符串编码的函数，分别是： escape,encodeURI,encodeURIComponent

## escape()

通常用于对字符串编码，不适用于对 URL 编码

除了 ASCII 字母、数字和特定的符号外，对传进来的字符串全部进行转义编码，因此如果想对 URL 编码，最好不要使用此方法。

escape 不会编码的字符有 69 个：`* + - . / @ _ 0-9 a-z A-Z`

当然如果没有必要，不要使用 escape。

## encodeURI()

`encodeURI()`不会进行编码的字符有 82 个 ：` ; , / ? : @ & = + $ - _ . ! ~ * ' ( ) # 0-9 a-z A-Z`

使用`encodeURI()`编码后的结果是除了空格之外的其他字符都原封不动，只有空格被替换成了`%20`,`encodeURI`主要用于直接赋值给地址栏。

## encodeURIComponent()

`encodeURIComponent`:不会进行编码的字符有 71 个：`! ' ( ) * - . _ ~ 0-9 a-z A-Z`

`encodeURIComponent`比`encodeURI`编码的范围更大，如`encodeURIComponent`会把 `http://` 编码成 `http%3A%2F%2F` 而`encodeURI`不解码维持`http://`

`encodeURIComponent()` 方法在编码单个 URIComponent（指请求参数）应当是最常用的，它可以将参数中的中文、特殊字符进行转义，而不会影响整个 URL

## 如何选择和使用三个函数

- 如果只是编码**字符串**，和 URL 没有关系，才可以用`escape`。（但它已经被废弃，尽量避免使用，应用`encodeURI` 或 `encodeURIComponent`）
- 如果需要编码**整个 URL**，然后需要使用这个 URL，那么用`encodeURI`
- 如果需要编码 **URL 中的参数**的时候，那么`encodeURIComponent`是最好方法。

```
encodeURI("https://github.com/Jacky-Summer/test params");
```

编码后变成

```
https://github.com/Jacky-Summer/test%20params
```

其中空格被编码成了`%20`，而如果是用`encodeURIComponent`

```
https%3A%2F%2Fgithub.com%2FJacky-Summer%2Ftest%20params
```

连 `/` 都被编码了，整个 URL 已经没法用了。

当编码 URL 的特殊参数时：

```
// 参数的 / 是需要编码的，而如果是 encodeURI 编码则不编码 / 就会出问题
let param = "https://github.com/Jacky-Summer/";
param = encodeURIComponent(param);
const url = "https://github.com?param=" + param;
console.log(url) // "https://github.com?param=https%3A%2F%2Fgithub.com%2FJacky-Summer%2F"
```

参考：https://www.zhihu.com/question/21861899

<br>

- ps： [个人技术博文 Github 仓库](https://github.com/Jacky-Summer/personal-blog)，觉得不错的话欢迎 star，给我一点鼓励继续写作吧~

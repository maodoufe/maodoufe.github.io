---
layout: post
title: JavaScript函数之参数解析
categories: [Function, javascript]
tags: [Function, javascript]
fullview: true
comments: true
---

作者：一介书生@[毛豆前端](https://maodoufe.github.io/)

在JavaScript世界中函数是一等公民，它不仅拥有一切传统函数的使用方式（声明和调用），而且可以做到像简单值一样赋值、传参、返回，这样的函数也称之为第一级函数（First-class Function）。不仅如此，JavaScript中的函数还充当了类的构造函数的作用，同时又是一个Function类的实例(instance)。这样的多重身份让JavaScript的函数变得非常重要。本次讲一下有关函数参数的细节知识。


## 函数形参的默认值
es5中模拟默认参数：

```javascript
function makeRequest(url, time, callback) {
    time = (typeof time !== 'undefined') ? time : 2000;
    callback = (typeof callback !== "undefined") ? callback : function () {
        //
    }
}
```

在这个函数中timeout,callback为可选参数，如果不传系统会给他们赋予默认值。这种写法虽然很严谨，但是需要额外的代码。  
es6中模拟默认参数：

```javascript
function makeRequest(url, time=2000, callback=function () {}) {
  //
}
```

在es6的标准中，只有url是必传参数，其余参数都是可选参数。如下调用都可以生效。

```javascript
makeRequest('/foo)
makeRequest('/foo, 500)
makeRequest('/foo, 500, function() {//...})
```

### 默认参数对arguments对象的影响

```javascript
function mixArgs(first, second) {
    console.log(first === arguments[0])
    console.log(second === arguments[1])
    first = 'c'
    second = 'd'
    console.log(first === arguments[0])
    console.log(second === arguments[1])
}
```
结果：

```javascript
true  
true  
true  
true
```
在非严格模式下，命名参数的变化会同步到arguments对象中，当first，second被赋予新值时arguments[0]，arguments[1]也跟着更新了。但是在严格模式下，情况变得不一样。

```javascript
function mixArgs(first, second) {
    'use strict'
    console.log(first === arguments[0])
    console.log(second === arguments[1])
    first = 'c'
    second = 'd'
    console.log(first === arguments[0])
    console.log(second === arguments[1])
}
mixArgs('a', 'b')
```
    
结果如下：

```javascript
true  
true  
false  
false
```
	
严格模式下，无论参数如何变化，arguments对象是不变的。  
在es6中，如果一个函数使用了默认参数值，则无论是否显式定义了严格模式，arguments对象的行为都将于es5的严格模式保持一致。

```javascript
function mixArgs(first, second='b') {
    console.log(arguments.length)
    console.log(first === arguments[0])
    console.log(second === arguments[1])
    first = 'c'
    second = 'd'
    console.log(first === arguments[0])
    console.log(second === arguments[1])
}
mixArgs('a')
```

结果如下：

```javascript
1  
true  
false
false
false
```
	
### 默认参数的临时死区

es6的let和const是存在临时死区TDZ，默认参数也存在同样的临时死区，在这里参数不可访问。与let声明类似，定义参数时会为每个参数创建一个新的标识符绑定，更改绑定在初始化时不可被访问。

```javascript
function add(first = second, second) {
	return first + second
}
add(1, 1)
add(undefined, 1)
```
	
add(1, 1)相当于执行以下代码：

```javascript
let first = 1
let second = 1
```

add(undefined, 1)相当于执行以下代码:

```javascript
let first = second
let second = 1
```
在初始化first时，second尚未初始化，所以导致程序报错。

## 不定参数

在函数的命名参数前加“…”就表明这是个不定参数，该参数作为一个数组，包含字它之后传入的所有参数。

```javascript
function pick(obj, ...keys) {
	let result = {}
	for (let i = 0; i < keys.length; i++) {
		result[keys[i]] = obj[keys[i]]
	}
	return result
}
```
    
这个函数模仿了Undescore.js的pick()方法，返回一个给定对象的副本。示例中只定义了一个参数是被复制的原始对象，其他参数为被复制属性的名称。

```javascript
var article = {
	title: 'js-函数',
	author: 'zh',
	age: '20',
}
console.log(pick(article, 'title', 'age'))
```

结果: 

```javascript
{title: "js-函数", age: "20"}
```

此时函数的length属性统计的是函数的命名参数的数量，不定参数加入不会影响length属性的值。示例中pick的length为1，因为它值计算obj。  

__不定参数的使用有两条限制：__

+ 1. 首先每个函数最多只能声明一个不定参数，不定参数的位置一定要放在所有参数的末尾。
+ 2. 不定参数不能用于对象字面量setter之中。如下写法是报错的。


```javascript
let object = {
	set a(...val) {
		//
    }
}
```

下节我们会讨论函数的箭头函数，下次再见。





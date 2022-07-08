---
layout:       post
title:        "javascript知识"
date:         2017-11-08 12:00:00
categories: document
tag: javascript
---

* content
{:toc}

### map
var f=function(x){
  return x*x
}
var array=[1,2,3,4,5,6,7]
var result=[]
for (var i=0;i<array.length;i++){
  result.push(f(array[i]))
}

### 浏览器对象

#### window

window对象不但可以当全局作用域，需且表示浏览器窗口。该对象有innerWidth和innerHeight属性，可以分别获取浏览器窗口的宽度和高度。内部完高是指除去菜单栏、工具栏、边框等占位元素后，用于显示网页的净宽高。对应的还有一个outerWidth和outerHeight属性，可以获取浏览器窗口的整个宽高度。
```
alert('window inner size: '+ window.innerWidth + 'x' + window.innerHeight);
```
#### navigator对象表示浏览器的信息，最常用的属性包括：
+ navigator.appName:浏览器的名称
+ navigator.appVersion:浏览器的版本
+ navigator.language:浏览器设置的语言
+ navigator.paltform:操作系统类型
+ navigator.userAgent:浏览器设定的User-Agent字符串

```
alert()
```

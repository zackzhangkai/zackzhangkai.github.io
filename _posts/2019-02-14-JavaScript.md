---
layout:         post
published:         true
title:          JavaScript关键特性
categories:         [前端]
tags:         [javascript]
---

* content
{:toc}

### 前言
昨天的关于JavaScript基础的笔记因没有保存，笔记本重启后全部没了，今后要记得如果是用atom，一定要明白ctrl+s的重要性，操作不谨慎、亲人两行泪！

### 内容
知识点：条件语句、循环语句、函数

#### 条件  
+ if...else...  
语法
```
if(条件) {
  //条件为true时执行的语句
}else {
  //条件为false时执行的语句
}
```
如：
```
if(3>2) {
  console.log('fuck you')
}else {
  console.log('fuck me!')
}
```
如果是多个条件
```
if(条件1) {
  //条件1执行的语句
}else if(条件2){
  //条件2执行的语句
}else {
  //前面的条件都为false时执行的语句
}
```

+ switch case  
语法：
```
switch(k)
{
case 1:
  执行代码块 1 ;
  break;
case 2:
  执行代码块 2 ;
  break;
default:
  默认执行（k 值没有在 case 中找到匹配时）;
}
```

#### 三元运算符
语法  
```
条件表达式?结果1:结果2
```
如
```
3>2?console.log("3比2大"):console.log("3比2小")
```

#### 循环
```
for (初始化;条件;增量){
         循环代码;
}
```
举个例子  
```
for(var i=1;i<=100;i++){
    console.log(i);
}
```

> 注意break和continue的使用，只针对循环语句，不针对判断语句；---稍等，，这个也不对，break是针对循环，直接跳出循环，移到循环语句之后，如switch case之后或for之后；而continue是不跳出循环，而跳出当前的循环，而进行下一个循环；

+ while和do while  
循环语句不仅有for还有while；while 循环，先判断再执行。do while 循环先执行一次再判断


#### 函数
+ join  
+ split  
+ Number  
+ ...

语法为：
```
function functionName(parameters) {
  //执行的代码
}
```
函数的两种表达形式
```
function f(x,y){
	return x+y
}
undefined
f(3,4)
7
add = function(x,y){
	return x+y
}
ƒ (x,y){
	return x+y
}
add(3,9)
12
```

函数表达式

```
//此处的代码执行没有问题，JavaScript解析器首先会把当前作用域的函数声明提前到整个作用域的最前面。
   f(2,3);
   function f(a,b) {
       console.log(a+b);
   }

```

函数声明
```
//报错：f is not a function
//这是因为函数表达式，如同定义其它基本类型的变量一样，只在执行到某一句时也会对其进行解析

    f(2,3);
    var f = function(a,b){
    console.log(a+b);
    }
```

函数的参数：形参、实参

#### 返回值

如果函数中没有 return 语句，那么函数默认的返回值是：undefined。
如果函数中有 return 语句，那么跟着 return 后面的值就是函数的返回值。
如果函数中有 return 语句，但是 return 后面没有任何值，那么函数的返回值也是：undefined。
函数在执行 return 语句后会停止并立即退出，也就是说 return 语句执行之后，剩下的代码都不会再执行了。
当函数外部需要使用函数内部的值的时候,我们不能直接给予，需要通过 return 返回
```
var f = function(a,b){
    a+b;
}
console.log(f(2,3)); // 结果为undefined
```

```
var f = function(a,b){
    return a+b;
}
console.log(f(2,3)); // 结果为5。
```

#### 匿名函数

匿名函数就是没有命名的函数，一般用在绑定事件的时候

语法

```
function(){

}

```
如
```
var myButton = document.querySelector('button');

myButton.onclick = function() {
  alert('hello');
}
```

>将匿名函数分配为变量的值，也就是我们前面所讲的函数表达式创建函数。一般来说，创建功能，我们使用函数声明来创建函数。使用匿名函数来运行负载的代码以响应事件触发（如点击按钮） ，使用事件处理程序。


#### 自调用函数

```
(function () {
  alert('hello');
})()

```

#### 实战
自定义一个函数，提示用户输入一个正整数。如果用户输入的非正整数，提示用户输入错误，并返回输入界面让用户输入。直到用户输入一个正确的正整数后
，在页面打印出一个由\*组成的直角三角形。第一行打印一个\* ，第二行打印两个\* ，以此类推下去，最后一行的 个数为用户输入的正整数。比如我们输入7，最后打效果如下：
```
*
**
****
*****
******
********
```
参考代码：


```html
<!DOCTYPE html>
<html>

    <head>
        <meta charset="UTF-8">
        <title></title>
    </head>

    <body>
        <script>
            function star(i) {
                for(var j = 1; j <= i; j++) {
                    for(var k = 1; k <= j; k++) {
                        document.write("*");
                    }
                    document.write("<br>");
                }
            }
            do {
                var n = prompt("请输入一个正整数");
                if(Number(n) > 0 && parseInt(n) == parseFloat(n)) {
                    star(n);
                } else {
                    alert("输入错误，请输入一个正整数");

                }
            }
            while (!(Number(n) > 0 && parseInt(n) == parseFloat(n)))
        </script>
    </body>

</html>
```

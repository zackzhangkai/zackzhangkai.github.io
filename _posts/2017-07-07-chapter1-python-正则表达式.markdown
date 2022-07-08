---
layout:       post
title:        "chapter1-python-正则表达式"
date:         2017-07-07 12:00:00
categories: project
tag: python
---

* content
{:toc}

## 特殊符号和字符

### 择一匹配 (|)
择一匹配(|)，表示一个“从多个模式中选择其一”的操作。它用于分割不同的正则表达式
```
at|home  -> at、home
r2d2|c3po  -> r2d2、c3po
bat|bet|bit -> bat、bet、bit
```
择一匹配有时候也叫作并(union)或者逻辑或(logical OR)

### 匹配任一单一字符
点号.匹配除了换行符\n以外的任何字符，即字母、数字、空格(不包含换行符\n)、可打印字符、不可打印字符、符号都可匹配
```
f.o  -> f和o之间任意字符，fao、f9o、f#o
..  -> 任意两个字符
.end  -> 匹配在end开头的任意一个字符
```
### 从字符串起始或者结尾或者单词边界匹配
+ 类似shell, ^匹配开头,$匹配末尾
+ \b和\B可以用来匹配字符边界，\b用于匹配一个单词的边界，同时\B将匹配出现在一个单词中间模式（即，不是单词边界）
```
the -> 任何包含the的字符串
\bthe -> 任何以the开始的字符串
\bthe\b -> the
\Bthe  -> 任何包含不以the作为起始的字符串
```
```
m = re.search('^The','The end.')
if m is not None: m.group()
>'The'
m = re.search('^The','end.The')
if m is not None: m.group()
>
m = re.search(r'\bthe','bite the dog')
if m is not None: m.group()
>'the'
m = re.search(r'\bthe','bitethe dog')
if m is not None: m.group()
>
m = re.search(r'\Bthe','bitethe dog')
if m is not None: m.group()
>'the'
```

### 创建字符集
尽管句点.能够匹配任一单一字符集，但某些时候想要匹配某些特定字符，此时可以用方括号[],与shell类似，匹配方括号中的任一字符
```
b[aeiu]t -> bat,bet,bit,but
[cr][23][dp][o2] -> c2do,r2do,c2do……
```
[]表示逻辑或的功能

### 限定范围和否定
除了单字符外，字符集还支持匹配指定的字符范围。方括号中间用-连接两个字符，指定其范围；如A-Z,a-z,0-9；如果^紧跟在左方括号后面，则表示不匹配给定字符中的任一字符，如[^aeiou]，[^\t\n]
### 使用闭包操作符实现存在性和频数匹配
+ * 匹配左边的正则表达示0次或多次
+ + 匹配左边的正则表达示1次或多次
+ ？ 匹配一次或0次
+ {} ： {M}:匹配M次，{M,N}:匹配M到N次
```
[dn]ot?  -> do、no、dot、not
0?[1-9] -> 001,01,009,09……
[0-9]{15,16} -> 匹配15或者16个数字，如信用卡号码
[^aeiou]  -> 不匹配给定字符中的任一字符
```
### 表示字符集的特殊字符
与使用0-9这个范围表示十进制数相比，可以简单地使用d表示匹配任何十进制数字。另一个特殊字符\w能够用于不表示全部字母数字的字符集，相当 于[A-Za-z0-9_]的缩写形式，\s可以用来表示空格字符。这些特殊字符的大写版表示不匹配；例如，\d表示匹配十进制，\D表示任何非十进制数，与[^0-9]相同。
```
\w+-|d+ -> 一个由字母数字组成的字符串和一串由一个连字符分隔的数字
[A-Za-z]\w* 第一个字母，其余字符（如果存在）是字母或数字
\d{3}-\d{3}-\d{4}  美国电话号码的的格式，前面是区号前缀，例如800-555-1212
\w+@\w+\.com  xxx@yyy.com
```
### 使用圆括号指定分组
对已经匹配过的字符串结果进行再匹配，用()
```
\d+(\.\d*)?  -> 表示简单浮点数的字符串：也就是说，任何十进制数字，后面可以接一个小数点和零个或者多个十进制数字，例如"0.004" "2" "75."
(Mr?s?)[A-Z][a-z]*[A-Za-z]+  -> 名字和姓氏，以及对名字的限制(如果有，首字母必须大写，后续字母小写)，全名前可以有可选的“Mr.”，“Mrs.”,"Ms."或者"M."作为称谓，以及灵活可选的姓氏，可以有多个单词、横线以及大写字母
```

## python与正则

### re
+ compile(pattern,flags = 0） ： re.compile()预编译，能提高执行性能。模块函数会对已编译的对象进行缓存，所以不是所有使用相同正则表达式模式的search()和match()都需要编译，purge()函数能够用于清除这些缓存。
+ match(pattern, string,flags=0)：尝试使用带有可选标记的正则表达式的模式来匹配字符串，如果匹配成功，返回匹配对象，若失败，返回None
```
m = re.match('foo','foo')
if m is not None:
  m.group()
-> 'foo'
```
+ search(pattern, string, flags=0)：使用可选标记搜索字符串中第一次出现的正则表达式模式，如果匹配成功，则返回匹配对象；如果失败，则返回None
```
m = re.match('food','seafood')
if m is not None:
  m.group()
->   #匹配失败
```
```
m = re.search('foo','seafood')
if m is not None:
  m.group()
->'foo' #搜索成功，匹配失败
```
+ findall(pattern,string[,flags]):查询某个字符串中某个正则表达式模式全部的非重复出现情况。这与search()在执行字符串搜索时类似，但与match()与search()不同之处在于它返回一个列表，成功则返回所有成功匹配的部份；若未找到匹配的部份，则返回空列表。
```
re.findall('car','car')
>['car']
re.findall('car','scary')
>['car']
re.findall('car','carry the barcardi to the car')
>['car','car','car']
```
+ finditer(pattern,string[,flags])
+ split(pattern,string,max=0)
+ sub(pattern,repl,string,count=0)
+ purge()
+ group(num=0) ：返回整个匹配对象，或者编号为num的特定子组
+ groups(default=None) ：返回一个包含所有匹配子组的元组（如果没成功匹配，则返回一个空元组）
+ groupdict(default=None): 返回字典，类似上
+ re.I、re.IGNORECASE ： 不区分大小写的匹配
+ re.L、reLOCALE:
+ re.M、re、MULTILINE
+ re.S、rer.DOTALL
+ re.X、re.VERBOSE

## 按要求创建正则表达式
+ 识别后续的字符串： “bat”、“bit”、“but”、“hat”、“hit”或者“hurt”
```
data = 'bat bit but hat hit hurt'
m = re.findall("[bh][aiu].?t",data)
if m is not None:
  print m
>['bat', 'bit', 'but', 'hat', 'hit', 'hurt']
```

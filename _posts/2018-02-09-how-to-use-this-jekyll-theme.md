---
layout: post
title:  如何使用LessOrMore这个Jekyll模版
date:   2018-02-09 01:08:00 +0800
categories: document
tag: jekyll
published: false
---

* content
{:toc}


### jekyll
正式把博客从hexo迁至jekyll并采用github pages；记录如下

官网 https://jekyllrb.com/

使用git从[LessOrMore](https://github.com/luoyan35714/LessOrMore.git)主页下载项目

### _config配置
```bash
name: 博客名称
email: 邮箱地址
author: 作者名
url: 个人网站
### baseurl修改为项目名，如果项目是'***.github.io'，则设置为空''
baseurl: "/LessOrMore"
resume_site: 个人简历网站
github: github地址
github_username: github用户名称
FB:
  comments :
    provider : duoshuo
    duoshuo:
        short_name : 多说账户
    disqus :
        short_name : Disqus账户
```

### 写文章

如何写文章							{#How-to-write-document}
------------------------------------

在`LessOrMore/_posts`目录下新建一个文件，可以创建文件夹并在文件夹中添加文件，方便维护。在新建文件中粘贴如下信息，并修改以下的`titile`,`date`,`categories`,`tag`的相关信息，添加`* content {:toc}`为目录相关信息，在进行正文书写前需要在目录和正文之间输入至少2行空行。然后按照正常的Markdown语法书写正文。

```bash
---
layout: post
#标题配置
title:  标题
#时间配置
date:   2016-08-27 01:08:00 +0800
#大类配置
categories: document
#小类配置
tag: 教程
---

* content
{:toc}

我是正文。我是正文。我是正文。我是正文。我是正文。我是正文。
```

```bash
jekyll build将markdown文件生成html文件
#若要在localhost直接预览可以直接运行后，浏览器访问4000端口看效果
jekyll server
```

### 插入图片
```
{% raw %}
<img src="{{ '/styles/images/jiezishu.jpg' | prepend: site.baseurl }}" alt="诫子书" width="310" />
{% endraw %}
```

诫子书				{#zhugeliang}
------------------------

<img src="{{ '/styles/images/jiezishu.jpg' | prepend: site.baseurl }}" alt="诫子书" width="310" />

[诸葛亮](#)


夫君子之行，静以修身，俭以养德。非淡泊(澹泊)无以明志，非宁静无以致远。夫学须静也，才须学也。非学无以广才，非志无以成学。淫慢则不能励精，险躁则不能冶性。
年与时驰，意与日去，遂成枯落，多不接世，悲守穷庐，将复何及！

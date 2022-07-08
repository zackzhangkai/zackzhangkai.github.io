---
layout: post
title: scapy-Request and Post
date:   2016-02-24
categories: document
tag:
  - python
  - scrapy
  - 爬虫
---

* content
{:toc}

### Request对象

```python
class scrapy.http.Request(url[, callback, method='GET', headers, body, cookies, meta, encoding='utf-8', priority=0, dont_filter=False, errback, flags])
```

一个request 对象代表了一上HTTP请求，在spider里产生，被Downloader执行后，产生一个Response

####利用callback

```python
def parse_page1(self, response):
    return scrapy.Request("http://www.example.com/some_page.html",
                          callback=self.parse_page2)

def parse_page2(self, response):
    # this would log http://www.example.com/some_page.html
    self.logger.info("Visited %s", response.url)
```

#### 参数
+ url：
+ callback:如果不指定，则将调用下面的parse()方法
+ method:默认`GET`
+ meta:(dict)
+ body
+ headers
+ cookies:两种形式，如下

egs:

1.用dict

```python
reqeust_with_cookies = Request(url="http://www.example.com",cookies={'currency':'USD','country':'UY'})
```

2.用list

```python
reqeust_with_cookies = Request(url="http://www.example.com",
                               cookies=[{'name': 'currency',
                                         'value': 'USD',
                                         'domain': 'example.com',
                                         'path': '/currency'}])
```


### FormRequest

```python
class scrapy.http.FormRequest(url[, formdata, ...])
```

egs:

+ 利用HTTP post用FormRequests发送数据
```python
return [FormRequest(url="http://www.example.com/post/action",
                    formdata={'name': 'John Doe', 'age': '27'},
                    callback=self.after_post)]
```

+ 模拟登陆
```python
# -*- coding: utf-8 -*-
import scrapy
from logging import warning

class ZhilianSpider(scrapy.Spider):
    name = 'zhilian'
    # allowed_domains = ['www.zhaopin.com']
    start_urls = ['https://passport.zhaopin.com/account/login']

    def parse(self, response):
        warning("{}".format(response.status))
        return scrapy.FormRequest.from_response(
                response,
                formdata={
                    'int_count': '999',
                    'errUrl': "https://passport.zhaopin.com/account/login",
                    'RememberMe': 'false',
                    'requestFrom': 'portal',
                    'loginname':'925370765@qq.com',
                    'Password': 'xxx',
                },
                callback=self.after_login
        )

    def after_login(self,response):
        self.logger.warn("{}".format(response.url))
        #self.logger.warn("{}".format(type(response.url)))
        #由于登陆成功后redirect到https://i.zhaopin.com,因此通过这个来判断是否登陆成功
        if  response.url == "https://i.zhaopin.com":
             self.logger.warn("Login sucess")
        else:
            self.logger.error("Login fail")
        return
```

####  HtmlResponse
```python
class scrapy.http.HtmlResponse(url[, ...])
```
导入
```python
In [1]: from scrapy.http import HtmlResponse

In [2]: from scrapy.selector import Selector

In [3]: body="""
   ...: <html>
   ...:  <head>
   ...:   <base href='http://example.com/' />
   ...:   <title>Example website</title>
   ...:  </head>
   ...:  <body>
   ...:   <div id='images'>
   ...:    <a href='image1.html'>Name: My image 1 <br /><img src='image1_thumb.
   ...: jpg' /></a>
   ...:    <a href='image2.html'>Name: My image 2 <br /><img src='image2_thumb.
   ...: jpg' /></a>
   ...:    <a href='image3.html'>Name: My image 3 <br /><img src='image3_thumb.
   ...: jpg' /></a>
   ...:    <a href='image4.html'>Name: My image 4 <br /><img src='image4_thumb.
   ...: jpg' /></a>
   ...:    <a href='image5.html'>Name: My image 5 <br /><img src='image5_thumb.
   ...: jpg' /></a>
   ...:   </div>
   ...:  </body>
   ...: </html>
         """
In [4]: url = "https://doc.scrapy.org/en/latest/_static/selectors-sample1.html"
In [6]: response = HtmlResponse(url=url,body=body,encoding="utf8")
Out[6]: <200 https://doc.scrapy.org/en/latest/_static/selectors-sample1.html>
In [13]: response.xpath("//title")
Out[13]: [<Selector xpath='//title' data='<title>Example website</title>'>]

```

#### response.urljoin
该方法是对urlparse.urljoin的封装
```
urlparse.urljoin(base, url[, allow_fragments])
```
+ 若后面的url是相对路径，则与第一个url合并产生新的url
```python
>>> from urlparse import urljoin
>>> urljoin('http://www.cwi.nl/%7Eguido/Python.html', 'FAQ.html')
'http://www.cwi.nl/%7Eguido/FAQ.html'
```
+ 若后面的url是绝对路径，则直接用第二个url
```
>>> urljoin('http://www.cwi.nl/%7Eguido/Python.html',
...         '//www.python.org/%7Eguido')
'http://www.python.org/%7Eguido'
```
用法：response.urljoin(url)，与此类似，只不过省略了base，base默认为response.url; 另response.follow是对该方法的进一步封装
用法如下：
```python
yield response.follow(next_page,callback=self.parse)
yield Request(response.urljoin(next_page),callback=self.parse)
```
以上二者的作用一样

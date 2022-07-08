---
layout: post
published: true
title:  python之mock
categories: [document]
tags: [python]
---
* content
{:toc}

### 前言
Mock是Python中一个用于支持单元测试的库，它的主要功能是使用mock对象替代掉指定的Python对象，以达到模拟对象的行为。

### 示例
```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import requests


def send_request(url):
    r = requests.get(url)
    return r.status_code


def visit_ustack():
    return send_request('http://www.ustack.com')
```

外部模块调用visit_ustack()来访问UnitedStack的官网。下面我们使用mock对象在单元测试中分别测试访问正常和访问不正常的情况。

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import unittest
import mock
import client


class TestClient(unittest.TestCase):

    def test_success_request(self):
        success_send = mock.Mock(return_value='200')
        client.send_request = success_send
        self.assertEqual(client.visit_ustack(), '200')

    def test_fail_request(self):
        fail_send = mock.Mock(return_value='404')
        client.send_request = fail_send
        self.assertEqual(client.visit_ustack(), '404')
```

找到要替换的对象：我们需要测试的是visit_ustack这个函数，那么我们需要替换掉send_request这个函数。

实例化Mock类得到一个mock对象，并且设置这个mock对象的行为。在成功测试中，我们设置mock对象的返回值为字符串“200”，在失败测试中，我们设置mock对象的返回值为字符串"404"。

使用这个mock对象替换掉我们想替换的对象。我们替换掉了client.send_request

写测试代码。我们调用client.visit_ustack()，并且期望它的返回值和我们预设的一样。

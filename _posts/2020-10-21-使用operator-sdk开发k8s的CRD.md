---
layout: post
published: false
title:  写作模版
categories: [document]
tags: [operator,go,k8s]
---
* content
{:toc}

## 安装

https://sdk.operatorframework.io/docs/installation/install-operator-sdk/#install-from-homebrew-macos

```bash
brew install operator-sdk
```

## 创建项目

```bash
operator-sdk init --domain=example.com --repo=github.com/example-inc/memcached-operator
```

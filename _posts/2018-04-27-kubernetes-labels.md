---
layout: post
title:  kubernetes labels
date:   2018-04-27 01:08:00 +0800
categories: document
tag:
  - kubernetes
---

* content
{:toc}

# Labels and Selectors
labels就是key/value键值对，附加到如pod类似对象上，设置它的目的主要是便于用户来识别区分，对核心并不影响，用于组织和选择一组对象。labels在创建的时候就可以附加，但可以在任何时候修改。每个对象可以定义一系列键值对的标签。每个键必须是惟一的。
```
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```
我们最终将使用索引和反索引标签来高效率的查询和监视，在UI或是CLI上分类划是分组，不用非定义或是大型的或是非结构化的数据污染标签。非定义的信息应该写入annotations.

## Motivation
```
"release" : "stable", "release" : "canary"
"environment" : "dev", "environment" : "qa", "environment" : "production"
"tier" : "frontend", "tier" : "backend", "tier" : "cache"
"partition" : "customerA", "partition" : "customerB"
"track" : "daily", "track" : "weekly"
```
## Label selectors
不同于names和uids，labels并不是惟一的，我们希望多个对象有相同的labels，通过label selector,可以定义，label selector是主要用来分组的。API目前支持两种形式的selectors
+ equality-based

```
environment = production
tier != frontend
```

```
$ kubectl get pods -l environment=production,tier=frontend
```
+ set-based

hree kinds of operators are supported: in,notin and exists (only the key identifier)

```
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```

```
$ kubectl get pods -l 'environment in (production),tier in (frontend)'
$ kubectl get pods -l 'environment in (production, qa)'
```

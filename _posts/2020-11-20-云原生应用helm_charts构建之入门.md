---
layout: post
published: true
title:  云原生应用Helm应用helm_charts入门篇
categories: [document]
tags: [istio, helm, charts, kubernetes]
---
* content
{:toc}

## 前言

由于K8s的资源声明式API，所有的资源通过定义Yaml来管理；资源类型众多，如 `Service` `Deployment` `Ingress`等，从一个具体对外提供服务的应用的视角来看，这些资源之间相互依赖，因此我们将它们抽象出来统称为`应用`。因此`应用`本质上就是一堆相关联的yaml文件的集合。而这堆`yaml`之间存在着很多相同的共性，例如`Deployment`与`Service`的Name；或是有些变量我们需要抽象出来放在一个配置文件中，方便安装或是升级，如`Deployment`的`Replicas`副本数量。这就是Helm chart的来源。

## Helm安装

```bash
$ brew install helm

$ helm version

...
version.BuildInfo{Version:"v3.4.1", GitCommit:"c4e74854886b2efe3321e185578e6db9be0a6e29", GitTreeState:"dirty", GoVersion:"go1.15.4"}
```

## 制作第一个Chart包

```bash
$ helm create hello-world

WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /Users/hugo/.kube/config
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: /Users/hugo/.kube/config
Creating hello-world
```

可以看到第一个初始化的包已经制作出来了

```bash
$ tree .
.
└── hello-world
    ├── Chart.yaml # Chart及应用版本信息
    ├── charts # 依赖的Charts
    ├── templates
    │   ├── NOTES.txt # 输出安装完后的提示信息
    │   ├── _helpers.tpl # 内部的变量及条件判断逻辑
    │   ├── deployment.yaml
    │   ├── hpa.yaml
    │   ├── ingress.yaml
    │   ├── service.yaml
    │   ├── serviceaccount.yaml
    │   └── tests
    │       └── test-connection.yaml
    └── values.yaml  # 自定义的变量，如副本数、镜像等
```

看下values.yaml的信息，如果要修改默认值，可以在安装的时候加上`--set`

```bash
replicaCount: 1
image:
  repository: nginx
...
```

检验包

```bash
$ helm lint hello-world
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /Users/hugo/.kube/config
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: /Users/hugo/.kube/config
==> Linting hello-world
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

打包

```bash
$ helm package hello-world hello-world
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /Users/hugo/.kube/config
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: /Users/hugo/.kube/config
Successfully packaged chart and saved it to: /tmp/test/hello-world-0.1.0.tgz
Successfully packaged chart and saved it to: /tmp/test/hello-world-0.1.0.tgz
```

安装

```bash
$ helm install hello-world hello-world-0.1.0.tgz
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /Users/hugo/.kube/config
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: /Users/hugo/.kube/config
NAME: hello-world
LAST DEPLOYED: Fri Nov 20 11:28:45 2020
NAMESPACE: xxx
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace xxx -l "app.kubernetes.io/name=hello-world,app.kubernetes.io/instance=hello-world" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace xxx $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace xxx port-forward $POD_NAME 8080:$CONTAINER_PORT
```

此时应用已经安装完成

```bash
$ kubectl get po
NAME                               READY   STATUS    RESTARTS   AGE
hello-world-75c474b897-tfprp       1/1     Running   0          72s
```

上面的输出信息提供了三个步骤，分别是获取Pod Name，获取Pod的暴露端口，将这个端口暴露到外部

将这个pod 端口暴露看下效果

```bash
$ kubectl port-forward hello-world-75c474b897-tfprp 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

访问正常

```bash
$ curl http://localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
```
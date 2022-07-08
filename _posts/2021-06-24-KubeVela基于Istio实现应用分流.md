---
layout: post
published: true
title: KubeVela基于Istio实现应用分流
categories: [k8s]
tags: [k8s,kubevela,oam]
---
* content
{:toc}

## 创建application

针对业务开发人员来讲：应用分为`前端`、`后端`、（定时）`任务`。

对于kubernetes操作人员来讲，我不管你们什么前端、后端，都是deployment/pods、statefuset、job或是cronjob。从这个角度，其实并不清楚应用间的调用关系，并不明朗，这个是oam要解决的问题。

部署一个应用：

```bash
kubectl apply -f - <<EOF
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: example-app
  annotations:
    app.oam.dev/revision-only: "true"
spec:
  components:
    - name: testsvc
      type: webservice
      properties:
        addRevisionLabel: true
        image: crccheck/hello-world
        port: 8000
EOF
```

部署完后，发现其实并没有pods

```bash
$ k get po
No resources found in default namespace.
```

**原因是使用了annotation:  `app.oam.dev/revision-only: "true"`**

这个annotation是定义资源被appdeploymnet控制

可以看到已经有apprev

```bash
➜  ~ kubectl get apprev
NAME             AGE
example-app-v1   2m12s
```

## 创建appdeployment

```bash
kubectl apply -f - <<EOF
apiVersion: core.oam.dev/v1beta1
kind: AppDeployment
metadata:
  name: example-appdeploy
spec:
  appRevisions:
    - revisionName: example-app-v1

      placement:
        - distribution:
            replicas: 2
EOF
```

应用appdeployment后，可以看到负载已经正确创建，且deployment有两个副本

```bash
k get deploy
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
testsvc-v1   2/2     2            2           91s

k get po
NAME                          READY   STATUS    RESTARTS   AGE
testsvc-v1-5c94ff8db7-6tx5n   1/1     Running   0          94s
testsvc-v1-5c94ff8db7-dw558   1/1     Running   0          94s
```

## 更新application

应用名一样，镜像由hello-world变为nginx

```bash
cat <<EOF | kubectl apply -f -
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: example-app
  annotations:
    app.oam.dev/revision-only: "true"
spec:
  components:
    - name: testsvc
      type: webservice
      properties:
        addRevisionLabel: true
        image: nginx
        port: 80
EOF
```

更新完后，会有两个版本

```bash
$ kubectl get apprev
NAME             AGE
example-app-v1   9m58s
example-app-v2   13s
```

虽然application的revision更新了，但是pods仍旧没有变化。

原因是owner不是application，而是appdeployment

```bash
$ kubectl get deploy -oyaml |grep -i owner -C 2
...
      blockOwnerDeletion: true
      controller: true
      kind: AppDeployment
```

而appdeployment中定义的还是version v1

```bash
# k get appdeployment example-appdeploy -oyaml
...
spec:
  appRevisions:
  - placement:
    - distribution:
        replicas: 2
    revisionName: example-app-v1
```

## 更新appdeployment实现分流

分流的功能本质上是借助于istio 的能力，但是屏蔽了istio的复杂配置。

***安装istio***

`istioctl install --demo`

等待安装成功：

```bash
➜ kubectl get po -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-bd477794-md2n6      1/1     Running   0          4h3m
istio-ingressgateway-79df7c789f-pbw2v   1/1     Running   0          4h3m
istiod-6dc55bbdd-fv859                  1/1     Running   0          4h5m
```

***部署appdeployment，安流量比例分***

**apply gateway**

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: example-app-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
EOF
```

检查gateway成功创建

```bash
kubectl get gateway
NAME                  AGE
example-app-gateway   4h12m
```

**更新appdeployment**

```bash
kubectl apply -f - <<EOF
apiVersion: core.oam.dev/v1beta1
kind: AppDeployment
metadata:
  name: example-appdeploy
spec:
  traffic:
    hosts:
      - example-app.example.com
    gateways:
      - example-app-gateway
    http:
      - weightedTargets:
          - revisionName: example-app-v1
            componentName: testsvc
            port: 8000
            weight: 50
          - revisionName: example-app-v2
            componentName: testsvc
            port: 80
            weight: 50

  appRevisions:
    - revisionName: example-app-v1
      placement:
        - distribution:
            replicas: 1

    - revisionName: example-app-v2

      placement:
        - distribution:
            replicas: 1
EOF
```

检查deployment成功创建

```bash
kubectl get deploy
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
testsvc-v1   1/1     1            1           35m
testsvc-v2   1/1     1            1           23m
```

## 验证

把istio ingress gateway svc暴露，让外网访问

```bash
kubectl -n istio-system port-forward service/istio-ingressgateway 8080:80
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

访问：

```bash
➜  ~ curl -H "Host: example-app.example.com" http://localhost:8080/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

再次访问：

```bash
➜  ~ curl -H "Host: example-app.example.com" http://localhost:8080/
<xmp>
Hello World


                                       ##         .
                                 ## ## ##        ==
                              ## ## ## ## ##    ===
                           /""""""""""""""""\___/ ===
                      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
                           \______ o          _,/
                            \      \       _,'
                             `'--.._\..--''
</xmp>
```

依次访问，会发现交叉出现两个页面，证明分流成功。

## 总结：

这个流程总结为：  

1. 创建app，指定使用appdeployment控制；  
2. 更新app，生成两个版本v1,v2；  
3. 更新appdeployment，指定两个版本的流量比例；  

**疑问1：在app中不能一下次定义两个版本吗？**

**疑问2：这个分流是使用istio的功能吗**
是的，使用的是istio的virtualservice，如：

```bash
➜  ~ k get virtualservice
NAME                GATEWAYS                  HOSTS                         AGE
example-appdeploy   ["example-app-gateway"]   ["example-app.example.com"]   18m
```

<https://kubevela.io/docs/end-user/scopes/appdeploy/>

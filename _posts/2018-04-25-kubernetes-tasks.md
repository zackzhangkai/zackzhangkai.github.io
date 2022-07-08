---
layout: post
title:  kubernetes tasks from official website
date:   2018-04-25 01:08:00 +0800
categories: document
tag:
  - kubernetes
---

* content
{:toc}

###  Configure a Pod to Use a Volume for Storage
```
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - name: redis-storage
      mountPath: /data/redis
  volumes:
  - name: redis-storage
    emptyDir: {}
```

```
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/pod-redis.yaml

kubectl get pod redis --watch

NAME      READY     STATUS    RESTARTS   AGE
redis     1/1       Running   0          13s

kubectl exec -it redis -- /bin/bash

root@redis:/data# cd /data/redis/
root@redis:/data/redis# echo Hello > test-file

root@redis:/data/redis# ps aux

USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
redis        1  0.1  0.1  33308  3828 ?        Ssl  00:46   0:00 redis-server *:6379
root        12  0.0  0.0  20228  3020 ?        Ss   00:47   0:00 /bin/bash
root        15  0.0  0.0  17500  2072 ?        R+   00:48   0:00 ps aux

root@redis:/data/redis# kill <pid>
```
此时redis pod会直接重新创建，根据配置可以预判这个持久化的文件应该仍然存在

```
kubectl exec -it redis -- /bin/bash
ls /data/redis  ## test-file
```

### Configure a Pod to Use a PersistentVolume for Storage

即 pvc，它的过程如下：

1. 集群管理员创建一个由物理存储器支持的pv， 管理员不会将卷与任何Pod关联起来。
2. 集群用户创建的pvc，会自动地绑定到一个合适的pv上
3. 用户创建的pod会使用pvc作为存储

#### Create an index.html file on your Node
```
mkdir /mnt/data
echo 'Hello from Kubernetes storage' > /mnt/data/index.html
```

#### Create a PersistentVolume

在本章我们将创建一个hostpath的 pv。kubernetes在单结点上支持的hostpath的开发和测试。一个hostPath pv在node上使用一个文件或是一个目录来模拟网络连接的存储。

在生产集群中，你将不会使用hostPath。相反，集群管理员将提供一个网络存储，如gce的持久化磁盘，一个NFS共享，或是亚马逊弹性块存储（）。同样可以StorageClasses来设置动态提供（dynamic provisioning）.

```
kind: PersistentVolume
apiVersion: v1
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```
该配置指明卷在的集群Node节点的/mnt/data,容量为10G，模式为ReadWriteOnce，即表明可挂载在单个Node上。


```
[root@master1 data]# kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/task-pv-volume.yaml
persistentvolume "task-pv-volume" created
[root@master1 data]# kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
redis     1/1       Running   2          5h
[root@master1 data]# kubectl get pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
task-pv-volume   10Gi       RWO            Retain           Available             manual                   37s
```
可以看到pv的STATUSo是Available，也就意味着这个还没有绑定到PVC上面。

#### 创建一个PVC（PersistentVolumeClaim）
下一步就是创建pvc，pod利用pvc去请求物理内存

```
#task-pv-claim.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

```
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/task-pv-claim.yaml
```
当你创建了PVC，kubernetes会自动寻找符合要要求的pv,pv就是一个卷，当发现是同样的存储类别storageClass后，将claim绑定到volume
```
[root@master1 ~]# kubectl get pvc
NAME            STATUS    VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
task-pv-claim   Bound     task-pv-volume   10Gi       RWO            manual         15m
[root@master1 ~]# kubectl get pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                   STORAGECLASS   REASON    AGE
task-pv-volume   10Gi       RWO            Retain           Bound     default/task-pv-claim   manual                   1h
```

#### Create Pod
下一步就是将你的pvc当作一个卷来创建pod
```
kind: Pod
apiVersion: v1
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
       claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```
注意pod的配置文件里指定了pvc，但是没有指定pv，从pod的角度，claim即是一个volume卷

```
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/task-pv-pod.yaml
```

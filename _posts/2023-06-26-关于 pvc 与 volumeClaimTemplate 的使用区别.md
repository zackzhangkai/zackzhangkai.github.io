---
layout: post
published: true
title: 2023-06-26-关于 pvc 与 volumeClaimTemplate 的使用区别
categories: [document]
tags: [k8s,pvc,volumeClaimTemplate]
---
* content
{:toc}


# 区别：

这两者都可以声明出一个 pvc，并给容器使用。

pvc 声明的时候，需要在 volumes 中声明使用的 pvc，然后在 container 中 volumeMounts 挂载到容器中：

```yaml
spec:
      containers:
        - name: mysql
					...
          volumeMounts:
            - name: mysql-pv-claim
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: mysql-pv-claim
```

volumeClaimTemplate 只能在 statefulSet中使用：

如：以下为 mysql 的 statefulSet 的 yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:latest
          env:
            - name: MYSQL_USER
              value: gorm
            - name: MYSQL_PASSWORD
              value: gorm
            - name: MYSQL_DATABASE
              value: review
          ports:
            - name: mysql
              containerPort: 3306
          volumeMounts:
            - name: mysql-pv-claim
              mountPath: /var/lib/mysql
#      volumes:
#        - name: mysql-persistent-storage
#          persistentVolumeClaim:
#            claimName: mysql-pv-claim
  volumeClaimTemplates:
    - metadata:
        name: mysql-pv-claim
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi
```

当有多副本的时候，每个 pod 会绑定使用一个 pvc，pvc 的名字由模版中的名字加上 pod 的名字组成。

```yaml
✗ k get pvc -n default
NAME                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
mysql-pv-claim-mysql-0   Bound    pvc-c0021ef0-59b9-451d-beb3-369d4c890119   1Gi        RWO            openebs-hostpath   34m
```
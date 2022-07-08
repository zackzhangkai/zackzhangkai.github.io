---
layout: post
title:  基于kubeadm安装kubernetes cluster
date:   2018-03-09 01:08:00 +0800
categories: document
tag:
  - kubernetes
---

* content
{:toc}

### 什么是kubernets

[官网](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)

k8s是一个便携式、可扩展的开源平台，管理容器负载和服务，配置及自动化。提供了应用部署，规划，更新，维护的一种机制

### 安装docker
```
apt-get install -y docker.io
```
### 内核
内核必须支持 memory and swap accounting 。确认你的linux内核开启了如下配置
```bash
#cat /boot/config-***-generic
CONFIG_RESOURCE_COUNTERS=y
CONFIG_MEMCG=y
CONFIG_MEMCG_SWAP=y
CONFIG_MEMCG_SWAP_ENABLED=y
CONFIG_MEMCG_KMEM=y
```
以命令行参数方式,在内核启动时开启 memory and swap accounting 选项:
```
GRUB_CMDLINE_LINUX="cgroup_enable=memory	swapaccount=1"
```

```bash
$vim /etc/default/grub
```
修改 GRUB_CMDLINE_LINUX="" ==> GRUB_CMDLINE_LINUX="cgroup_enable=memory"

保存后, 更新grub.cfg
```bash
update-grub
reboot
$cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-3.18.4-aufs root=/dev/sda5 ro cgroup_enable=memory swapaccount=1
```
### 用kubeadm安装k8s
本机环境：ubuntu LTS 1604
```bash
Linux master-node-01 4.4.0-116-generic #140-Ubuntu SMP Mon Feb 12 21:23:04 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

#### 安装kubeadm
```bash
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
#Configure cgroup driver used by kubelet on Master Node
sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

#### 初始化master
```
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.12.9
```

在初始化输出后有提示
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

当然如果是root用户还可以设置
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```

#### net pod
继续看，还提示要为cluster部署一个网络pod


访问提示的网站，可以看到有很多供选择，如 Calico,Canal,Flannel,Kube-router,Romanna,Weave Net


```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
```

若network已安装成功，可通过检查kube-dns pod是否正常运行

```
kubectl get pods --all-namespaces
```

kube-dns pod运行成功后，就可以继续加入nodes


#### join nodes
再住下看，还有提示加入cluster的信息
```bash
 kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```

workloads工作负载包含pods和containers等，nodes是工作负载运行的地方。在每个新node上执行以下步骤可将其加入集群

```
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```

安装完flannel节点后，node节点上的images镜像如下：
```bash
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy-amd64   v1.10.1             6e6237849607        10 days ago         97.1 MB
k8s.gcr.io/pause-amd64        3.1                 da86e6ba6ca1        4 months ago        742 kB
quay.io/coreos/flannel        v0.9.1-amd64        2b736d06ca4c        5 months ago        51.3 MB
```

#### kubeadm token

```
kubeadm token create --print-join-command  #用这个命令可以查看kubeadm init输出加入cluster的信息
```

最后，在master上查看信息
```
[root@master1 ~]# kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
master1   Ready     master    17m       v1.10.1
node1     Ready     <none>    17m       v1.10.1
node2     Ready     <none>    17m       v1.10.1

[root@master1 ~]# docker images
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy-amd64                v1.10.1             6e6237849607        10 days ago         97.1 MB
k8s.gcr.io/kube-apiserver-amd64            v1.10.1             9df3c00f55e6        10 days ago         225 MB
k8s.gcr.io/kube-scheduler-amd64            v1.10.1             ceecd7155649        10 days ago         50.4 MB
k8s.gcr.io/kube-controller-manager-amd64   v1.10.1             8401bb3ff261        10 days ago         148 MB
k8s.gcr.io/etcd-amd64                      3.1.12              52920ad46f5b        6 weeks ago         193 MB
k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64     1.14.8              c2ce1ffb51ed        3 months ago        40.9 MB
k8s.gcr.io/k8s-dns-sidecar-amd64           1.14.8              6f7f2dc7fab5        3 months ago        42.2 MB
k8s.gcr.io/k8s-dns-kube-dns-amd64          1.14.8              80cc5ea4b547        3 months ago        50.5 MB
k8s.gcr.io/pause-amd64                     3.1                 da86e6ba6ca1        4 months ago        742 kB
k8s.gcr.io/kubernetes-dashboard-amd64      v1.8.1              e94d2f21bc0c        4 months ago        121 MB
quay.io/coreos/flannel                     v0.9.1-amd64        2b736d06ca4c        5 months ago        51.3 MB

[root@master1 ~]# kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                                    READY     STATUS             RESTARTS   AGE       IP             NODE
kube-system   etcd-master1                            1/1       Running            0          36m       192.16.35.12   master1
kube-system   kube-apiserver-master1                  1/1       Running            0          36m       192.16.35.12   master1
kube-system   kube-controller-manager-master1         1/1       Running            0          36m       192.16.35.12   master1
kube-system   kube-dns-86f4d74b45-j6mqc               3/3       Running            0          36m       10.244.0.7     master1
kube-system   kube-flannel-ds-clsww                   1/1       Running            0          36m       192.16.35.12   master1
kube-system   kube-flannel-ds-gh8rs                   1/1       Running            4          36m       192.16.35.10   node1
kube-system   kube-flannel-ds-tdhtz                   1/1       Running            0          36m       192.16.35.11   node2
kube-system   kube-proxy-4nzzc                        1/1       Running            0          36m       192.16.35.11   node2
kube-system   kube-proxy-lgfrk                        1/1       Running            0          36m       192.16.35.12   master1
kube-system   kube-proxy-mjjgk                        1/1       Running            0          36m       192.16.35.10   node1
kube-system   kube-scheduler-master1                  1/1       Running            0          36m       192.16.35.12   master1
kube-system   kubernetes-dashboard-8469446955-jsj6m   0/1       CrashLoopBackOff   11         36m       10.244.0.6     master1
```

### 利用ansible部署基于kubeadm的k8s
https://github.com/kaizamm/kubeadm-ansible.git


### Dashboard
```bash
kubectl proxy --address="192.168.12.9" -p 8001 --accept-hosts='^*$'
```

### troubleshooting

+ Q: kubelet无法启动
+ A: 通过journalctl -xeu kubelet查看提示信息，注意思如下三点：1.cgroupsfs的修改 2.kubeadm 的执行结果（master:kubeadm join; nodes: kubeadm join）

+ Q: 'kubectl get nodes'查看到的信息只有一个master提示ready，其余notready
+ A: 查看nodes的flannel网络是否正常，查看image是否已全部下载完毕

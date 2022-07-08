---
layout: post
published: true
title:  Kubernetes之Ingress Controller
categories: [ kubernetes ]
tags: [ ingress,kubernetes ]
---
* content
{:toc}

## 什么是 ingress？

ingress 是 在 service 上面，流量从 ingress 进入，七层负载的路由规则。可以根据 Path、URI、Header(如Host) 等参数通过路由规则来实现流量分发、url重写、ssl/tls卸载等功能。

ingress controller 根据ingress的规则及annotation/labels生成相应的配置文件，如nginx的configuration。

controller 有很多，比如：

1. Nginx Ingress：K8s社区开发的一款Ingress，nginx用的lua模块，功能较多，但是reload慢，稳定性不强；

2. Kubernetse Ingress Plus：Nginx官方开发，用的是nginx原生的，没有第三方模块，解决了reload慢的问题，提供商业支持。

> Nginx Plus 收费，在官网上注册后可以下载证书和私钥，免费使用一个月。控制器的安装方法：把证书私钥拷备到当前目录-打包镜像时指定自己的私有仓库地址（打包完镜像会自动上传至镜像仓库）-deployment 里面需要指定刚才这个镜像。

3. traefic: 有漂亮的 UI，用的人比较多。

4. kong: 也是基于 nginx的。

5. istio
...

![详情](/styles/images/ingress-controllers.png)

流量入口：将controller的service服务暴露（NodePort/LoadBalance），外部流量可以从此处进入。

## Nginx Ingress

本质上是生成nginx的配置文件，代理流量到后端。

查看配置文件

```bash
# kubectl -n kubesphere-controls-system exec kubesphere-router-ns1-6979bffc79-c4fqm -- cat /etc/nginx/nginx.conf

...
http {
    # Global filters
    ## start server productpage.ns1.192.168.0.12.nip.io
    server {
            # 指定Host curl -H 'Host: xxx'；或未显示指定，则默认为URL中的Host，即第一个分隔断，如www.baidu.com/hello?中的www.baidu.com
            server_name productpage.ns1.192.168.0.12.nip.io ;  
            listen 80;
            set $proxy_upstream_name "-"; 
            set $pass_access_scheme $scheme;
            set $pass_server_port $server_port;
            set $best_http_host $http_host;
            set $pass_port $pass_server_port;

            # 监听80/443端口，443端口会在此处Tls/Ssl证书卸载
            listen 443  default_server reuseport backlog=511 ssl http2;
            ssl_certificate     /etc/ingress-controller/ssl/default-fake-certificate.pem;
            ssl_certificate_key   /etc/ingress-controller/ssl/default-fake-certificate.pem;

            # 匹配上该Host，url默认进入下面的Location
            location / {
                    set $namespace      "ns1";
                    set $ingress_name   "bookinfo-ingress";
                    set $service_name   "productpage";
                    set $service_port   "9080";
                    set $location_path  "/";
                    set $proxy_upstream_name    "ns1-productpage-9080";
                    set $proxy_host             $proxy_upstream_name;
                    # 设置vhost，annotation的设置nginx.ingress.kubernetes.io/upstream-vhost
                    # 该项设置会匹配upstream时，类似 curl -H 'Host:'，只是设置了请求的Host
                    proxy_set_header Host   "productpage.ns1.svc.cluster.local";
                    # 启用websocket，当使用Nginx给其他服务代理时，若有websocket需启用
                    proxy_set_header Upgrade $http_upgrade;
                    # 默认会生成X-Request-ID，当使用Tracing功能时，依赖该ID
                    proxy_set_header X-Request-ID           $req_id;
                    proxy_set_header X-Real-IP              $the_real_ip;
                    # 透明代理时获取客户端IP
                    proxy_set_header X-Forwarded-For        $the_real_ip;
                    proxy_pass http://upstream_balancer;
                    proxy_redirect                          off;
            }
    }
    server {...}
}
```

`nginx.ingress.kubernetes.io/service-upstream` 默认值为false，将该ingress中定义的service的endpoint的IP，即pod的IP作为nginx的后端，此时可以利用nginx本身的访问策略，如轮循/最小连接/hash等；若为true，则将service的ip作为nginx controller的后端，此时无法使用nginx本身的访问策略，转而使用了service ipvs的功能。[关于Nginx Annotation](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#service-upstream)

`nginx.ingress.kubernetes.io/service-upstream` Rewrite Http Header的Host

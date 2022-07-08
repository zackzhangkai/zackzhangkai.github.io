---
layout:       post
title:        "Dockerfile reference"
date:         2016-12-20 12:00:00
categories: document
tag: docker
---

* content
{:toc}

### Dockerfile reference

参考[官方文档](https://docs.docker.com/engine/reference/builder/)

Docker 能通过Dockerfile文件自动创建镜像。一个dockerfile是由命令行执行创建镜像的所有命令的文本文档。用户可以通过docker build来在命令框执行命令来达到自动创建的目的。

本页面列出了能在Dockerfile里使用的命令。

### 用法

+  一般Dockerfile位于context目录。你可以使用 -f 参数 配合docker build来指定Dockerfile的位置，指定dockerfile的路径，docker build PATH context，path为dockerfile的路径，context为工作目录，build时会把context里的内容加进来，若不想将有些文件或目录加进来，则需要写在.ignoredockerfile里 `docker build -f /dockerfile/path .`  

+ 对build success的image做标签 `docker build -t kaiz/myapp .`

### 解析指令

### escape

转义字符，默认为\

### Environment replacement

${variable:-word} 当variable变量赋值后，该变量的值即为新赋的值；若未被赋值，则word为其初始值

${variable:+word} 当variable变量赋值后，该变量的值即为新赋的值，若未被赋值，则为空字符串

### FROM

FROM image

或者

+ FROM image:tag

+ FROM image@digest

若省掉tag或者digest，则认为latest

Dockerfile的第一条指令必须是FROM ，以此来指定你将选择的基础镜像

### RUN

两种格式：

+ RUN command  

+ RUN ["executable", "param1", "param2"]  

RUN将会在当前image的最上面新建一个layer并提交，新提交的image将会被下一个step使用

### CMD

三种格式：

+ `CMD ["executable","param1","param2"]`  首选，`exec form`

+ `CMD ["param1","param2"]`   ENTRYPOINT的默认参数

+ `CMD command param1 param2   shell form`

一个dockerfile只能有一个CMD，若有多个CMD，则最后一个生效，**CMD的主要作用是为容器提供一个默认项**，这些默认项可以被执行，但在使用ENTRYPOINT时则也可以被忽略。

> 注意:若CMD用于为ENTRYPOINT提供默认参数，则CMD和ENTRYPOINT指令需要使用JSON格式，即 单词两边加双引号 “”  ;与shell form不同，exec form不会去调用 command shell，即
> CMD [ "echo", "$HOME" ] 不会执行，需改成CMD[ "sh", "-c", "echo $HOME" ]

在使用exec 和 shell form时，CMD的命令将在容器开始run时执行。

```
FROM centos
CMD ["/usr/bin/wc","--help"]
```

### LABEL

LABEL指令以键值对的形式出现，给镜像加了很多标签，一个image可以有很多lable，每个LABLE指令会产新建一个layer，因此建议，只用一个LABLE指令包含所有的lables；
```
LABEL version="1.0" author="kaiz" date="2016.12.20"
```

### EXPOSE

格式：EXPOSE <port> [<port>...]
该指令让container在运行时监听特定的端口，但不暴露容器端口给宿主机，若要暴露端口，需加上-p来暴露一系列端口，或者用-P来暴露所有的端口。[更多](https://docs.docker.com/engine/reference/run/#expose-incoming-ports)

### ADD

两种格式：
+ `ADD src... dest`
+ `ADD ["src",..."dest"]`

ADD指令从src复制了新文件、目录、或远程文件URLs 并把他们加进image 的<dest>

```
ADD a.txt TESTDIR/    #在workdir下面须有一个TESTDIR的目录
ADD a.txt /data/
```

>注意：若<src> 是一个tar/gzip/zip/bzip/xz，则被解压成一个目录。一个文件是否被认为是压缩格式，是根据文件内容来识别的，并非由其命名来决定。

### COPY

两种格式：
+ COPY src... dest
+ COPY ["src",... "dest"]
若src为一个目录，则其本身目录不会被复制，仅仅是它下面的内容会被复制。
>ADD 与 COPY 的区别在于ADD可解压压缩文件，且ADD的src可为URL

### ENV

两种格式：
+ EVN key value
+ EVN key=value ...
第一种只能设置一个变量，第二种能同时设置很多变量。每个ENV新建一个层，因此尽量用第二种。

```
ENV myName="John Doe" myDog=Rex\ The\ Dog \
    myCat=fluffy
```

Rex后的\为转义符

```
输入 ENV abc=hello
输入 ENV abc=bye def=$abc
输入 ENV ghi=$abc
结果 def=hello,ghi=bye
```

可以通过 docker inspect 查看变量，也可以通过 docker run --env key=value来设置。
> 注意：evn设置的变量在下行生效

### ENTRYPOINT

两种格式：
+ ENTRYPOINT ["executable", "param1", "param2"]   exec form, 偏好此种
+ ENTRYPOINT command param1 param2   shell form
可利用此命令设置默认的命令和参数 ，利用CMD设置另外可以改变的参数

>CMD 和 ENTRYPOINT的区别：

>一个dockerfile中至少要有一个CMD或ENTRYPOINT命令;

>CMD可做为ENTRYPOINT的参数，CMD在启动container的时候可以被替代。

### .dockerignore file

不加进context的文件或目录，可用正则，[更多详情](https://docs.docker.com/engine/reference/builder/#dockerignore-file)

### USER

### WORKDIR

格式：WORKDIR /path/to/workdir

```
输入：WORKDIR /a
输入：WORKDIR b
输入：WORKDIR c
输入：RUN pwd
输出：/a/b/c
```

```
ENV DIRPATH /path
WORKDIR $DIRPATH/$DIRNAME
RUN pwd
```

### ARG

格式： ARG name[=default-value]
+ FROM ubuntu
+ ARG CONT_IMG_VER
+ ENV CONT_IMG_VER v1.0.0
+ RUN echo $CONT_IMG_VER

`docker build --build-arg CONT_IMG_VER=V2.0 .`
CONT_IMG_VER为v1.0.0，原因为ENV的变量全局变量在IMAGE中一直存在。
command-line用法
`--build-arg varname=value`，前提ARG 定义的变量需要一个初始值。

ARG定义的变量与ENV不同，ENV全局变量在IMAGE中一直存在，ARG类似于shell里的local定义变量，在cache中存在。

### VOLUME
格式：VOLUME ["/data"]

``
FROM centos
RUN mkdir /myvol
RUN echo "hello world" > /myvol/greeting
``

创建一个挂载点

### ONBUILD

格式：ONBUILD [INSTRUCTION]

ONBUILD 指令给image加了一个trigger 指令，当这个image被下一个镜像构建当做基础镜像时，这个指令会在下一个镜像构建时执行。即当前镜像不生产，下一个镜像FROM 当前镜像构建时，ONBUILD里的指令会被执行。

### STOPSIGNAL

STOPSIGNAL signal

### HEALTHCHECK

两种格式：

+ HEALTHCHECK [OPTIONS] CMD command (通过在container里运行命令来健康检查)
+ HEALTHCHECK NONE (禁止健康检查)
一般是检查WEB server

在CMD前的[OPTIONS]可以为以下几个：
+ --interval=DURATION (default: 30s)
+ --timeout=DURATION (default: 30s)
+  --retries=N (default: 3)

```
HEALTHCHECK --interval=5m --timeout=3s  CMD curl -f http://localhost/ || exit 1
```

### 几个特殊用法

[参考istio官方构建的](https://github.com/istio/istio/blob/be8d7d33ae674da390f4b850ba2dd0b948cbf7af/samples/jwt-server/src/Dockerfile#L17)

```bash
FROM golang:1.15 as builder
# 可以指定工作目录，该目录如果不存在就会创建
WORKDIR /go/src/istio.io/jwt-server/
# 将该目录下的资源拷贝到WORKDIR工作目录中
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o jwt-server main.go

FROM gcr.io/distroless/static-debian10@sha256:4433370ec2b3b97b338674b4de5ffaef8ce5a38d1c9c0cb82403304b8718cde9 as distroless

WORKDIR /bin/
# copy the jwt-server binary to a separate container based on BASE_DISTRIBUTION
COPY --from=builder /go/src/istio.io/jwt-server .
ENTRYPOINT [ "/bin/jwt-server" ]
EXPOSE 8000
```


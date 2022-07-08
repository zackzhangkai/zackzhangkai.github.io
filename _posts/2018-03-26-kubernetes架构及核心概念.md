---
layout: post
title:  kubernetes架构及核心概念
date:   2018-03-26 01:08:00 +0800
categories: document
tag:
  - kubernetes
---

* content
{:toc}

### kubernetes架构

<img src="{{ '/styles/images/architecture.png' | prepend: site.baseurl }}" alt="kubernetes架构" width="810" />

kubernetes几大核心组件：
+ etcd  保存了整个集群的状态
+ apiserver 提供了资源操作的唯一入口，并提供授权、认证、访问控制、API注册和发现等机制
+ controller manager 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等
+ scheduler 负责资源的调试，按照预定的调试策略将pod调试到相应的机器上
+ kubelet 负责维护容器的生命周期，同时也负责Volume(CVI)和网络（CNI）的管理
+ Container runtime 负责镜像管理以及Pod和容器的真正运行(CRI)
+ Kube-proxy 负责为Service提供cluster内部的服务发现和负载均衡；

除了核心组件，还有一些addons：
+ kube-dns 负责为整个集群提供DNS服务
+ Ingress Controller 为服务提供外网的入口
+ Heapster 提供资源监控
+ Dashboard 提供GUI
+ Federation提供跨可用区的集群

#### 整体架构

下图清晰表明了Kubernetes架构设计及组件之间通信协议

<img src="{{ '/styles/images/kubernetes-high-level-component-archtecture.jpg' | prepend: site.baseurl }}" alt="kubernetes架构" width="810" />

下面这图更抽象

<img src="{{ '/styles/images/kubernetes-whole-arch.png' | prepend: site.baseurl }}" alt="kubernetes架构" width="810" />

master架构

<img src="{{ '/styles/images/kubernetes-master-arch.png' | prepend: site.baseurl }}" alt="kubernetes架构" width="810" />

Node架构

<img src="{{ '/styles/images/kubernetes-node-arch.png' | prepend: site.baseurl }}" alt="kubernetes架构" width="810" />

#### 分层架构
kubernetes设计理念和功能其实就是一个类似linux分层架构

<img src="{{ '/styles/images/kubernetes-layers-arch.jpg' | prepend: site.baseurl }}" alt="kubernetes分层架构示意图" width="810" />

+ 核心层： kubernetes最核心的功能，对外提供API构建高层的应用，对内提供插件式应用执行环境
+ 应用层： 部署（无状态应用、有状态应用、批处理任务、集群应用等）和路由（服务发现、DNS解析等）
+ 管理层： 系统度量（如基础设施、容器和网络度量），自动化（如自动扩展、动态Provision等）以及策略管理（RBAC、Quota、PSP、NetworkPolicy等）
+ 接口层： kubectl命令工具、客户端SDK以及集群联邦
+ 生态系统： 在接口层之上的庞大容器集群管理调度的生态系统，可以划分为两个范畴
  - kubernetes外部：日志、监控、配置管理、CI、CD、Workflow、FaaS、OTS应用、ChatOps等
  - Kubernetes内部： CRI、CNI、CVI、镜像仓库、Cloud Provider、集群自身的配置和管理等

  https://jimmysong.io/kubernetes-handbook/concepts/

### 核心概念和API对象
API对象是Kubernetes集群中的管理操作单元。kubernetes集群系统每引入一项新功能，引入一项新技术，一定会新引入对应API对象，支持对该功能的管理操作。例如副本集Replica Set对应的API对象是RS。

每个API对象都有3大类属性：元数据metadata、规范spec和状态status。元数据是用来标识API对象的，每个对象都至少有3个元数据：namespace,name和uid；除此以外还有各种各样的的标签labels用来标识和匹配不同的对象，例如用户可以用标签env来标识区分不同的服务部署环境，分别用env=dev、env=testing、env=producting来标识开发、测试、生产的不同服务。规范描述了用户期望kubernetes集群中的分布式系统达到的理想状态（Desired State）,例如用户可以通过复制控制器Replication Controller设置的期望的pod副本数为3；status描述了系统实际当前达到的状态（Status）,例如系统当前实际的Pod副本数为2；那么复制控制器当前的程序逻辑就是自动启动新的Pod，争取达到副本数为3。

Kubernetes中所有的配置都是通过API对象的spec去设置的，也就是用户通过配置系统的理想状态来改变系统，这是kubernetes重要设计理念之一，即所有的操作都是声明式（Dclarative）的而不是命令式（Imperative）的。声明式操作在分布式系统中的好处是稳定，不怕丢操作或运行多次，例如设置副本数为3的操作运行多次也还是一个结果，而给副本数加1的操作就不是声明式的，运行多次结果就出错了。

### Pod
kubernetes有很多技术概念，同时对应很多API对象，最重要的也是最基础的是Pod。Pod是在Kubernetes集群中运行部署应用或服务的最小单元，它是可以支持多容器的。Pod的设计理念就是多个容器在一个Pod中共享网络地址和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。Pod对容器的支持是K8最基础的设计理念。比如你运行一个操作系统发行版的软件仓库，一个Nginx容器用来发布软件，另一个容器专门用来从源仓库同步，这两个容器的镜像不大可能是同一个团队开发的，但是他们一块儿工作才能提供一个微服务；这种情况下，不同的团队各自开发构建自己的容器镜像，在部署的时候组合成一个微服务对外提供服务。

Pod是kubernetes集群中所有业务类型的基础，可以看作运行在K8集群中的小机器人，不同类型的业务就需要不同类型的小机器人去执行。目前 kubernetes中的业务主要可以分为长期伺服型(long-running)、批处理型（batch）、节点后台支撑型（node-daemon）和有状态应用型（stateful application）；分别对应的小机器人控制器为Deployment、Job、DaemonSet和PetSet。

### 副本控制器（Replication Controller,RC）
RC是kubernetes集群中最早的保证Pod高可用的API对象。通过监控运行中的Pod来保证集群中运行指定数目的Pod副本。指定的数目可以是多个也可以是1个；少于指定数目时，RC就会启动运行新的Pod副本；多于指定数目时，RC就会杀死多余的Pod副本。即使在指定数目为1的情况下，通过RC运行Pod也比直接运行Pod更明智，因为RC也可以发挥它高可用的能力，保证永远有一个Pod在运行。RC是Kubernetes较早期的技术概念，只适用于长期伺服型的业务类型，比如控制小机器人提供高可以用的Web服务。

### 部署（Deployment）
部署表示用户对Kubernetes集群的一次更新操作。部署是一个比RS应用模式更广的API对象，可以是创建一个新的服务，更新一个新的服务，也可以是滚动升级一个服务。滚动升级一个服务，实际是创建一个新的RS，然后逐渐将新的RS中副本数增加到理想状态，将旧RS中的副本数减小到O的复合操作；这样一个复合操作用一个RS是不太好描述的，所以用一个更通用的Deployment来描述。以Kubernetes的发展方向，未来对所有长期伺服型的业务管理，都会通过Deployment来管理

### 服务（Service）
RC、RS和Deployment只是保证了支撑服务的微服务Pod数量，便是没有解决如何访问这些服务的问题。一个Pod只是一个运行服务的实例，随时可能在一个节点上停止，在另一个节点以一个新的IP启动一个的Pod，因此不能以确定的IP和端口号提供服务。要稳定地提供服务需要服务发现和负载均衡的能力。服务发现完成的工作， 是针对客户端访问的服务，找到对应的后端服务实例。在K8集群中，客户端需要访问的服务就是Service对象。每个Service会对应一个集群内部有效的虚拟IP，集群内部通过虚拟IP访问一个服务。在Kubernetes集群中微服务的负载均衡是由Kube-proxy实现的。Kube-proxy是Kubernetes集群内部的负载均衡器。它是一个分布式代理服务器，在Kubernetes的每个节点上都有一个；这一设计体现了它的伸缩性优势，需要访问服务的节点越多，提供负载均衡能力的Kube-proxy就越多，主可用节点也随之增多。与之相比，我们平时在服务器端做个反向代理做负载均衡，还要进一步解决反向代理的负载均衡和高可用问题。

### 任务（Job）
Job是kubernetes用来控制批处理型任的API对象。批处理业务与长期伺服业务的主要区别是批处理业务的运行有头有尾，而长期伺服业务在用户不停止的情况下永远运行。Job管理的Pod根据用户的设置把任务成功完成后就自动退出了。成功完成的标志根据不同的spec.complietions策略不同：单Pod型任务有一个Pod成功完成就标志完成；定数成功型任务保证有N个任务全部成功；工作队列型任务根据应用确认的全局成功而标志成功。

### 后台支撑服务集（DaemonSet）
长期伺服型和批处理型服务的核心在业务应用，可能有些节点运行多个同类业务的Pod，有些节点上又没有类Pod运行；而后台支撑型服务的核心关注点在Kubernetes集群中的节点（物理机或虚拟机），要保证每个节点上都有一个些类Pod运行。节点可能是所有集群节点也可能是通过nodeSelector选定的一些特定节点。典型的后台支撑型服务包括：存储、日志和监控等在每个节点上支持Kubernetes集群运行的服务。

### 有状态服务集（PetSet）
Kubernetes在1.3版本里发布了Alpha版的PetSet功能。在云原生应用体系里，有下面两组近义词；第一组是无状态（stateless）、牲畜（cattle）、无名（nameless）、可丢弃（disposable）;第二组是有状态（stateful）、宠物（pet）、有名（having name）、不可丢弃（non-disposable）。RC和RS主要是控制提供无状态服务的，其所控制的Pod的名字是随机设置的，一个Pod故障了就被丢弃，在另一个地方重启一个新的Pod，名字变了、名字和启动在哪儿都不重要，重要只是pod总数；而PetSet是用来控制有状态服务，PetSet中的每个Pod的名字都是事先确定的，不能更改。PetSet中的名字的作用，是关联与该pod对应的状态。

对于RC和RS中的Pod，一般不挂载存储或者挂载共享存储，保存的是所有Pod共享状态，Pod像牲畜一样没有分别；对于PetSet中的Pod，每个Pod挂载自己独立的存储，如果一个Pod出现故障，从其他节点启动一个同样名字的Pod，要挂在上原来Pod的存储继续以它的状态提供服务。

适合于PetSet业务包括数据库服务MySQL和Postgre SQL，集群化管理服务Zookeeper、etcd等有状态服务。PetSet的另一种典型应用场景是作为一种比普通容器更稳定可靠的模拟虚拟机的机制。传统的虚拟机正是一种有状态的宠物，运维人员需要不断地维护它，容器刚开始流行时，我们用容器来模拟虚拟机使用，所有状态都保存在这容器里，而这被证明是非常不安全、不可靠的。使用PetSet，Pod仍然可以通过漂移到不同节点提供高可用，而存储也可以通过外挂的存储提供高可靠性，PetSet做的只是将确定的Pod与确定的存储关联起来保证状态的连接性。PetSet还只在Alpha阶段，后面的设计如何演变，我们还要继续观察。

### 集群联邦（Federation）
kubernetes在1.3版本里发布了beta版的Federation功能。在云计算环境中，服务的作用距离范围从近到远一般可以有：同主机（Host、Node）、跨主机同可用区（Available Zone）、跨可用区同地区（Region）、跨地区同服务商（Clound Service Provider）、跨云平台。Kubernetes的设计定位是单一集群在同一个地域内，因为同一个地区的网络性能才能满足Kubernetes的调试和计算存储连接要求。而联合集群服务就是为提供跨Region跨服务商Kubernetes集群服务而设计的。

### 存储卷（Volume）

kubernetes集群中的存储卷跟Docker的存储卷有些类似，只不过Docker的存储卷作用范围为一个容器，而Kuebernetes的存储卷的生命周期和作用范围是一个Pod。每个Pod中声明的存储卷由pod中所有容器共享。Kubernetes支持非常多的存储卷类型，特别的，支持多种公有云平台的存储，包括AWS、Google和Azure云；支持多种分布式存储包括GlusterFS和Ceph；也支持容易使用的主机本地目录hostPath和NFS。kubernetes还支持使用Persistent Volume Claim即PVC这种逻辑卷存储，使用这种存储，使得存储的使用者可以忽略后台的实际存储技术（例如AWS、Google或GlusterFS和Ceph），而将有关存储实际技术的配置交给存储管理员通过Persistent Volume来配置。

### 持久存储卷(Persistent Volume,PV)和持久存储卷声明（Persistent Volume Claim，PVC）

PV和PVC使得Kubernetes集群具备了存储的逻辑抽象能力，使得在配置Pod的逻辑里可以忽略实际后台存储技术的配置，而把这项配置的工作交给PV的配置者，即集群的管理者。存储的PV和PVC的这种关系，跟计算的Node和Pod的关系是非常类似的；PV和Node是资源的提供者，根据集群的基础设施变化而变化，由Kubernetes集群管理员配置；而PVC和Pod是资源的使用者，根据业务服务的需求变化而变化，有Kubernetes集群的使用者即服务的管理员来配置。

### 节点（Node）
Kubernetes集群中的计算能力由Node提供，最初Node称为服务节点Minion，后来改名为Node。kubernetes集群中的Node就等同于Messos集群中的Slave节点，是所有Pod运行所在的工作主机，可以是物理机也可以是虚拟机。不论是物理机还是虚拟机，工作主机的统一特征是上面要运行kubelet，管理节点上运行的容器

### 密钥对象（Secret）
Secret是用来保存和传递密码、密钥、认证凭证这些敏感信息的对象。使用Secret的好处是可以避免把敏感信息明文信息写在配置文件里。在Kubernetes集群中配置和使用服务不可避免的要用到各种敏感信息实现登录、认证等功能，例如访问AWS存储的用户名密码。为了避免将类似的敏感信息明文写在所有需要使用的配置文件中，可以将这些信息存入一个Secret对象，而在配置文件中通过Secret对象引用这些敏感信息。 这种方式的好处包括：意图明确，避免重复，减少暴露机会。

### 用户帐户（User Account）和服务帐户（Service Account）
顾名思义，用户帐户为人提供帐户标识，而服务帐户为计算机进程和Kubernetes集群中运行的Pod提供账户标识。用户账户和服务帐户的一个区别是作用范围；用户帐户对应是人的身份，人的身份与服务的namespace无关，所以用户帐户是跨namespace的；而服务帐户对应的是一个运行中程序身份，与特定namespace是相关的。

### 命名空间（Namespace）
命名空间为Kubernetes集群提供虚拟隔离作用，Kubernetes集群初始有两个命名空间，分别是default和系统命名空间kube-system,除此以外，管理员可以创建新的命名空间满足需要。

### RBAC访问授权

kubernetes在1.3版本中发布了alpha版的基于角色控制（Role-based Access Control,RBAC）的授权模式。相对于基于属性的访问控制（Attribute-based Access Control，ABAC）,RBAC主要是引入了角色（Role）和角色绑定（RoleBinding）的抽象概念。 在ABAC中，Kubernetes集群中访问策略只能跟用户直接关联；而在RBAC中，访问策略可以跟某个角色关联，具体的用户在跟一个或多个角色相关联。显然RBAC像其他新功能一样，每次引入新功能，都会引入新的API对象，从而引入新的概念抽象，而这一新的概念抽象一定会使集群管理和使用更容易扩展和重用。

### 总结
从kubernetes的系统架构、技术概念和设计理念，我们可以看到Kubernetes系统最核心的两个设计理念：一个是容错性、一个是易扩展性。容错性实际是保证Kubernetes系统稳定性和安全性的基础，易扩展性是保证Kubernetes对变更友好，可以快速迭代增加新功能的基础。

按照分布式系统性一致性算法Paxos发明人计算机科学家Leslie Lamport的理念，一个分布式系统有两类特性：安全性Safety和活性Liveness。安全性保证系统的稳定，保证系统不会崩溃，不地邮现业务错误，不会做坏事，是严格约束的；活性使得系统可以提供功能、提高性能、增加易用性，让系统可以在用户“看到时间内”做些好事，是尽力而为的。Kubernetes系统的设计理念正好与Lamport安全性与活性的理念不谋而合，也正是因为Kubernetes在引入功能和技术的时候，非常好的划分了安全性和活性，才可以让Kubernetes能有这么快版本迭代，快速引入RBAC、Federation和PetSet这种新功能。

### 延伸
#### 三个定律
+ 墨菲定律：
+ 二八定律：
+ 康威定律：
#### 分布式架构要素
指标： CAP,即一致性（consistence）(等同于所有节点访问同一份最新的数据副本)、可用性（Availability）（每次请求都能获取到一份非错的响应--但是不保证获取的数据是最新的数据）、分区容错性（Network Partitioning）

性能、伸缩性、扩展性、安全性  ；要在性能确定性用户体验性之间做平衡

服务架构关键词  zookper spring boot  

高性能、并发架构原则  无状态 并行、异步 分级缓存 数据异构 消息队列、消息驱动 服务化 拆分
RPC高性能服务性框架： 第一、服务端尽可能的处理并发请求  第二、同时尽可能短处理请求

高性能服务架构： 同步 异步  IO模型 1、传统阻塞IO  2非阻塞  3IO多路复用，两种IO复用设计模式  同步异步阻塞非阻塞  同步异步关注的是多件事情并发，阻塞与非阻塞关注的是程序等待调用的结果时的状态  

IO逻辑  Read Decode Compute Encode Send  

传统阻塞IO+线程池  即c10k问题; IO多路复用 -> Reactor模式（redis,Nginx,Node.js,Netty）; 异步IO -> Proactor(Windows IOCP)

常见缓存：浏览器缓存http缓存  CDN 反向代理如nginx缓存  分布式缓存redis/memcache   按端划分：客户端缓存、服务端缓存

分布式服务存在的问题：时延问题、事务一致性（一处修改多处生效）、问题bug定位（日志分散）

衡量系统可用性指标SLA:俗称几个9（3分钟定位问题5分钟修复业务2个月一次故障即99.99%），手段：减少故障次数，缩短处理时间；比如我们说月度99.5%的SLA，意味着每个月服务出现故障的时间只能占总时间的0.05%，如果这个月是30天，即21.6分钟

单实例故障、网络故障、应用崩溃

健康心跳，负载均衡

有状态单实例故障，如MYsql主宕

服务集群故障、容量不足、变更引起、网络、  解决方法：限流，按系统负载限流、按业务优先限流；降级，依赖模块的降级、业务功能降级；快速变更：容量伸缩、应用规范、自动部署；网络故障，多机房容灾，主从多活（要考虑同步延时）同城多机房异地容灾

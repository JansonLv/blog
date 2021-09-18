---
title: "K8s"
date: 2021-09-17T23:09:24+08:00
draft: false
categories: ["学习"]
tags: ["技术","容器", "docker", "k8s"]
---
# 容器篇

**## **进程

**进程是分配资源的最小单元**

**容器技术的核心功能，就是通过约束资源和修改进程的动态表现，从而为其创造出一个边界。**

**Cgroups**技术就是用来制造约束的主要手段**
****Namespace**技术则是用来修改进程视图的主要方法**
****rootfs**挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统

# K8S 篇

**k8s：按照用户的意愿和整个系统的规则，完全自动化的处理好容器之间的各种关系。也就是****编排**，本质就是为用户提供一个具有普遍意义的容器编排工具

## 安装：

**使用**[kind](https://kind.sigs.k8s.io/docs/user/quick-start#creating-a-cluster)做k8s安装，也可以使用kubeadm**
**docker内置安装[https://codechina.csdn.net/mirrors/AliyunContainerService/k8s-for-docker-desktop/-/tree/v1.21.3](https://codechina.csdn.net/mirrors/AliyunContainerService/k8s-for-docker-desktop/-/tree/v1.21.3)

**个人推荐docker内置安装，避免翻墙**

## 简单使用（后期补充，都是新手教程）

### 元数据

### 发布一个应用

### 修改一个应用

### 挂载目录

### 进入容器

### 删除一个应用

### 执行一个job

1. **kubectl apply -f xxxxx （任务已存在不可重复执行，需要先delete）**
2. **强制重启kubectl replace –force -f xxxxx**

## Pod

**Pod，其实是一组共享了某些资源的容器， pod里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume。**

### 为什么需要pod

#### pod和容器之间的关系

#### 使用pod的优点

### pod的基础使用

#### 如何理解一切属性、namespace的操作都是pod级别的

#### 相关操作

1. **共享进程（pid namespace）**
2. **Lifecycle 容器的hook**

#### pod对象在k8s中的声明周期

**pod的状态**

### pod 进阶操作

#### Volume 投射数据卷

**用来为容器促提供预制好的数据的，比如配置等**
**常见的Projected Volume一共有四种**

1. **Secret**
2. **ConfigMap**
3. **Downward API**
4. **ServiceAccountToken**

#### 健康检查和恢复机制

#### PodPreset Pod预设值功能（很实用）

### 控制器模型

#### 控制器定义

#### 被控制对象

#### 和实践驱动的区别和练习

### 副本和扩展

**Deployment 控制器实际操纵的，正是这样的 ReplicaSet （副本集）对象，而不是 Pod 对象。**

**对于一个 Deployment 所管理的 Pod，它的 ownerReference 是ReplicaSet**

**一个 ReplicaSet 对象，其实就是由副本数目的定义和一个 Pod 模板组成的。**

![副本集](./01/png)

**可以通过kubectl get rs 查看**

#### 滚动发布

#### 扩展

#### 回滚

#### 回滚指定版本

[金丝雀发布](https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/canary)

### 深入理解StatefulSet（一）：拓扑状态

**两种状态**

1. **拓扑状态：多个应用按照顺序启动，有依赖关系**
2. **存储状态：应用的多个实例分别绑定了不同的存储数据**

**StatefulSet 的核心功能，就是通过某种方式记录这些状态，然后在 Pod 被重新创建时，能够为新 Pod 恢复这些状态。**

**service如何被访问：**

1. **service vip模式：虚拟ip，通过直接访问虚拟ip时，直接转发该service代理的某个pod上**
2. **以service的DNS方式：访问service会访问到其代理的DNS记录， 就可以访问到service所代理的pod**

**StatefulSet 这个控制器的主要作用之一，就是使用 Pod 模板创建 Pod 的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作。而当 StatefulSet 的“控制循环”发现 Pod 的“实际状态”与“期望状态”不一致，需要新建或者删除 Pod 进行“调谐”的时候，它会严格按照这些 Pod 编号的顺序，逐一完成这些操作。**

**所以，StatefulSet 其实可以认为是对 Deployment 的改良。**

**与此同时，通过 Headless Service 的方式，StatefulSet 为每个 Pod 创建了一个固定并且稳定的 DNS 记录，来作为它的访问入口。**

**实际上，在部署“有状态应用”的时候，应用的每个实例拥有唯一并且稳定的“网络标识”，是一个非常重要的假设。**

### 深入理解StatefulSet（二）：存储状态

**Volume之PVC（Persistent Volume Claim）**

**简单来说就是使用PV申明一个类似分布式存储的PersistentVolume pod服务，提供PVC给pod调用**

**PVC 其实就是一种特殊的 Volume。只不过一个 PVC 具体是什么类型的 Volume，要在跟某个 PV 绑定之后才知道。**

**首先，StatefulSet 的控制器直接管理的是 Pod。这是因为，StatefulSet 里的不同 Pod 实例，不再像 ReplicaSet 中那样都是完全一样的，而是有了细微区别的。比如，每个 Pod 的 hostname、名字等都是不同的、携带了编号的。而 StatefulSet 区分这些实例的方式，就是通过在 Pod 的名字里加上事先约定好的编号。**

**其次，Kubernetes 通过 Headless Service，为这些有编号的 Pod，在 DNS 服务器中生成带有同样编号的 DNS 记录。只要 StatefulSet 能够保证这些 Pod 名字里的编号不变，那么 Service 里类似于 web-0.nginx.default.svc.cluster.local 这样的 DNS 记录也就不会变，而这条记录解析出来的 Pod 的 IP 地址，则会随着后端 Pod 的删除和再创建而自动更新。这当然是 Service 机制本身的能力，不需要 StatefulSet 操心。**

**最后，StatefulSet 还为每一个 Pod 分配并创建一个同样编号的 PVC。这样，Kubernetes 就可以通过 Persistent Volume 机制为这个 PVC 绑定上对应的 PV，从而保证了每一个 Pod 都拥有一个独立的 Volume。**

**在这种情况下，即使 Pod 被删除，它所对应的 PVC 和 PV 依然会保留下来。所以当这个 Pod 被重新创建出来之后，Kubernetes 会为它找到同样编号的 PVC，挂载这个 PVC 对应的 Volume，从而获取到以前保存在 Volume 里的数据。**

**从这些讲述中，我们不难看出 StatefulSet 的设计思想：StatefulSet 其实就是一种特殊的 Deployment，而其独特之处在于，它的每个 Pod 都被编号了。而且，这个编号会体现在 Pod 的名字和 hostname 等标识信息上，这不仅代表了 Pod 的创建顺序，也是 Pod 的重要网络标识（即：在整个集群里唯一的、可被的访问身份）。**

**有了这个编号后，StatefulSet 就使用 Kubernetes 里的两个标准功能：Headless Service 和 PV/PVC，实现了对 Pod 的拓扑状态和存储状态的维护。**

**实际上，在下一篇文章的“有状态应用”实践环节，以及后续的讲解中，你就会逐渐意识到，StatefulSet 可以说是 Kubernetes 中作业编排的“集大成者”。**

**因为，几乎每一种 Kubernetes 的编排功能，都可以在编写 StatefulSet 的 YAML 文件时被用到。**

### 深入理解StatefulSet（三）：有状态应用实践

**搭建一主多从的mysql服务**

### 容器化守护进程的意义：DaemonSet

**DaemonSet特征：**

1. **每个节点****必有且只有一个**这样的pod实例
2. **当有新的节点加入集群，该pod就会自动的再新节点上被创建**

**使用场景：**

1. **日志上报**
2. **节点监控**
3. **k8s网络基础服务**

**每个节点上有且只有一个 Pod的保证基础：**
**nodeAffinity**
**Toleration**

### 撬动离线业务：Job与CronJob

**重启策略：restartPolicy**
**重启次数：backoffLimit**
**运行最大时间：activeDeadlineSeconds**

#### job controller并行作业

**parallelism：定义一个job在任意时间最多可以启动多少个pod**
**completions：定义的是job至少要完成的pod数量，即job最小完成数**

#### 三种常用的job用法

1. **最简单粗暴的用法：外部管理器 +Job 模板。（参考KubeFlow）**
2. **拥有固定任务数目的并行 Job**
3. **指定并行度（parallelism），但不设置固定的 completions 的值。**

#### cronjob job重复策略

1. **concurrencyPolicy=Allow，这也是默认情况，这意味着这些 Job 可以同时存在**
2. **concurrencyPolicy=Forbid，这意味着不会创建新的 Pod，该创建周期被跳过**
3. **concurrencyPolicy=Replace，这意味着新产生的 Job 会替换旧的、没有执行完的 Job**

**而如果某一次 Job 创建失败，这次创建就会被标记为“miss”。当在指定的时间窗口内，miss 的数目达到 100 时，那么 CronJob 会停止再创建这个 Job**

### 声明式API与Kubernetes编程范式

**kubectl create，再 replace 的操作，我们称为命令式配置文件操作，替换原有的api对象**

**kubectl apply 声明式 API：是对原有的api对象操作**

1. **指的就是我只需要提交一个定义好的 API 对象来“声明”，我所期望的状态是什么样子。**
2. **“声明式 API”允许有多个 API 写端，以 PATCH 的方式对 API 对象进行修改，而无需关心本地原始 YAML 文件的内容。**
3. **最后，也是最重要的，有了上述两个能力，Kubernetes 项目才可以基于对 API 对象的增、删、改、查，在完全无需外界干预的情况下，完成对“实际状态”和“期望状态”的调谐（Reconcile）过程。**

**声明式 API，才是 Kubernetes 项目编排能力“赖以生存”的核心所在**

**声明式api的好处，是不会重复生成对象，因为很多情况下都需要初始化处理**

#### 深入解析声明式API（一）：API对象的奥秘

**有点难**

#### 深入解析声明式API（二）：编写自定义控制器

**后期使用**

#### RBAC 角色权限控制

**偏于运维，过**

#### 聪明的微创新：Operator工作原理解读

**后面在学吧**

[官方介绍](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/operator/)
[operatorframework官方文档](https://sdk.operatorframework.io/docs/overview/)
[如何从零开始编写一个Kubernetes CRD](https://www.servicemesher.com/blog/kubernetes-crd-quick-start/)
[Kubernetes Operator 快速入门教程](https://www.qikqiak.com/post/k8s-operator-101/)
[知乎：十分钟弄懂 k8s Operator 应用的制作流程](https://zhuanlan.zhihu.com/p/246550722)
[云海天：Operator](https://www.yht7.com/kubernetes/6916.html)
[K8S – 光速理解operator](https://yuerblog.cc/2019/08/13/k8s-%e5%85%89%e9%80%9f%e7%90%86%e8%a7%a3operator/)
[如何在kubernetes中开发自己的Operator](https://www.jianshu.com/p/ad859a279d9a)
[书栈网：KubeOperator v2.0 使用教程](https://www.bookstack.cn/read/KubeOperator-2.0/646ef3fa13db2f0a.md)
[Kubernetes Operator最佳实践](http://weekly.dockerone.com/article/9088)

### 存储卷：PV、PVC、StorageClass，FlexVolume与CSI

**PV：是持久化存储的数据卷。是定义的一个持久化存储在宿主机上的目录，描述。**
**PVC：Pod所希望使用的持久化存储的属性。比如Volume存储的大小，权限等。**

**用go举个例子：PV就是一个对象具体实现了一个方法；pvc就是通过接口签名调用了pv这个方法。**

**这样做的好处是，作为应用开发者，我们只需要跟 PVC 这个“接口”打交道，而不必关心具体的实现是 NFS 还是 Ceph。毕竟这些存储相关的知识太专业了，应该交给专业的人去做。**

**CSI插件编写等**

### 容器网络

**Linux 容器能看见的“网络栈”，实际上是被隔离在它自己的 Network Namespace 当中的。**

**网络栈：网卡（Network Interface）、回环设备（Loopback Device）、路由表（Routing Table）和 iptables 规则。**

**在大多数情况下，我们都希望容器进程能使用自己 Network Namespace 里的网络栈，即：拥有属于自己的 IP 地址和端口。**

**这个被隔离的容器进程，该如何跟其他 Network Namespace 里的容器进程进行交互呢？**

**在 Linux 中，能够起到虚拟交换机作用的网络设备，是网桥（Bridge）。它是一个工作在数据链路层（Data Link）的设备，主要功能是根据 MAC 地址学习来将数据包转发到网桥的不同端口（Port）上。**

**当然，至于为什么这些主机之间需要 MAC 地址才能进行通信，这就是网络分层模型的基础知识了。不熟悉这块内容的读者，可以通过**[这篇文章](https://www.lifewire.com/layers-of-the-osi-model-illustrated-818017)来学习一下。

**为了实现上述目的，Docker 项目会默认在宿主机上创建一个名叫 docker0 的网桥，凡是连接在 docker0 网桥上的容器，就可以通过它来进行通信。**

**可是，我们又该如何把这些容器“连接”到 docker0 网桥上呢？**

**这时候，我们就需要使用一种名叫Veth Pair的虚拟设备了。**

**Veth Pair 设备的特点是：它被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式成对出现的。并且，从其中一个“网卡”发出的数据包，可以直接出现在与它对应的另一张“网卡”上，哪怕这两个“网卡”在不同的 Network Namespace 里。**

**这就使得 Veth Pair 常常被用作连接不同 Network Namespace 的“网线”。**

#### 深入解析容器跨主机网络

**理解容器“跨主通信”的原理，就一定要先从 Flannel 这个项目说起。**

**Flannel 项目是 CoreOS 公司主推的容器网络方案。事实上，Flannel 项目本身只是一个框架，真正为我们提供容器网络功能的，是 Flannel 的后端实现。目前，Flannel 支持三种后端实现，分别是：**

1. **VXLAN**
2. **host-gw**
3. **UDP**

**这三种不同的后端实现，正代表了三种容器跨主网络的主流实现方法。**

**CNI 网络插件**

**Fannel host-gw 模式和 Calico 这两种纯三层网络方案**

**NetworkPolicy：设置流量白名单**

##### 找到容器不容易：Service、DNS与服务发现

**Service** 是由 kube-proxy 组件，加上 iptables 来共同实现的。

**kubeproxy通过组件感知到service对象添加，会在iptables上添加规则（该规则就是为service设置一个固定的入口地址）：目的地址是xxxx，端口是xx的ip包都跳转到另一条iptables链路中处理，这条链路是一组规则的集合，其实也就是代理的pod的最终地址。**

**问题：一直以来，基于 iptables 的 Service 实现，都是制约 Kubernetes 项目承载更多量级的 Pod 的主要障碍。**

**而 IPVS 模式的 Service，就是解决这个问题的一个行之有效的方法。**

**IPVS 模式的工作原理，其实跟 iptables 模式类似。当我们创建了前面的 Service 之后，kube-proxy 首先会在宿主机上创建一个虚拟网卡（叫作：kube-ipvs0），并为它分配 Service VIP 作为 IP 地址。**

**接下来，kube-proxy 就会通过 Linux 的 IPVS 模块，为这个 IP 地址设置三个 IPVS 虚拟主机，并设置这三个虚拟主机之间使用轮询模式 (rr) 来作为负载均衡策略。我们可以通过 ipvsadm 查看到这个设置。**

**相比于 iptables，IPVS 在内核中的实现其实也是基于 Netfilter 的 NAT 模式，所以在转发这一层上，理论上 IPVS 并没有显著的性能提升。但是，IPVS 并不需要在宿主机上为每个 Pod 设置 iptables 规则，而是把对这些“规则”的处理放到了内核态，从而极大地降低了维护这些规则的代价。这也正印证了我在前面提到过的，“将重要操作放入内核态”是提高性能的重要手段。**

**所以，在大规模集群里，我非常建议你为 kube-proxy 设置–proxy-mode=ipvs 来开启这个功能。它为 Kubernetes 集群规模带来的提升，还是非常巨大的。**

**总结**：**
**ClusterIP 模式的 Service 为你提供的，就是一个 Pod 的稳定的 IP 地址，即 VIP。并且，这里 Pod 和 Service 的关系是可以通过 Label 确定的。

**而 Headless Service 为你提供的，则是一个 Pod 的稳定的 DNS 名字，并且，这个名字是可以通过 Pod 名字和 Service 名字拼接出来的。**

#### **从外界连通Service的方式”**

**如何从外部（Kubernetes 集群之外），访问到 Kubernetes 里创建的 Service？**

1. **NodePort**
2. **LoadBalancer**
3. **ExternalName**
4. **ingrss**

**NodePort注意点：一台宿主机上的 iptables 规则，会设置为只将 IP 包转发给运行在这台宿主机上的 Pod。当然，这也就意味着如果在一台宿主机上，没有任何一个被代理的 Pod 存在，比如上图中的 node 2，那么你使用 node 2 的 IP 地址访问这个 Service，就是无效的。此时，你的请求会直接被 DROP 掉。**

**LoadBalancer：适用于公有云上的 Kubernetes 服务，使用了一个叫作 CloudProvider 的转接层，来跟公有云本身的 API 进行对接。所以，在上述 LoadBalancer 类型的 Service 被提交后，Kubernetes 就会调用 CloudProvider 在公有云上为你创建一个负载均衡服务，并且把被代理的 Pod 的 IP 地址配置给负载均衡服务做后端。**

**Ingress：全局的、为了代理不同后端 Service 而设置的负载均衡服务；其实就是 Kubernetes 项目对“反向代理”的一种抽象。可以根据 Ingress 对象和被代理后端 Service 的变化，来自动进行更新的负载均衡器。**

**目前，Ingress 只能工作在七层，而 Service 只能工作在四层。所以当你想要在 Kubernetes 里为应用进行 TLS 配置等 HTTP 相关的操作时，都必须通过 Ingress 来进行。**

### k8s资源模型与资源管理

**在 Kubernetes 中，像 CPU 这样的资源被称作“可压缩资源”（compressible resources）。它的典型特点是，当可压缩资源不足时，Pod 只会“饥饿”，但不会退出。**

**而像内存这样的资源，则被称作“不可压缩资源（incompressible resources）。当不可压缩资源不足时，Pod 就会因为 OOM（Out-Of-Memory）被内核杀掉。**

**而由于 Pod 可以由多个 Container 组成，所以 CPU 和内存资源的限额，是要配置在每个 Container 的定义上的。这样，Pod 整体的资源配置，就由这些 Container 的配置值累加得到。**

**其中，Kubernetes 里为 CPU 设置的单位是“CPU 的个数”。比如，cpu=1 指的就是，这个 Pod 的 CPU 限额是 1 个 CPU。当然，具体“1 个 CPU”在宿主机上如何解释，是 1 个 CPU 核心，还是 1 个 vCPU，还是 1 个 CPU 的超线程（Hyperthread），完全取决于宿主机的 CPU 实现方式。Kubernetes 只负责保证 Pod 能够使用到“1 个 CPU”的计算能力。**

**此外，Kubernetes 允许你将 CPU 限额设置为分数，比如在我们的例子里，CPU limits 的值就是 500m。所谓 500m，指的就是 500 millicpu，也就是 0.5 个 CPU 的意思。这样，这个 Pod 就会被分配到 1 个 CPU 一半的计算能力。**

**你也可以直接把这个配置写成 cpu=0.5。但在实际使用时，我还是推荐你使用 500m 的写法，毕竟这才是 Kubernetes 内部通用的 CPU 表示方式。**

**而对于内存资源来说，它的单位自然就是 bytes。Kubernetes 支持你使用 Ei、Pi、Ti、Gi、Mi、Ki（或者 E、P、T、G、M、K）的方式来作为 bytes 的值。比如，在我们的例子里，Memory requests 的值就是 64MiB (2 的 26 次方 bytes) 。这里要注意区分 MiB（mebibyte）和 MB（megabyte）的区别。**

> **备注：1Mi=1024***1024；1M=1000*1000

**，Kubernetes 里 Pod 的 CPU 和内存资源，实际上还要分为 limits 和 requests 两种情况**

#### k8s默认资源调度器

#### k8s容器运行时

1. **sig-Node**
2. **CRI**

#### k8s容器监控和日志

##### 监控数据分类

**按照 Metrics 数据的来源，来对 Kubernetes 的监控体系分类**

1. **宿主机的监控数据**
2. **Kubernetes 的 API Server、kubelet 等组件的 /metrics API**
3. **Kubernetes 相关的监控数据，包括 Pod、Node、容器、Service 等主要 Kubernetes 核心概念的 Metrics**

**其中，容器相关的 Metrics 主要来自于 kubelet 内置的 cAdvisor 服务。在 kubelet 启动后，cAdvisor 服务也随之启动，而它能够提供的信息，可以细化到每一个容器的 CPU 、文件系统、内存、网络等资源的使用情况。**

##### Custom Metrics

##### 容器日志收集与管理

**方案：**

1. **Node上部署logging agent，将日志文件转发到后端存储里保存。**

**不难看到，这里的核心就在于 logging agent ，它一般都会以 DaemonSet 的方式运行在节点上，然后将宿主机上的容器日志目录挂载进去，最后由 logging-agent 把日志转发出去。**

---

**举个例子，我们可以通过 Fluentd 项目作为宿主机上的 logging-agent，然后把日志转发到远端的 ElasticSearch 里保存起来供将来进行检索。具体的操作过程，你可以通过**[阅读这篇文档](https://kubernetes.io/docs/user-guide/logging/elasticsearch)来了解。另外，在很多 Kubernetes 的部署里，会自动为你启用 [logrotate](https://linux.die.net/man/8/logrotate)，在日志文件超过 10MB 的时候自动对日志文件进行 rotate 操作。

**可以看到，在 Node 上部署 logging agent 最大的优点，在于一个节点只需要部署一个 agent，并且不会对应用和 Pod 有任何侵入性。所以，这个方案，在社区里是最常用的一种。**

**但是也不难看到，这种方案的不足之处就在于，它要求应用输出的日志，都必须是直接输出到容器的 stdout 和 stderr 里。**

2. **针对1的不足，当容器的日志只能输出到某些文件里的时候，我们可以通过一个 sidecar 容器把这些日志文件重新输出到 sidecar 的 stdout 和 stderr 上，这样就能够继续使用第一种方案了**
3. **就是通过一个 sidecar 容器，直接把应用的日志文件发送到远程存储里面去。也就是相当于把方案一里的 logging agent，放在了应用 Pod 里。**

**在这种方案里，你的应用还可以直接把日志输出到固定的文件里而不是 stdout，你的 logging-agent 还可以使用 fluentd，后端存储还可以是 ElasticSearch。只不过， fluentd 的输入源，变成了应用的日志文件。一般来说，我们会把 fluentd 的输入源配置保存在一个 ConfigMap 里**

**总结：**
**建议将应用日志输出到 stdout 和 stderr，然后通过在宿主机上部署 logging-agent 的方式来集中处理日志。**

**这种方案不仅管理简单，kubectl logs 也可以用，而且可靠性高，并且宿主机本身，很可能就自带了 rsyslogd 等非常成熟的日志收集组件来供你使用。**

**除此之外，还有一种方式就是在编写应用的时候，就直接指定好日志的存储后端**

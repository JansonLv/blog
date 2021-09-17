---
title: "容器学习笔记"
date: 2021-09-17T11:07:07+08:00
draft: false
author: JansonLv
cover: /post/xxx-cover.jpg
categories: ["学习"]
tags: ["技术","容器", "docker"]
typora-root-url: ./docker_learn_note
---

# 容器技术

1. 行为算法的区别（容器cpu和宿主机cpu占用率计算方式区别）
2. 隔离程度：cpu，memory，IO
3. 处理性能敏感的应用，需要做cgroup，网络优化等

两个重点：

1. Namespace和Cgroups
2. Namespace和Cgroups对Linux原来模块的影响

## Namespaces

linux创建容器的时候，就会建出PID Namespace，在这个pid namespace中只能看到namespace中的进程，而且看不到其他的namespace里的进程

所以namespace其实就是一个隔离机制，主要目的是隔离运行在同一个宿主机上的容器，让这些容器不能彼此访问彼此的资源

作用：

1. 充分利用系统的资源，也即是同一个宿主机可以运行多个用户的容器
2. 保证安全性，同一个用户之间不嫩个访问对方的资源

除了pid namespace（进程隔离）还有network namespace（网络隔离），mount namespace（文件隔离）统称资源隔离。

其实还有：user(用户和用户组)、uts（主机名与域名），ipc（信号量、消息队列和共享内存）隔离

## Cgroups

对计算机资源的限制，比如cpu的限制，内存的使用量，IO设备的流量。

通过不同的子系统限制不同资源

常用的Cgroups子系统：

1. CPU子系统，用来限制一个控制组（可理解为一个容器内所有进程）可使用的最大CPU
2. memory子系统，限制控制组最大的内存使用量
3. pids子系统，用来限制一个控制组最多可以运行多少进程
4. cpuset子系统：限制控制组的进行可以在几个物理cpu上运行

## 进程信号处理

### 忽略

对信号不做任何处理

### 捕获

用户进程自己注册信号handler

### 缺省

Linux为每个信号都定义了一个缺省的行为，对大部分的信号，用户不需要自己注册handler，使用系统的缺省定义行为即可

> 注意：	kill和stop除外，因为这是内核和超级用户删除任何进程的特权，只能有执行系统处理，用户不能单独处理

### 重点信号

#### SIGTERM

这个信号是kill缺省发出的。可以由用户注册handler去捕获

#### SIGKILL

这个是特权信号，杀死进程

最终可有确定两个

  1. 在容器中是不工作的，内核阻止了 1 号进程对 SIGKILL 特权信号的响应。
   2. kill 分两种情况，如果 1 号进程没有注册 SIGTERM 的 handler，那么对 SIGTERM 信号也不响应，如果注册了 handler，那么就可以响应 SIGTERM 信号。
        3. 在容器中，1号进程永远不会响应SIGKILL和SIGSTOP这两个特权信号。

### 为什么要防止僵尸进程

限制cgroup的进程数量限制，一旦超过进程数量，那么正常进程无法正常启动，且僵尸进程无法被杀死

### 为什么容器中的进程被杀死了

1当docker停止一个容器的时候，containerd服务会向容器的init进程发送一个SIGTERM信号，没有相应即30s之后给当前进程发送SIGKILL，当前进程关闭后给子进程发送SIGKILL。但是SIGKILL是不能捕获的

linux命令：strace -p pid

### 如何限制cpu

关注top下的Cpu

```
$ top
%Cpu(s):  0.1 us,  0.1 sy,  0.0 ni, 99.8 id,  0.0 wa,  0.0 hi,  0.0 si
```

us:  用户程序代码
sy:  系统调用，比如读取文件的系统调用过程
wa:  io等待时间，读取文件时交给了总线DMA操作
id： 空闲状态
hi：cpu硬中断时间
si：software interrupt软件中断，软中断
ni：nice，nice值1-19的进程用户态cpu时间
st：steal，表示同一个宿主机上的其他虚拟机抢走的cpu时间

cpu的cgroup一般限制的是us和ni，还有一部分内核态sy

#### cpu cgroups限制参数

cpu的Cgroups有三个参数

1. cpu.cfs_quota_us：cfs算法的一个调度周期，一般值为100000，也就是100ms
2. cpu.cfs_period_us：cfs算法中在一个调度周期里这个控制组被允许允许时间，比如值是50000，就是50ms
3. cpu.shares：控制组之间的cpu分配比例，缺省值1024

### 如何正确的获取容器cpu开销

首先，在容器中的top中的cpu是反应的宿主机的cpu情况
。。。。。

### 为什么容器很慢

继续top
load average: 系统中可运行队列中进程数量+休眠队列中不可打断的进程平均数
如果load average值升高，且应用性能下降，真正的原因是什么？
	ps aux | grep "D"
这样子可以找到有哪些进程休眠且不可被打断了。
D状态主要是因为disk i/o的党文和信号量的锁党文，因此,D状态在linux中很常见，这是一种对资源的竞争。

D状态引起的容器性能下降问题是CGroup无法解决的，因为资源的竞争是全局的

### 容器为什么被kill掉的

原因：容器中的进程使用了太多的内存，超过了memory cgroup的内存限制，linux就会主动杀死容器内的进程，可能造成整个容器退出
命令：查看容器退出原因 docker inspect 容器id | grep -i Status -A 10

#### memory Cgroups

和cpu的 cgroups一样有三个参数

1. memory.limit_in_bytes：限制控制组中所有进程可使用的内存最大值
2. memory.oom_control：当limit_in_bytes满足时触发的行为，默认OOM Killer
3. memory_usage_in_bytes：当前控制组中所有进程实际使用的内存总和

如何确定容器发生了OOM，可以通过内核日志及时发现

1. 使用journal -k
2. 查看/var/log/message

### 为什么容量总是在使用临界点

涉及到用户态相关的两个内存类型：RSS和Page Cache

1. RSS：进程真正申请到物理页面的内存大小，因为malloc分配的内存是虚拟地址，但没有被使用，其RSS为0，只有使用了才会有物理页面，而且实际使用多少，RSS就是多少。
2. Page Cache：进程对文件读写操作时，系统会分配内存将磁盘上读写到的页面放到内存中的数据。

如果系统中有空闲的内存，系统就会自动把读写过的磁盘文件放到page cache中，如果内存不够了，就会进行内存管理机制进行内存回收。
PageCache只是启动缓存作用，因此会被优先回收释放

	linux命令：查看具体的内存使用量：cat memory.stat

### 容器可以使用Swap空间吗？

本质上是可以使用swap空间的
但是内存泄露的进程没有被杀死，还会不断的读写swap磁盘，影响了整个节点的性能。
最终就是一个平衡点的，参考功能原理：swappiness
swappiness取值范围：
100：脂肪page cache和匿名内存是同等优先级
60：默认值，page cache的释放优先级高于匿名内存的释放
0：当系统中内存低于临界值时，仍然会释放匿名内存并把页面写写入到swap空间


### 在容器中读写文件变慢了

1. 容器文件系统的不支持

####  容器文件系统

	Linux命令：pref

*	减少相同镜像文件在同一个节点的数据冗余们可以节省磁盘空间，也可以减少镜像文件下载的网络资源
*	UnionFS作为容器文件系统，是通过多个目录挂在的方式工作
*	OverlayFS是UnionFS的一种实现，是目前主流Linux版本中的使用的容器文件系统
*	OverlayFS是吧多个目录合并挂载，被挂载的目录分为两大类：lowerdir和upperdir
*	lowerdir允许有多个目录，被挂载之后这些文件是不会被删除和修改的，也就是只读
*	upperdir只有一个，是可读写的，挂载点的目录中所有的文件修改都会在upperdir中反应

###  容器文件Quota：容器为什么把宿主机的磁盘写满了

容器写入的文件都是直接写入到宿主机上的，因此有可能让把宿主机写满

那如何都写入做限流呢？

1. linux都是使用的xfs和ext4文件系统
2. 使用Quota进行对一个用户，一个用户组，或者一个项目限制他们使用文件系统的额度（quota）
3. 具体方案
   4. 给目标目录打赏一个Project ID
   5. 给这个Project ID在XFS文件系统中设置一个写入数据块的限制



Docker就是使用了这个方法，用XFS Quote来限制OverlayFS的upperdir目标，通过这个方式控制同期OverlayFS根目录大小


### 容器里磁盘读写不稳定

具体来说就是多个容器同时读写节点上的同一块磁盘，那么他们的磁盘读写相互之间的影响如何解决。
Linux命令： fio

通过fio命令进行测试，发现多个同期同时写一块磁盘的时候，它的性能受到了干扰。

之前，我们用Cgroups来保证容器的CPU使用率，以及控制memory的可用大小，我们是不是也可以通过Cgroups来保证容器的磁盘读写性能？

在Cgroups v1中有一个blkio子系统，用来限制磁盘IO的

磁盘两个常见指标

1. IOPS：每秒钟磁盘读写次数
2. throughput：每秒钟磁盘数据吞吐量也可以成为带宽
   两者关系是：吞吐量=数据块大小 * IOPS

在blkio Cgroups中有四个主要的参数，用来限制磁盘的io性能

```
blkio.throttle.read_iops_device 
blkio.throrrle.read_bps_device // 吞吐量
blkio.throttle.write_iops_device
blkio.throrrle.write_bps_device
```

但是该模式是Linux系统下的Direct IO模式

Linux有两者文件IO模式：Direct IO和Buffered IO。

DirectIO：文件系统层->块设备层->磁盘驱动->磁盘硬件写入
BufferedIO: 用户进程吧文件数据写入到内存中就返回了，Linux自由的线程会通过DMA模式，写入到磁盘中

DirectIO可以用通过blkio限制磁盘IO，但是bufferedIO不能被限制

如何限制bufferedIO呢？
这就需要Cgroups v2了

1. 第一步打开Cgroups v2的功能，因为限制及时最新的linux系统也是默认关闭的
2. 在进行设（百）置（度）

### 容器中的内存与I/O：容器写文件的延时为什么波动很大？

在Linux默认系统调用下，buffered io是默认模式，使用方便

但是在容器中使用的话，发现多了memory cgroup限制之后，write写相同大小的数据块花费的的时间，延时波动比较大

这是为什么呢？
Linux命令：perf和ftrace

由于这是 Buffered I/O 方式，对于写入文件会先写到内存里，这样就产生了 dirty pages，所以我们先研究了一下 Linux 对 dirty pages 的回收机制是否会影响到容器中写入数据的波动。

在这里我们最主要的是理解这两个参数，dirty_background_ratio 和 dirty_ratio，这两个值都是相对于节点可用内存的百分比值。

当 dirty pages 数量超过 dirty_background_ratio 对应的内存量的时候，内核 flush 线程就会开始把 dirty pages 写入磁盘 ; 当 dirty pages 数量超过 dirty_ratio 对应的内存量，这时候程序写文件的函数调用 write() 就会被阻塞住，直到这次调用的 dirty pages全部写入到磁盘。

在节点是大内存容量，并且 dirty_ratio 为系统缺省值 20%，dirty_background_ratio 是系统缺省值 10% 的情况下，我们通过观察 /proc/vmstat 中的 nr_dirty 数值可以发现，dirty pages 不会阻塞进程的 Buffered I/O 写文件操作。
所以我们做了另一种尝试，使用 perf 和 ftrace 工具对容器中的写文件进程进行 profile。我们用 perf 得到了系统调用 write() 在内核中的一系列子函数调用，再用 ftrace 来查看这些子函数的调用时间。

根据 ftrace 的结果，我们发现写数据到 Page Cache 的时候，需要不断地去释放原有的页面，这个时间开销是最大的。造成容器中 Buffered I/O write() 不稳定的原因，正是容器在限制内存之后，Page Cache 的数量较小并且不断申请释放。

其实这个问题也提醒了我们：在对容器做 Memory Cgroup 限制内存大小的时候，不仅要考虑容器中进程实际使用的内存量，还要考虑容器中程序 I/O 的量，合理预留足够的内存作为 Buffered I/O 的 Page Cache。

比如，如果知道需要反复读写文件的大小，并且在内存足够的情况下，那么 MemoryCgroup 的内存限制可以超过这个文件的大小。

还有一个解决思路是，我们在程序中自己管理文件的 cache 并且调用 Direct I/O 来读写文件，这样才会对应用程序的性能有一个更好的预期。

## 网络

### 容器网络

Network Namespace可以隔离网络设备，ip协议栈，ip路由表，防火墙规则等，以及可以显示独立的网络状态信息

我们可以通过clone()或者unshare()系统调用来简历新的network namespace。

此外，还有一些工具“ip netns”，“unshare”，“lsns”和“nsenter”也可以操作。
![network_工具包](./network_工具包.jpeg)

#### 如何修改容器中的网络参数

由于安全原因，普通容器的/proc/sys/是简历在容器文件系统底层的，是read-only mount的，所以在容器启动以后，我们无法在容器内部修改/proc/sys/net下的网络相关的参数。

如果需要修改，需要通过runC sysctl相关的接口，在容器启动的时候对容器内部的网络参数做配置。

docker：加上--sysctl参数

k8s：需要allowed unsafe sysctl特性

#### 如何配置容器网络接口

且如何在容器不通的时候，如何简单测试

思考下让容器中的数据包最终发送到物理网卡上，需要哪些步骤

1. 数据包从容器的network namespace发送到host network namespace上
2. 数据发到host network namespace之后，还要解决数据包怎么从宿主机上eth0发送出去的问题

先解决第一步从容器到宿主机的network namespace有哪些方式呢？

1. veth
2. macvlan/ipvlan

在docker启动的容器缺省的默认网络接口模式是veth

veth是一个虚拟的网络设备，一般都是成对创建的，而且这对设备是相互连接的，当每个设备在不同的network namespaces的时候，namespace之间就可以用这对veth进行网络通讯了。

使用veth进行容器网络发送到宿主机网络了，之后怎么从宿主机eth0发送出去？

解决方法：

1. nat转发
2. overlay网络发送
3. proxy arp+路由方式

docker默认使用bridge+nat转发

重点是，如果网络不通，使用tcpdump找到具体哪一步数据包停止了转发？
Linux命令：tcpdump

#### 如何优化容器网络时延问题

veth网络接口从配置上看，一个数据包要从容器里发送到宿主机外，需要先从容器里的eth0把包发送到宿主机的veth_host，然后再从宿主机上通过nat或者路由的方式，经过宿主机的eth0向外发送。

![](./veth网络1.jpeg)
这种容器向外发送数据包的路径，相比宿主机直接向外发送数据包的路径，明显多了一次接口层的发送和接收，增加开销



如果程序对网络性能有很高要求，如何解决呢？

可以使用netperf模拟测试下网络迟延问题

Linux命令：netperf



可以换成其他网络噢诶之模式，比如macvlan或者ipvlan

原理：在物理的网络接口上配置几个虚拟的网路接口，配置独立的ip，这些ip可以属于不同的namespace。

不同点：macvlan有独立的mac地址，而ipvlan是所有共享一个mac地址

缺点：由于ipvlan/macvlan网络接口直接挂在物理网络接口上，对于需要使用iptables规则的容器，比如k8s使用service服务的容器就不能工作了。

#### 乱序重传问题

veth接口会不会引起乱序重传？原因是什么？如何解决？
Linux命令：iperf3

数据包重传可能原因：

1. 数据包网络中丢了
2. 数据包乱序

使用tcpdump抓包量比较大，额外系统开销也比较大
可以使用netstat进行查看协议栈中丢包和重传的情况
可以看到 fast retransmits （快速回传）

快速回传：如果发送端收到3个重复的ack，那么发送端就可以立刻重新发送ack对应的下一个数据包，而不用等待发送超时。

通过对veth网络接口研究，我们知道他可能增加数据包乱序的几率。

RPS和RSS作用类似，都是把数据包分散到更多的CPU上进行处理，使得系统有更强的网络报处理能力。他们的区别是RSS工作在网卡的硬件层，而RPS工作在Linux内核的软件层。

在把数据包分散到各个cpu时，RPS保证了同一个数据流是在同一个CPU上的，这样就可以有效减少包的乱序。那么我们可以把RPS的这个特性配置到veth网络接口上，来减少数据包乱序的几率

但是RPS配置还是会带来额外的系统开销，在某些网络环境中会引起softriq CPU使用率的增加大。是否开启需要根据场景进行评估。

同时tcp的乱序包也不一样会产生数据包的回传。想要减少网络包的回传也可以参考协议栈的其他参数设置，如/proc/sys/net/ipv4/tcp_reordering。

另外使用http2.0可以有效减少回传。

### 容器安全

运行容器的时候，在安全方面可以做哪些？

1. 赋予容器合理的capabilities
2. 在容器中以非root用户运行程序

使用系统命令但没有权限，即时以root？为什么？因为执行需要capabilities，只要在启动的时候，需要加上privileged参数

#### Linux capabilities

Linux命令：capsh
capabilities：简单来说对应任意一个进程，在做任意一个特权操作的时候，都需要有这个特权操作对应的capability。就是吧linux root用户原来所有的特权做了细化，可以更加细粒度的给进程赋予不同的权限。

但是也不能直接给容器直接设置成privileged，也就是把所有capability都赋予容器，否则容器就可以直接修改宿主机上的所有文件了。

可以按需给与，比如需要iptables，可以设置 --cap-add NET_ADMIN就可以了

#### root模式

尽管容器中的root用户的linux capabilities已经减少很多，但是在没有没有User Namespace的情况下，容器中的root用户和宿主机上的root用户的uid是完全相同的，一旦有软件的漏洞，容器中的root用户就可以操作整个宿主机。

为了减少安全风险，业界都见识在容器中以非root用户来运行进程，不过在没有 USer Namespace的情况下，容器中使用给root用户，对容器云平台来说，对uid的管理会比较麻烦。

所以，我们分析下User Namespace，它带来的好处有两个。一个是吧容器中的root用户映射成宿主机上的普通用户，另外一个好处是在云平台里对容器的uid分配较为容易。

除了在容器中以非root用户来运行进程外，docker和podman都已经支持rootless container，进一步降低了容器的安全风险。

学习自：极客时间-容器实战高手
待补充实战课

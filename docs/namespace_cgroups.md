# Namespace & Cgroups

> 支撑容器的两大关键技术就是：Namespace 和 Cgroups。

简单来说，容器就是个小工具，可以把应用程序、库文件，配置文件都一起“打包”。然后，在任何一个计算机的节点上，都可以使用这个打好的包。有了容器，一个命令就能把应用程序跑起来，做到了**一次打包，就可以到处使用**。

> 镜像是 Docker 公司的创举，也是一个伟大的发明。它就是一个特殊的文件系统，提供了容器中程序执行需要的所有文件。具体来说，就是应用程序想启动，需要三类文件：相关的程序**可执行文件**、**库文件**和**配置文件**，这三类文件都被容器打包做好了。这样，在容器运行的时候就不再依赖宿主机上的文件操作系统类型和配置了，做到了想在哪个节点上运行，就可以在哪个节点上立刻运行。

从用户使用的角度来看，容器和一台独立的机器或者虚拟机没有什么太大的区别，但是和虚拟机相比，却没有各种复杂的硬件虚拟层，没有独立的 Linux 内核。Namespace 和 Cgroups 可以让程序在一个资源可控的独立（隔离）环境中运行，这就是容器。

## Namespace

[官方文档](https://man7.org/linux/man-pages/man7/namespaces.7.html)

Linux 中所有的 Namespace。

|Namespace|Flag|Page|Isolates|description
|---|---|---|---|---|
|Cgroup|CLONE_NEWCGROUP|cgroup_namespaces|Cgroup root directory|
|IPC|CLONE_NEWIPC|ipc_namespaces|System V IPC, POSIX message queues|
|Network|CLONE_NEWNET|network_namespaces|Network devices, stacks, ports, etc.|负责管理网络环境的隔离
|Mount|CLONE_NEWNS|mount_namespaces|Mount points|管理文件系统的隔离
|PID|CLONE_NEWPID|pid_namespaces|Process IDs|负责隔离不同容器的进程
|Time|CLONE_NEWTIME|time_namespaces|Boot and monotonic clocks|
|User|CLONE_NEWUSER|user_namespaces|User and group IDs|
|UTS|CLONE_NEWUTS|uts_namespaces|Hostname and NIS domain name|

**Namespace 是 Linux 中实现容器的两大技术之一，它最重要的作用是保证资源的隔离。主要目的是隔离运行在同一个宿主机上的容器，让这些容器之间不能访问彼此的资源**。这种隔离有两个作用：

1. 充分利用系统资源，在同一台宿主机上可以运行多个用户的容器；
2. 保证安全性，不同用户之间不能访问对方的资源。

### PID Namespace

![PID Namespace](/resources/pid-namespace.webp)

Linux 在创建容器的时候，就会建出一个 PID Namespace，PID 其实就是进程的编号。

> PID Namespace 是指每建立出一个 Namespace，就会单独对进程进行 PID 编号，每个 Namespace 的 PID 编号都从 1 开始。

1. 在 PID Namespace 中只能看到当前 Namespace 中的进程，而且看不到其他 Namespace 里的进程。
2. 在宿主机上的 Host PID Namespace 是其他 Namespace 的父 Namespace，可以看到在这台机器上的所有进程，不过进程 PID 编号不是 Container PID Namespace 里的编号，而是把所有在宿主机运行的进程放在一起，再进行编号。

### Network Namespace

在 Network Namespace 中都有一套独立的网络接口比如 lo，eth0，还有独立的 TCP/IP 的协议栈配置。

```bash
lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever

eth0@if169: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

### Mount Namespace

容器和宿主机上的根文件系统也是不一样的，**容器中的根文件系统，其实就是我们做的镜像**。Mount Namespace 保证了每个容器都有自己独立的文件目录结构。

## Cgroups

[官方文档](https://man7.org/linux/man-pages/man7/cgroups.7.html)

Cgroup 是一个 Linux 内核特性，对一组进程的资源使用（CPU、内存、磁盘 I/O 和网络等）进行限制、审计和隔离。

Cgroups(Control Groups) 是 linux 内核提供的一种机制，这种机制可以根据需求把一系列系统任务及其子任务整合 (或分隔) 到按资源划分等级的不同组内，从而为系统资源管理提供一个统一的框架。简单说，cgroups 可以限制、记录任务组所使用的物理资源。本质上来说，cgroups 是内核附加在程序上的一系列钩子 (hook)，通过程序运行时对资源的调度触发相应的钩子以达到资源追踪和限制的目的。

> 将容器看作是一台“计算机”，那么这台“计算机”有多少 CPU，有多少 Memory 呢？Linux 如何为这些“计算机”来定义 CPU，定义 Memory 的容量呢？想要定义“计算机”各种容量大小，就涉及到支撑容器的第二个技术 Cgroups （Control Groups），它可以对指定的进程或进程组做各种计算机资源的**限制**，比如限制 CPU 的使用率，内存使用量，IO 设备的流量等等。

Cgroups 通过不同的子系统限制了不同的资源，每个子系统限制一种资源。每个子系统限制资源的方式都是类似的，就是把相关的一组进程分配到一个控制组里，然后通过树结构进行管理，每个控制组都设有自己的资源控制参数。

几种比较常用的 Cgroups 子系统：

- CPU 子系统，限制一个控制组（一组进程，可以理解为一个容器里所有的进程）可使用的最大 CPU。
- memory 子系统，限制一个控制组最大的内存使用量。
- pids 子系统，限制一个控制组里最多可以运行多少个进程。
- cpuset 子系统，限制一个控制组里的进程可以在哪几个**物理 CPU** 上运行。

### Cgroup 和 systemd

docker 默认的 Cgroup Driver 是 cgroupfs：

```shell
$ docker info | grep cgroup
 Cgroup Driver: cgroupfs
```

cgroup 提供了一个原生接口并通过 `cgroupfs` 提供（cgroupfs 就是 Cgroup 的一个接口的封装），类似于 procfs 和 sysfs，是一种虚拟文件系统。并且 cgroupfs 是可以挂载的，默认情况下挂载在 `/sys/fs/cgroup` 目录。

`systemd` 也是对于 cgroup 接口的一个封装。systemd 以 PID1 的形式在系统启动的时候运行，并提供了一套系统管理守护程序、库和实用程序，用来控制、管理 Linux 计算机操作系统资源。

> 当某个 Linux 系统发行版使用 **systemd** 作为其初始化系统时，初始化进程会生成并使用一个 root 控制组（`cgroup`），并充当 cgroup 管理器。Systemd 与 cgroup 集成紧密，并将为每个 systemd 单元分配一个 cgroup。也可以配置容器运行时和 kubelet 使用 `cgroupfs`。连同 systemd 一起使用 `cgroupfs` 意味着将有两个不同的 cgroup 管理器。
>
> 单个 cgroup 管理器将简化分配资源的视图，并且默认情况下将对可用资源和使用中的资源具有更一致的视图。 当有两个管理器共存于一个系统中时，最终将对这些资源产生两种视图。在此领域人们已经报告过一些案例，某些节点配置让 kubelet 和  docker 使用 `cgroupfs`，而节点上运行的其余进程则使用 systemd; 这类节点在资源压力下会变得不稳定。

**注意事项：**不要尝试修改集群里面某个节点的 cgroup 驱动，如果有需要，最好移除该节点重新加入。

### v1 & v2

Cgroups v1 中各种子系统比较独立，每个进程在各个 Cgroups 子系统中独立配置，可以属于不同的 group。

虽然这样比较灵活，但是也存在问题，会导致对同一进程的资源协调比较困难（比如 memory Cgroup 与 blkio Cgroup 之间就不能协作）。

> 在主流的生产环境中，大部分使用的还是 v1。

Cgroups v2 解决了 v1 的问题，使各个子系统可以协调统一地管理资源。

Cgroups v2 在生产环境的应用还很少，因为该版本很多子系统的实现需要较新版本的 Linux 内核，还有无论是主流的 Linux 发行版本还是容器云平台，对 v2 的支持也刚刚起步。

### Memory 子系统

memory 子系统的限制参数最简单，对于启动的每个容器，都会在 Cgroups 子系统下建立一个目录，在 Cgroups 中这个目录也被称作控制组，比如下图里的"`docker-<id1>`"和"`docker-<id2>`"等，设置这个控制组的参数，来限制这个容器的内存资源。

![Memory Cgroups](/resources/memory-cgroup.webp)

> 对于创建的容器，在每个 Cgroups 子系统下，对应这个容器就会有一个目录 `docker-c5a9ff78d9c1……`。

- `cgroup.procs`：保存这个控制组中所有的进程的编号
- `memory.limit_in_bytes`：设置控制组中所有进程内存使用率的总和

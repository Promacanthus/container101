# Namespace & Cgroups

> 支撑容器的两大关键技术就是：Namespace 和 Cgroups。

简单来说，容器就是个小工具，可以把应用程序、库文件，配置文件都一起“打包”。然后，在任何一个计算机的节点上，都可以使用这个打好的包。有了容器，一个命令就能把应用程序跑起来，做到了**一次打包，就可以到处使用**。

> 镜像是 Docker 公司的创举，也是一个伟大的发明。它就是一个特殊的文件系统，提供了容器中程序执行需要的所有文件。具体来说，就是应用程序想启动，需要三类文件：相关的程序**可执行文件**、**库文件**和**配置文件**，这三类文件都被容器打包做好了。这样，在容器运行的时候就不再依赖宿主机上的文件操作系统类型和配置了，做到了想在哪个节点上运行，就可以在哪个节点上立刻运行。

从用户使用的角度来看，容器和一台独立的机器或者虚拟机没有什么太大的区别，但是和虚拟机相比，却没有各种复杂的硬件虚拟层，没有独立的 Linux 内核。Namespace 和 Cgroups 可以让程序在一个资源可控的独立（隔离）环境中运行，这就是容器。

## Namespace

[官方文档](https://man7.org/linux/man-pages/man7/namespaces.7.html)

Linux 中所有的 Namespace，常用的 Namespace API，主要是和进程相关的系统调用函数。

- `clone()`：用于创建新进程，通过传入一个或多个系统调用参数（ flags 参数）可以创建出不同类型的 NameSpace ，并且子进程也将会成为这些 NameSpace 的成员。
- `setns(int fd, int nstype)`： 用于将进程加入到一个现有的 Namespace 中。其中 fd 为文件描述符，引用 `/proc/[pid]/ns/` 目录里对应的文件，nstype 代表 NameSpace 类型。
- `unshare(int flags)`：用于将进程移出原本的 NameSpace ，并加入到新创建的 NameSpace 中。同样是通过传入一个或多个系统调用参数（ flags 参数）来创建新的 NameSpace 。
- `ioctl()`：用于发现有关 NameSpace 的信息。

|Namespace|Flag|Page|Isolates|description|
|---|---|---|---|---|
|Control Group(Cgroup) Namespace|CLONE_NEWCGROUP|cgroup_namespaces|Cgroup root directory|Cgroup 信息隔离。用于隐藏进程所属的控制组的身份，使命名空间中的 cgroup 视图始终以根形式来呈现，保障安全|
|Interprocess Communication<br />(IPC)|CLONE_NEWIPC|ipc_namespaces|System V IPC, POSIX message queues|进程 IPC 通信隔离，让只有相同 IPC 命名空间的进程之间才可以共享内存、信号量、消息队列通信|
|Network<br />(net)|CLONE_NEWNET|network_namespaces|Network devices, stacks, ports, etc.|负责管理网络环境的隔离，使每个 net 命名空间有独立的网络设备，IP 地址，路由表，`/proc/net` 目录等网络资源|
|Mount<br />(mnt)|CLONE_NEWNS|mount_namespaces|Mount points|管理文件系统的隔离，隔离各个进程看到的挂载点视图|
|Process ID<br />(pid)|CLONE_NEWPID|pid_namespaces|Process IDs|负责隔离不同容器的进程，使每个命名空间都有自己的初始化进程，PID 为 1，作为所有进程的父进程|
|Time Namepsace|CLONE_NEWTIME|time_namespaces|Boot and monotonic clocks|系统时间隔离。允许不同进程查看到不同的系统时间|
|User ID<br />(user)|CLONE_NEWUSER|user_namespaces|User and group IDs|用户 UID 和组 GID 隔离。例如每个命名空间都可以有自己的 root 用户|
|UTS|CLONE_NEWUTS|uts_namespaces|Hostname and NIS domain name|主机名或域名隔离。使其在网络上可以被视作一个独立的节点而非主机上的一个进程|

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

Cgroup 是一个 Linux 内核特性，对一组进程的资源使用（CPU、内存、磁盘 I/O 和网络等）进行限制、审计和隔离。Linux 内核提供的这种 Cgroups 的机制可以根据需求把一系列系统任务及其子任务整合 (或分隔) 到按资源划分等级的不同组内，从而为系统资源管理提供一个统一的框架。简单说，cgroups 可以限制、记录任务组所使用的物理资源。本质上来说，cgroups 是内核附加在程序上的一系列钩子 (hook)，通过程序运行时对资源的调度触发相应的钩子以达到资源追踪和限制的目的。

> 将容器看作是一台“计算机”，那么这台“计算机”有多少 CPU，有多少 Memory 呢？Linux 如何为这些“计算机”来定义 CPU，定义 Memory 的容量呢？想要定义“计算机”各种容量大小，就涉及到支撑容器的第二个技术 Cgroups （Control Groups），它可以对指定的进程或进程组做各种计算机资源的**限制**，比如限制 CPU 的使用率，内存使用量，IO 设备的流量等等。

Cgroups 通过不同的子系统限制了不同的资源，每个子系统限制一种资源。每个子系统限制资源的方式都是类似的，就是把相关的一组进程分配到一个控制组里，然后通过树结构进行管理，每个控制组都设有自己的资源控制参数。

在 Linux Kernel 中，为了让 Cgroups 的配置更直观，使用了目录的层级关系来模拟 hierarchy ，以此通过虚拟的树状文件系统的方式暴露给用户调用。系统已经默认为每个 subsystem 创建了一个默认的 hierarchy ，我们可以直接使用。

几种比较常用的 Cgroups 子系统：

| 子系统   | 作用                                                         | 目录                    |
| -------- | ------------------------------------------------------------ | ----------------------- |
| cpu      | 限制一个控制组（一组进程，可以理解为一个容器里所有的进程）可使用的最大 CPU |                         |
| cpuacct  | 统计一个控制组的 CPU 使用情况                                |                         |
| cpuset   | 在多核机器上为一个控制组分配单独的 CPU 节点或者内存节点（仅限 NUMA 架构） |                         |
| memory   | 限制一个控制组最大的内存使用量                               | `/sys/fs/cgroup/memory` |
| blkio    | 限制一个控制组对块设备（例如硬盘） io 的访问                 |                         |
| devices  | 限制一个控制组对设备的访问                                   |                         |
| net_cls  | 标记进程的网络数据包，以便可以使用 tc 模块（traffic control）对数据包进行限流、监控等控制 |                         |
| net_prio | 限制一个控制组产生的网络流量的优先级                         |                         |
| freezer  | 挂起或者恢复进程                                             |                         |
| pids     | 限制一个控制组里最多可以运行多少个进程                       |                         |

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

### [Memory 子系统](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt)

memory 子系统的限制参数最简单，对于启动的每个容器，都会在 Cgroups 子系统下建立一个目录，在 Cgroups 中这个目录也被称作控制组，比如下图里的"`docker-<id1>`"和"`docker-<id2>`"等，设置这个控制组的参数，来限制这个容器的内存资源。

![Memory Cgroups](/resources/memory-cgroup.webp)

> 对于创建的容器，在每个 Cgroups 子系统下，对应这个容器就会有一个目录 `docker-c5a9ff78d9c1……`。

在内存子系统中创建文件夹，文件夹内的所有文件都是系统自动创建的。常用的几个文件功能如下：

| 文件名                          | 功能                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| tasks                           | cgroup 中运行的进程（ PID）列表。将 PID 写入一个 cgroup 的 tasks 文件，可将此进程移至该 cgroup |
| cgroup.procs                    | cgroup 中运行的线程群组列表（ TGID ）。将 TGID 写入 cgroup 的 cgroup.procs 文件，可将此线程组群移至该 cgroup |
| cgroup.event_control            | event_fd() 的接口。允许 cgroup 的变更状态通知被发送          |
| notify_on_release               | 用于自动移除空 cgroup 。默认为禁用状态（0）。设定为启用状态（1）时，当 cgroup 不再包含任何任务时（即，cgroup 的 `tasks` 文件包含 PID，而 PID 被移除，致使文件变空），kernel 会执行 `release_agent` 文件（仅在 root cgroup 出现）的内容，并且提供通向被清空 cgroup 的相关路径（与 root cgroup 相关）作为参数 |
| memory.usage_in_bytes           | 显示 cgroup 中进程当前所用的内存总量（以字节为单位）         |
| memory.memsw.usage_in_bytes     | 显示 cgroup 中进程当前所用的内存量和 swap 空间总和（以字节为单位） |
| memory.max_usage_in_bytes       | 显示 cgroup 中进程所用的最大内存量（以字节为单位）           |
| memory.memsw.max_usage_in_bytes | 显示 cgroup 中进程的最大内存用量和最大 swap 空间用量（以字节为单位） |
| memory.limit_in_bytes           | 设定用户内存（包括文件缓存）的最大用量                       |
| memory.memsw.limit_in_bytes     | 设定内存与 swap 用量之和的最大值                             |
| memory.failcnt                  | 显示内存达到 `memory.limit_in_bytes` 设定的限制值的次数      |
| memory.memsw.failcnt            | 显示内存和 swap 空间总和达到 `memory.memsw.limit_in_bytes` 设定的限制值的次数 |
| memory.oom_control              | 可以为 cgroup 启用或者禁用“内存不足”（Out of Memory，OOM） 终止程序。默认为启用状态（0），尝试消耗超过其允许内存的任务会被 OOM 终止程序立即终止。设定为禁用状态（1）时，尝试使用超过其允许内存的任务会被暂停，直到有额外内存可用。 |


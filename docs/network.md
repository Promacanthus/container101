# 容器网络

很大一部分网络参数都在 `/proc` 文件系统下的 [`/proc/sys/net/`](https://www.kernel.org/doc/Documentation/sysctl/net.txt) 目录里，修改这些参数主要有两种方法：

1. 直接到 `/proc` 文件系统下的 "`/proc/sys/net/`" 目录里对参数做修改
2. 使用 [`sysctl`](https://man7.org/linux/man-pages/man8/sysctl.8.html) 工具来修改

同样的容器内也有这些网络参数，但是没有 privileged 权限的普通容器内是无法修改这些内核参数的，因为容器内的 `/proc/sys` 是read-only mount。

通常容器已经启动的话，很多 TCP 连接已经建立，即使修改参数也不会生效，因此正确的修改时机是**在容器启动的时候**。[runC](https://github.com/opencontainers/runc) 的 sysctl 参数修改接口，允许容器在启动时修改容器 Namespace 里的参数，因为无论是 Docker 或 Containerd 都是调用的 runC 在Linux 中把容器启动起来。

- Docker：使用 [--sysctl](https://docs.docker.com/engine/reference/commandline/run/#configure-namespaced-kernel-parameters-sysctls-at-runtime) 参数，如 `docker run -d --name net_para --sysctl net.ipv4.tcp_keepalive_time=600 centos:8.1.1911 sleep 3600`
- Kubernetes：使用 [allowed-unsafe-sysctls](https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/) 特性

> 在容器内能看到的网络参数都是 network namespace 下，而不是宿主机级别的。runC 只允许修改 namesapce 下的参数，不会修改宿主机级别的参数。

## Network Namespace

[Network Namespace](https://man7.org/linux/man-pages/man7/network_namespaces.7.html) 负责管理 Linux 节点上网络环境的隔离，主要包括：

1. **网络设备**，这里指的是 lo，eth0 等网络设备，通过 `ip link` 命令看到它们。
2. **IPv4/IPv6 协议栈**，也就是说 IP 层以及上面的 TCP 和 UDP 协议栈也是每个 Namespace 独立工作的。所以 IP、TCP、UDP 的很多协议，它们的相关参数也是每个 Namespace 独立的，这些参数大多数都在 `/proc/sys/net/` 目录下面，同时也包括了 TCP 和 UDP 的 port 资源。
3. **IP 路由表**，在不同的 Network Namespace 运行 `ip rout`e 命令，就能看到不同的路由表。
4. **防火墙规则**，也就是 iptables 规则，每个 Namespace 里都可以独立配置 iptables 规则。
5. **网络状态信息**，这些信息可以从 `/proc/net` 和 `/sys/class/net` 里得到，这里的状态基本上包括了前面 4 种资源的状态信息。

**通过系统调用 `clone()` 或者 `unshare()` 这两个函数可以建立新的 Network Namespace。**具体的操作如下：

1. 在新的进程创建的时候，伴随新进程建立，同时也建立出新的 Network Namespace。通过 `clone()` 系统调用带上 `CLONE_NEWNET` flag 来实现的，如 `clone(new_netns, stack + STACK_SIZE, CLONE_NEWNET | SIGCHLD, NULL)`。`Clone()` 建立出来一个新的进程，这个新的进程所在的 Network Namespace 也是新的。
2. 调用 `unshare()` 系统调用来直接改变当前进程的 Network Namespace，如 `unshare(CLONE_NEWNET)` 。

## 容器网络接口

如下图所示：

- 容器有自己的 Network Namespace，eth0 是这个 Network Namespace 里的网络接口。
- 宿主机也有自己的 eth0，这是真正的物理网卡，可以和外面通讯。

![network-interface](/resources/newowrk-namespace-interface.webp)

要让容器 Network Namespace 中的数据包最终发送到物理网卡上，大致应该包括这两步：

1. 让数据包从容器的 Network Namespace 发送到 Host Network Namespace 上。
2. 数据包发到了 Host Network Namespace 之后，还要解决数据包怎么从宿主机上的 eth0 发送出去的问题。

### 数据包从容器到宿主机

在 [Docker 网络文档](https://docs.docker.com/network/)或者 [Kubernetes 网络文档](https://kubernetes.io/docs/concepts/cluster-administration/networking/)中介绍了很多种容器网络配置的方式。对于容器从自己的 Network Namespace 连接到 Host Network Namespace 的方法，一般来说就只有两类设备接口：

1. [veth](https://man7.org/linux/man-pages/man4/veth.4.html)：是一个虚拟的网络设备，一般都是成对创建，而且这对设备是相互连接的。当每个设备在不同的 Network Namespaces 的时候，Namespace 之间就可以用这对 veth 设备来进行网络通讯。这是使用最多的方式，Docker 启动容器缺省的网络接口，主要通过 [`ip netns`](https://man7.org/linux/man-pages/man8/ip-netns.8.html) 命令对 network namespace 操作。
2. macvlan/ipvlan：无论是 macvlan 还是 ipvlan，都是在一个物理的网络接口上再配置几个虚拟的网络接口。在这些虚拟的网络接口上，都可以配置独立的 IP，并且这些 IP 可以属于不同的 Namespace。容器的虚拟网络接口，直接连接在了宿主机的物理网络接口上了，形成了一个网络二层的连接。
   1. macvlan：每个虚拟网络接口都有自己独立的 mac 地址
   2. ipvlan ：虚拟网络接口是和物理网络接口共享同一个 mac 地址

### 数据包从宿主机 eth0 发送出去

这就是一个普通 Linux 节点上数据包转发的问题。这里解决问题的方法有很多种：

1. 用 nat 来做个转发
2. 建立 Overlay 网络发送
3. 通过配置 proxy arp 加路由的方法来实现

考虑到网络环境的配置，Docker 缺省使用的是 bridge + nat 的转发方式， 对于其他的配置方法，可以查阅 Docker 或者 Kubernetes 相关的文档。

Docker 程序在节点上安装完之后，就会自动建立了一个 docker0 的 bridge interface。所以 veth_host 这个设备会接入到 docker0 这个 bridge 上，这样容器和 docker0 组成了一个子网，docker0 上的 IP 就是这个子网的网关 IP。要让子网通过宿主机上 eth0 去访问外网，加上 iptables 的规则（`iptables -P FORWARD ACCEPT`）同时配置两个网络设备接口 docker0 和 eth0 之间的数据包转发（`echo 1 > /proc/sys/net/ipv4/ip_forward`）就可以了。

## 容器网络接口对网络的延时的影响

从 veth 的这种网络接口配置上看，一个数据包要从容器里发送到宿主机外，需要先从容器里的 eth0 (veth_container) 把包发送到宿主机上 veth_host，然后再在宿主机上通过 nat 或者路由的方式，经过宿主机上的 eth0 向外发送。

这种容器向外发送数据包的路径，相比宿主机上直接向外发送数据包的路径，很明显要多了一次接口层的发送和接收。尽管 veth 是虚拟网络接口，在软件上还是会增加一些开销。

虽然 veth 是一个虚拟的网络接口，但是在接收数据包的操作上，这个虚拟接口和真实的网络接口并没有太大的区别。除了没有硬件中断的处理，其他操作都差不多，特别是软中断（softirq）的处理部分其实就和真实的网络接口是一样的，可以通过阅读 Linux 内核里的 veth 的驱动代码（[drivers/net/veth.c](drivers/net/veth.c)）确认。

> 在处理网络数据时，一些运行时间较长且不能在硬中断中处理的工作，就会通过 softirq 来处理。一般在硬件中断处理结束之后，网络 softirq 的函数才会再去执行没有完成的包的处理工作。即使这里 softirq 的执行速度很快，还是会带来额外的开销。

由于 veth 接口是成对工作，在对外发送数据的时候，peer veth 接口都会 raise softirq 来完成一次收包操作，这样就会带来数据包处理的额外开销。

如果要减小容器网络延时，就可以给容器配置 ipvlan/macvlan 的网络接口来替代 veth 网络接口。Ipvlan/macvlan 直接在物理网络接口上虚拟出接口，在发送对外数据包的时候可以直接通过物理接口完成，没有节点内部类似 veth 的那种 softirq 的开销。**容器使用 ipvlan/maclan 的网络接口，它的网络延时可以非常接近物理网络接口的延时。**

**所以，根据 veth 这个虚拟网络设备的实现方式，可以看到它必然会带来额外的开销，这样就会增加数据包的网络延时。**对于网络延时敏感的应用程序，可以考虑使用 ipvlan/macvlan 的容器网络配置方式来替换缺省的 veth 网络配置。

> 对于延时敏感的应用程序，我们可以考虑使用 ipvlan/macvlan 网络接口的容器。不过，由于 ipvlan/macvlan 网络接口直接挂载在物理网络接口上，对于需要使用 iptables 规则的容器，比如 Kubernetes 里使用 service 的容器，就不能工作了。这就需要你结合实际应用的需求做个判断，再选择合适的方案。

## 容器中网络乱序包

网络中发生数据包的重传，有可能是数据包在网络中丢失，也有可能是数据包乱序导致的。

最直接的方法就是用 `tcpdump` 去抓包，不过对于大流量的网络，瞬间就会有几个 GB 的数据。这样会带来额外的系统开销，特别是在生产环境中这个方法也不太好用。

所以更简单的方法是运行 `netstat` 命令来查看协议栈中的丢包和重传的情况。

### TCP 快速重传

运行 netstat 命令后，查看“retransmit”，能看到有快速重传（fast retransmit）的次数。

```shell
nsenter -t 51598 -n netstat -s | grep retran
    454 segments retransmited
    442 fast retransmits
```

TCP 协议里，发送端（sender）向接受端（receiver）发送一个数据包，接受端（receiver）都回应 ACK。如果超过一个协议栈规定的时间（RTO），发送端没有收到 ACK 包，那么发送端就会重传（Retransmit）数据包。这样等待一个超时之后再重传数据，对于实际应用来说太慢了，所以 TCP 协议定义了快速重传 （fast retransmit）的概念。它的基本定义是这样的：**如果发送端收到 3 个重复的 ACK，那么发送端就可以立刻重新发送 ACK 对应的下一个数据包。**

如下图所示，接受端没有收到 Seq 2 这个包，但是收到了 Seq 3–5 的数据包，那么接收端在回应 Ack 的时候，Ack 的数值只能是 2。这是因为按顺序来说收到 Seq 1 的包之后，后面 Seq 2 一直没有到，所以接收端就只能一直发送 Ack 2。那么当发送端收到 3 个重复的 Ack 2 后，就可以马上重新发送 Seq 2 这个数据包了，而不用再等到重传超时之后了。

![fast-retransmit](/resources/fast-retransmit.webp)

虽然 TCP 快速重传的标准定义是需要收到 3 个重复的 Ack，但在 Linux 中常常收到一个 Dup Ack（重复的 Ack）后，就马上重传数据了。因为 SACK 也就是选择性确认（Selective Acknowledgement），与普通的 ACK 相比呢，**SACK 会把接收端收到的所有包的序列信息，都反馈给发送端**。对于发送端来说，在收到 SACK 之后就已经知道接收端收到了哪些数据，没有收到哪些数据。

> 在 Linux 内核中会有个判断，如果在接收端收到的数据和还没有收到的数据之间，两者数据量差得太大的话（超过了 `reordering*mss_cache`），就可以马上重传数据。**这里的数据量差是根据 bytes 来计算的，而不是按照包的数目来计算的，所以即使只收到一个 SACK，Linux 也可以重发数据包。**

运行 netstat 命令后，查看"reordering"，就可以看到大量的 SACK 发现的乱序包。

```shell
nsenter -t 51598 -n netstat -s  | grep reordering
    Detected reordering 501067 times using SACK
```

**在云平台的网络环境里，网络包乱序 +SACK 之后，产生的数据包重传的量要远远高于网络丢包引起的重传。**

### veth 接口的数据包发送

网络包乱序会造成数据包的重传，容器的 veth 接口配置可能会引起数据包的乱序：

1. 通过 veth 接口从容器向外发送数据包，会触发 peer veth 设备去接收数据包，这个接收的过程就是一个网络的 softirq 的处理过程。
2. 在触发 softirq 之前，veth 接口会模拟硬件接收数据的过程，通过 `enqueue_to_backlog()` 函数把数据包放到某个 CPU 对应的数据包队列里（softnet_data）。
3. 在缺省的状况下（也就是没有 RPS 的情况下），`enqueue_to_backlog()` 把数据包放到了“当前运行的 CPU”（`get_cpu()`）对应的数据队列中。如果是从容器里通过 veth 对外发送数据包，那么这个“当前运行的 CPU”就是容器中发送数据的进程所在的 CPU。

对于多核的系统，这个发送数据的进程可以在多个 CPU 上切换运行。进程在不同的 CPU 上把数据放入队列并且 raise softirq 之后，因为每个 CPU 上处理 softirq 是个异步操作，所以两个 CPU network softirq handler 处理这个进程的数据包时，处理的先后顺序并不能保证。所以，**veth 对的这种发送数据方式增加了容器向外发送数据出现乱序的几率。**

![softnet-data](/resources/softnet-data.webp)

### RSS 和 RPS

veth 接口模拟硬件接收数据的过程，在调用 `enqueue_to_backlog()` 的时候，传入的 CPU 并不是当前运行的 CPU，而是通过 `get_rps_cpu()` 得到的 CPU，这个 RSS 不是内存 RSS，而是和网卡硬件相关的一个概念。

> RSS 是 Receive Side Scaling 的缩写。
>
> 现在的网卡性能越来越强劲，从原来一条 RX 队列扩展到了 N 条 RX 队列，而网卡的硬件中断也从一个硬件中断，变成了每条 RX 队列都会有一个硬件中断。
>
> 每个硬件中断可以由一个 CPU 来处理，对于多核的系统，多个 CPU 可以并行的接收网络包，这样就大大地提高了系统的网络数据的处理能力。

在网卡硬件中，可以根据数据包的 4 元组或者 5 元组信息来保证同一个数据流，比如一个 TCP 流的数据始终在一个 RX 队列中，这样也能保证同一流不会出现乱序的情况。

![receive-side-scaling](/resources/receive-side-scaling.webp)

RSS 的实现在网卡硬件和驱动里面，而 RPS（Receive Packet Steering）就是在软件层面实现类似的功能。它主要实现的代码框架就在 `netif_rx_internal()` 代码里。

如下图所示：

1. 在硬件中断后，CPU2 收到了数据包，再一次对数据包计算一次四元组的 hash 值，得到这个数据包与 CPU1 的映射关系。
2. 接着会把这个数据包放到 CPU1 对应的 softnet_data 数据队列中，同时向 CPU1 发送一个 IPI 的中断信号。这样一来，后面 CPU1 就会继续按照 Netowrk softirq 的方式来处理这个数据包了。

![receive-packet-steering](/resources/receive-packet-steering.webp)

RSS（工作在网卡的硬件层） 和 RPS（工作在 Linux 内核的软件层） 的目的都是把数据包分散到更多的 CPU 上进行处理，使得系统有更强的网络包处理能力。在把数据包分散到各个 CPU 时，保证了同一个数据流在一个 CPU 上，这样就可以减少包的乱序。

如果对应的 veth 接口上打开了 RPS 的配置，那么对于同一个数据流，就可以始终选择同一个 CPU 了。

在 `/sys` 目录下，在网络接口设备接收队列中修改队列里的 `rps_cpus` （是一个 16 进制的数，每个 bit 代表一个 CPU） 的值，就可以开启 RPS。

比如说，在一个 12 个 CPU 的节点上，想让 host 上的 veth 接口在所有的 12 个 CPU 上，都可以通过 RPS 重新分配数据包。那么就可以执行下面这段命令：

```shell

cat /sys/devices/virtual/net/veth57703b6/queues/rx-0/rps_cpus
000

echo fff > /sys/devices/virtual/net/veth57703b6/queues/rx-0/rps_cpus
cat /sys/devices/virtual/net/veth57703b6/queues/rx-0/rps_cpus
fff
```


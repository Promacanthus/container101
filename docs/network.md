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
2. macvlan/ipvlan

### 数据包从宿主机 eth0 发送出去

这就是一个普通 Linux 节点上数据包转发的问题。这里解决问题的方法有很多种：

1. 用 nat 来做个转发
2. 建立 Overlay 网络发送
3. 通过配置 proxy arp 加路由的方法来实现

考虑到网络环境的配置，Docker 缺省使用的是 bridge + nat 的转发方式， 对于其他的配置方法，可以查阅 Docker 或者 Kubernetes 相关的文档。

Docker 程序在节点上安装完之后，就会自动建立了一个 docker0 的 bridge interface。所以 veth_host 这个设备会接入到 docker0 这个 bridge 上，这样容器和 docker0 组成了一个子网，docker0 上的 IP 就是这个子网的网关 IP。要让子网通过宿主机上 eth0 去访问外网，加上 iptables 的规则（`iptables -P FORWARD ACCEPT`）同时配置两个网络设备接口 docker0 和 eth0 之间的数据包转发（`echo 1 > /proc/sys/net/ipv4/ip_forward`）就可以了。


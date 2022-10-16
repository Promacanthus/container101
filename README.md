# Container 101

容器是一种轻量级的隔离技术，这种轻量级隔离造成了一些行为模式的不同，比如原来运行在虚拟机里的 CPU 监控程序，移到容器之后，再用原来的算法计算容器 CPU 使用率就不适用了。

> 容器技术的代表之作 Docker ，是一个基于 Linux 操作系统，使用 Go 语言编写，调用了 Linux Kernel 功能的虚拟化工具。

从隔离程度这个方面考虑，CPU、memory、IO （disk and network）真的能做到精确隔离吗？

- 对于性能敏感的应用。容器技术会带来新的开销必然会影响性能。需要对容器网络和 Cgroup 做优化才能保证迁移过来的程序，运行在容器中时，性能差异控制在 2% 以内（例如以 2% 作为迁移的标准）。
- 对于高内存使用的应用。需要考虑 PageCache、Swap，还有 HugePage 等问题，在叠加了 Cgroup 之后，会带来新的变化。

**容器问题虽然有很多类型，既有基本功能问题，也有性能问题，还有稳定性问题。但大部分问题最终都会归结到 Linux 操作系统上。**

> Linux 操作系统不外乎是进程管理、内存管理、文件系统、网络协议栈，再加上一些安全管理。结合 Linux 操作系统的主要模块，把容器的知识结构系统地串联起来，同时看到 Namespace 和 Cgroups 带来的特殊性是如何影响传统操作系统的行为。

![容器](/resources/%E5%AE%B9%E5%99%A8.png)

**容器是一个很好的技术窗口，可以帮助你在瞬息万变的计算机世界里看到后面那些“不变”的技术，只有掌握好那些“不变”的技术，才可以更加从容地去接受技术的瞬息万变。**

容器化带来的好处是，开发效率的提高，资源利用率的提高。

## Docker

Docker Daemon 守护进程一直都是 CS 架构，守护进程负责和 Docker Client 端交互，并管理 Docker 镜像和容器。现在的架构中组件 containerd 就会负责集群节点上容器的生命周期管理，并向上为 Docker Daemon 提供 gRPC 接口。

![docker arch](/resources/docker-arch.png)

当我们要创建一个容器的时候：

1. Docker Daemon 请求 `containerd` 来创建一个容器

2. containerd 收到请求后，创建一个叫做 `containerd-shim` 的进程，让这个进程去操作容器

> 指定容器进程是需要一个父进程来做状态收集、维持 stdin、fd 打开等工作的，假如这个父进程是 containerd，那如果 containerd 挂掉的话，整个宿主机上所有的容器都得退出了，而引入 `containerd-shim` 这个垫片就可以来规避这个问题了。

3. 创建容器需要做一些 namespaces 和 cgroups 的配置，以及挂载 root 文件系统等操作，这些操作有标准的 [OCI](https://opencontainers.org/) 规范，`runc` 就是一个参考实现
4. `runc` 启动完容器后本身会直接退出，`containerd-shim` 则会成为容器进程的父进程, 负责收集容器进程的状态, 上报给 containerd, 并在容器中 pid 为 1 的进程退出后接管容器中的子进程进行清理, 确保不会出现僵尸进程。

## [CRI](https://github.com/kubernetes/cri-api)

在 Kubernetes 早期通过硬编码的方式直接调用 Docker API，随着 Docker 的发展以及 Google 的主导，出现了更多容器运行时，Kubernetes 为了支持更多更精简的容器运行时，Google 和红帽主导推出了 CRI 标准，用于将 Kubernetes 平台和特定的容器运行时解耦。

`CRI`（Container Runtime Interface 容器运行时接口）本质上就是 Kubernetes 定义的一组与容器运行时进行交互的接口，所以只要实现了这套接口的容器运行时都可以对接到 Kubernetes 平台上来。

> Kubernetes 推出 CRI 标准时还没有现在的统治地位，所以有一些容器运行时可能不会自身就去实现 CRI 接口，于是就有了 `shim（垫片）`， 一个 shim 的职责就是作为适配器将各种容器运行时本身的接口适配到 Kubernetes 的 CRI 接口上，其中 `dockershim` 就是 Kubernetes 对接 Docker 到 CRI 接口上的一个垫片实现。

![cri shim](/resources/cri-shim.png)

Kubelet 通过 gRPC 框架与容器运行时或 shim 进行通信，其中 kubelet 作为客户端，CRI shim（也可能是容器运行时本身）作为服务器。

CRI 定义的 API 主要包括两个 gRPC 服务，`ImageService` 和 `RuntimeService`，可以通过 kubelet 中的标志 `--container-runtime-endpoint` 和 `--image-service-endpoint` 来配置这两个服务的套接字。

- `ImageService` 服务主要是拉取镜像、查看和删除镜像等操作
- `RuntimeService` 则是用来管理 Pod 和容器的生命周期，以及与容器交互的调用（exec/attach/port-forward）等操作

![kubelet cri](/resources/kubelet-cri.png)

与此同时 Kubernetes 社区也做了一个专门用于 Kubernetes 的 CRI 运行时 CRI-O，直接兼容 CRI 和 OCI 规范。

![cri-o](/resources/cri-o.png)

## [Containerd](https://containerd.io/)

其实使用 Docker 的话调用链比较长的，真正容器相关的操作有 containerd 就足够，Docker 太过于复杂笨重了，当然 Docker 深受欢迎的很大一个原因就是提供了很多对用户操作比较友好的功能，但是对于 Kubernetes 来说压根不需要这些功能，因为都是通过接口去操作容器的，所以自然也就可以将容器运行时切换到 containerd 来。

![docker vs containerd](/resources/docker-vs-containerd.png)

在 containerd 1.0 中，对 CRI 的适配是通过一个单独的 `CRI-Containerd` 进程来完成的，因为最开始 containerd 还会去适配其他的系统（比如 swarm），所以没有直接实现 CRI，所以这个对接工作就交给 `CRI-Containerd` 这个 shim 了。

然后到了 containerd 1.1 版本后就去掉了 `CRI-Containerd` 这个 shim，直接把适配逻辑作为插件的方式集成到了 containerd 主进程中。

![containerd cri](/resources/containerd-cri.png)

containerd 是一个工业级标准的容器运行时，它强调简单性、健壮性和可移植性，containerd 可以负责干下面这些事情：

- 管理容器的生命周期（从创建容器到销毁容器）
- 拉取/推送容器镜像
- 存储管理（管理镜像及容器数据的存储）
- 调用 runc 运行容器（与 runc 等容器运行时交互）
- 管理容器网络接口及网络

![containerd arch](/resources/containerd-arch.png)

containerd 采用的也是 C/S 架构，服务端通过 `unix domain socket` 暴露低层的 gRPC API 接口，客户端通过这些 API 管理节点上的容器，每个 containerd 只负责一台机器，Pull 镜像，对容器的操作（启动、停止等），网络，存储都是由 containerd 完成。**具体运行容器由 runc 负责，实际上只要是符合 OCI 规范的容器都可以支持**。

为了解耦，containerd 将系统划分成了不同的组件，每个组件都由一个或多个模块协作完成（Core 部分），每一种类型的模块都以插件的形式集成到 containerd 中，而且插件之间是相互依赖的，例如，上图中的每一个长虚线的方框都表示一种类型的插件，包括 Service Plugin、Metadata Plugin、GC Plugin、Runtime Plugin 等，其中 Service Plugin 又会依赖 Metadata Plugin、GC Plugin 和 Runtime Plugin。每一个小方框都表示一个细分的插件，例如 Metadata Plugin 依赖 Containers Plugin、Content Plugin 等。比如:

- `Content Plugin`: 提供对镜像中可寻址内容的访问，所有不可变的内容都被存储在这里。
- `Snapshot Plugin`: 用来管理容器镜像的文件系统快照，镜像中的每一层都会被解压成文件系统快照，类似于 Docker 中的 graphdriver。

总体来看 containerd 可以分为三个大块：Storage、Metadata 和 Runtime。

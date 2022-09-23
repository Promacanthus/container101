# 容器安全

容器的安全性很大程度是由容器的架构特性所决定的，比如：

- 容器与宿主机共享 Linux 内核，
- 通过 Namespace 来做资源的隔离，
- 通过 `shim`/`runC` 的方式来启动等等。

这些容器架构特性，在选择使用容器之后，作为使用容器的用户，其实已经没有多少能力去对架构这个层面做安全上的改动了。

那么对于使用容器的用户，在运行容器的时候，在安全方面主要可以从这两个角度来考虑：

1. 赋予容器合理的 capabilities，Docker 提供 privileged 参数将所有 capabilities 都赋予给容器。
2. 在容器中以非 root 用户来运行程序。

## [Linux capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html)

在 `Linux capabilities` 出现前，进程的权限可以简单分为两类：

- 特权用户的进程（进程的有效用户 ID 是 0，可以认为就是 root 用户的进程），
- 非特权用户的进程（进程的有效用户 ID 是非 0，可以理解为非 root 用户进程）。

特权用户进程可以执行 Linux 系统上的所有操作，而非特权用户在执行某些操作的时候就会被内核限制执行。其实这也是通常对 Linux 中 root 用户与非 root 用户的理解。

从 kernel 2.2 开始，Linux 把特权用户所有的这些“特权”做了更详细的划分，这样被划分出来的每个单元就被称为 capability。对于任意一个进程，在做任意一个特权操作时，都需要有这个特权操作对应的 capability。比如说：-

- 运行 iptables 命令，对应的进程需要有 `CAP_NET_ADMIN` capability。
-  mount 文件系统，对应的进程需要有 `CAP_SYS_ADMIN`  capability。

> `CAP_SYS_ADMIN`  capability 里允许了大量的特权操作，包括文件系统，交换空间，还有对各种设备的操作，以及系统调试相关的调用等等。

**在普通 Linux 节点上，非 root 用户启动的进程缺省没有任何 `Linux capabilities`，而 root 用户启动的进程缺省包含了所有的 `Linux capabilities`。**

在进程的 `/proc/<pid>/status` 中可以查看 capabilities 相关的参数。对于当前进程，直接影响某个特权操作是否可以被执行的参数，是"`CapEff`"，也就是"Effective capability sets"，这是一个 bitmap，每一个 bit 代表一项 capability 是否被打开。

一个进程是否具有某个 capabilities 不仅和进程 status 中的相关参数有关，还有应用程序文件属性中的 capabilities 有关系，在启动进程的时候，这两部分共同起作用。

如果要新启动一个程序，在 Linux 里的过程就是先通过 `fork()` 来创建出一个子进程，然后调用 `execve()` 系统调用读取文件系统里的程序文件，把程序文件加载到进程的代码段中开始运行。这个新运行的进程里的相关 capabilities 参数的值，是由它的父进程以及程序文件中的 capabilities 参数值计算得来的。

**文件中可以设置 capabilities 参数值，并且这个值会影响到最后运行它的进程。**比如，把 `iptables` 的应用程序加上 `CAP_NET_ADMIN` 的 capability，那么即使是非 root 用户也有执行 `iptables` 的权限了。

因为安全方面的考虑，容器缺省启动的时候，哪怕是容器中 root 用户的进程，系统也只允许了 15 个 capabilities，查看容器 init 进程 status 里的 Cap 参数，如下所示就是容器中缺省的 capabilities。

```shell
docker run --name iptables -it registry/iptables:v1 bash

cat /proc/1/status  |grep Cap
CapInh:          00000000a80425fb
CapPrm:          00000000a80425fb
CapEff:          00000000a80425fb
CapBnd:          00000000a80425fb
CapAmb:          0000000000000000
```

容器中的权限越高，对系统安全的威胁显然也是越大的。比如，容器中的进程有了 `CAP_SYS_ADMIN` 的特权之后，这些进程可以在容器里直接访问磁盘设备，直接读取或者修改宿主机上的所有文件。所以，在容器平台上是基本不允许把容器直接设置为"`privileged`"的，需要根据容器中进程需要的最少特权来赋予 capabilities。

## 容器中的用户

尽管容器中 root 用户的 Linux capabilities 已经减少了很多，但是在没有 User Namespace 的情况下，容器中 root 用户和宿主机上的 root 用户的 uid 是完全相同的，一旦有软件的漏洞，容器中的 root 用户就可以操控整个宿主机。**为了减少安全风险，业界都是建议在容器中以非 root 用户来运行进程。**

### run as non-root user（给容器指定一个普通用户）

让容器以非 root 用户运行：

1. 给容器指定一个普通用户 uid。比如在 docker 启动容器的时候加上"`-u`"参数，在参数中指定 `uid/gid`
2. 在创建容器镜像时，用 Dockerfile 在容器镜像里建立一个用户，使用 `USER` 命令指定用户名，这样容器里缺省的进程都会以这个用户启动

存在潜在问题：

1. 由于用户 uid 是整个节点共享的，在容器中定义的 uid，也就是宿主机上的 uid，这样就很容易引起 uid 的冲突。
2. 在一台 Linux 系统上，每个用户下的资源是有限制的，比如打开文件数目（open files）、最大进程数目（max user processes）等等。一旦有很多个容器共享一个 uid，这些容器就很可能很快消耗掉这个 uid 下的资源，这样很容易导致这些容器都不能再正常工作。

### [User Namespace](https://man7.org/linux/man-pages/man7/user_namespaces.7.html)（用户隔离技术的支持）

User Namespace 隔离了一台 Linux 节点上的 User ID（uid）和 Group ID（gid），并**给 Namespace 中的 uid/gid 的值与宿主机上的 uid/gid 值建立了一个映射关系（如 "`--uidmap 0:2000:1000`" 即 "`ns_uid:host_uid:amount`"）。**经过 User Namespace 的隔离，在 Namespace 中看到的进程的 uid/gid，就和宿主机 Namespace 中看到的 uid 和 gid 不一样了。

> 注意：User Namespace 是可以嵌套的，比如在 namespace_2 里可以再建立一个 namespace_3，这个嵌套的特性是其他 Namespace 没有的。

优势：

1. 把容器中 root 用户（uid 0）映射成宿主机上的普通用户。
   1. 作为容器中的 root，还是可以有一些 Linux capabilities，在容器中还是可以执行一些特权的操作。
   2. 而在宿主机上 uid 是普通用户，即使这个用户逃逸出容器 Namespace，它的执行权限还是有限的。
2. 对于用户在容器中自己定义普通用户 uid 的情况，只要为每个容器在节点上分配一个 uid 范围，就不会出现在宿主机上 uid 冲突的问题了。

### rootless container（以非 root 用户启动和管理容器）

[rootless container](https://developers.redhat.com/blog/2020/09/25/rootless-containers-with-podman-the-basics) 中的 "rootless" 的含义：

1. 容器中以非 root 用户来运行进程，
2. 以非 root 用户来创建容器，管理容器。

也就是说，启动容器的时候，Docker 或者 podman 是以**非 root 用户**来执行的。这样能进一步提升容器中的安全性，不用再担心因为 containerd 或者 RunC 里的代码漏洞，导致容器获得宿主机上的权限。

### 相关阅读

- [Capabilities: Why They Exist and How They Work](https://blog.container-solutions.com/linux-capabilities-why-they-exist-and-how-they-work)
- [Linux Capabilities in Practice](https://blog.container-solutions.com/linux-capabilities-in-practice)

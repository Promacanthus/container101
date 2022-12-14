# 容器内存

容器在系统中被杀掉，只可能是容器中的所有进程使用了太多的内存，超过了容器在 Memory Cgroup 里的内存限制。这时 Linux 系统就会主动杀死容器中的一个进程，往往这会导致整个容器的退出。

## OOM Killer

在 Linux 系统里内存不足时，就需要杀死一个正在运行的进程来释放一些内存，是一种内存过载后的保护机制，通过牺牲个别的进程，来保证整个节点的内存不会被全部消耗掉。

Linux 进程在调用 `malloc()` 来申请内存是的申请策略是，允许进程在申请内存的时候 overcommit ，就是说允许进程申请超过实际物理内存上限的内存。

> 比如说，节点上的空闲物理内存只有 512MB 了，但是如果一个进程调用 `malloc()` 申请了 600MB，那么这次申请还是被允许的。因为 `malloc()` 申请的是内存的虚拟地址，系统只是给了程序一个地址范围，由于没有写入数据，所以程序并没有得到真正的物理内存。物理内存只有程序真的往这个地址写入数据的时候，才会分配给程序。

overcommit 的内存申请模式：

- 好处：有效提高系统的内存利用率
- 不足：遇到内存不够时，会杀死某个正在运行的进程

在发生 OOM 时 Linux 内核里有一个 `oom_badness()` 函数，它定义了选择杀死进程的标准，涉及两个条件：

1. 进程已经使用的物理内存页面数
2. 每个进程的 OOM 校准值 `oom_score_adj`。在 /proc 文件系统中，每个进程都有一个 `/proc/<pid>/oom_score_adj` 的接口文件。可以在这个文件中输入 -1000 到 1000 之间的任意一个数值，调整进程被 OOM Kill 的几率。

计算公式：`系统总的可用页面数 X 进程的 oom_score_adj + 进程已使用物理内存页面数`，计算出来的值越大，那么这个进程被 OOM Kill 的几率也就越大。

## Memory Cgroup

Memory Cgroup 的作用是对一组进程的 Memory 使用做限制。Memory Cgroup 的虚拟文件系统的挂载点一般在 "`/sys/fs/cgroup/memory`" 这个目录下，可以在 Memory Cgroup 的挂载点目录下，创建一个子目录作为控制组。控制组之间是树状的层级结构，在这个结构中，父节点的控制组里的 `memory.limit_in_bytes` 值，可以限制它的子节点中所有进程的内存使用。

每一个控制组下面有不少参数，这里主要看与 OOM 最相关的 3 个参数：

- `memory.limit_in_bytes`：是每个控制组里最重要的参数，限制一个控制组里所有进程可使用内存的最大值。
- `memory.oom_control`：当控制组中所有进程内存使用总和达到上限值时，决定会不会触发 OOM Killer，缺省值是会触发 OOM Killer 从而杀掉某个进程，设置为 1 表示不触发，此时进程调用 `malloc()` 会暂停申请内存，进程状态因为等待资源而变成 `TASK_INTERRUPTIBLE` 。
- `memory.usage_in_bytes`：只读参数，表示当前控制组里所有进程实际使用的内存总和。
- `memory.stat`：显示当前控制组里各种内存类型的实际开销。
- `memory.swappiness`：控制整个控制组内匿名内存和 page cache 的回收，取值的范围和工作方式和全局的 swappiness 一样，但是优先级比全局的高，详见 swappiness 部分的表格。

计算公式：`控制组中总的可用页面 X 进程的 oom_score_adj + 进程已经使用的物理内存页面数`，所得值最大的进程，就会被系统选中杀死。

对于每个容器创建后，系统都会为它建立一个 Memory Cgroup 的控制组，容器的所有进程都在这个控制组里。可以通过查看内核日志判断容器是否发生 OOM，执行 `journalctl -k` 或者直接查看日志文件 `/var/log/message`,当容器发生 OOM Kill 的时候，内核会输出下面的这段信息，大致包含下面这三部分的信息：

1. 容器里每一个进程使用的内存页面数量。在"rss(Resident Set Size)"列里，指进程真正在使用的物理内存页面数量。
2. "`oom-kill:`" 列出了发生 OOM 的 Memroy Cgroup 的控制组，从控制组的信息中知道 OOM 是在哪个容器发生的。
3. "`Memory cgroup out of memory: Killed process <pid> (mem_alloc)`" 显示最终被 OOM Killer 杀死的进程

分析 OOM 的原因：

1. 进程本身需要很大的内存，`memory.limit_in_bytes` 里的内存上限值设置小了。
2. 进程的代码中有 Bug，导致内存泄漏，进程内存使用到达了 Memory Cgroup 中的上限。

## Linux 内存类型

Linux 的各个模块都需要内存，

- 比如内核需要分配内存给**页表**，**内核栈**，还有 **slab（内核各种数据结构的 Cache Pool）**；
- 用户态进程里的**堆内存**和**栈内存**，**共享库的内存**，还有文件读写的 **Page Cache**。

对于不同类型的内存，一旦总内存增高到容器里内存最高限制的数值，相应的处理方式是不同的。Memory Cgroup并不会对内核的内存做限制，主要是对用户态相关的 `RSS` 和 `Page Cache` 做限制。

### Resident Set Size

`RSS` 是指进程真正申请到物理页面的内存大小，因为当应用程序在申请内存的时候，比如说，调用 `malloc()` 来申请 100MB 的内存大小，`malloc()` 返回成功了，这时候系统其实只是把 100MB 的虚拟地址空间（`VIRT`）分配给了进程，并没有把实际的物理内存页面分配给进程。当进程对这块内存地址开始做真正读写操作的时候，系统才会把实际需要的物理内存分配给进程。而这个过程中，进程真正得到的物理内存，就是这个 `RSS(RES)` 了。

对于进程来说，`RSS` 内存包含了进程的代码段内存，栈内存，堆内存，共享库的内存, 这些内存是进程运行所必须的。具体的每一部分的 `RSS` 内存的大小，可以查看 `/proc/[pid]/smaps` 文件。

### Page Cache

每个进程除了各自独立分配到的 `RSS` 内存外，如果进程对磁盘上的文件做了读写操作，Linux 还会分配内存，把磁盘上读写到的页面存放在内存中，这部分的内存就是 `Page Cache`，主要作用是提高磁盘文件的读写性能，因为系统调用 `read()` 和 `write()` 的缺省行为都会把读过或者写过的页面存放在 `Page Cache` 里。在 Linux 系统里只要有空闲的内存，系统就会自动地把读写过的磁盘文件页面放入到 `Page Cache` 里。

Linux 的内存管理有一种内存页面回收机制（page frame reclaim），会根据系统里空闲物理内存是否低于某个阈值（wartermark），来决定是否启动内存的回收。内存回收的算法会根据不同类型的内存以及内存的最近最少用原则（LRU，Least Recently Used 算法）决定哪些内存页面先被释放。

因为 `Page Cache` 的内存页面只是起到 Cache 作用，自然是会被优先释放的。所以，`Page Cache` 是一种为了**提高磁盘文件读写性能**而利用空闲物理内存的机制。同时，内存管理中的页面回收机制，又能保证 Cache 所占用的页面可以及时释放，这样一来就不会影响程序对内存的真正需求了,但是在回收 page cache 的内存时，会影响进程申请内存的延时而影响进程的性能，同时 page cache 的回收也会影响到磁盘的读写性能。

### RSS & Page Cache in Memory Cgroup

Memory Cgroup 只统计 RSS 和 Page Cache 这两部分的内存。

- RSS 的内存，就是在当前 Memory Cgroup 控制组里所有进程的 RSS 的总和；
- Page Cache 这部分内存是控制组里的进程读写磁盘文件后，被放入到 Page Cache 里的物理内存。

Memory Cgroup 控制组里 RSS 内存和 Page Cache 内存的和，正好是 `memory.usage_in_bytes` 的值。

当控制组里的进程需要申请新的物理内存，而且 `memory.usage_in_bytes` 里的值超过控制组里的内存上限值 `memory.limit_in_bytes`，这时 Linux 的内存回收就会被调用起来。在这个控制组里的 page cache 的内存会根据新申请的内存大小释放一部分，这样还是能成功申请到新的物理内存，整个控制组里总的物理内存开销 `memory.usage_in_bytes` 还是不会超过上限值 `memory.limit_in_bytes`。

从这里可以看出，Page Cache 内存对判断容器实际内存使用率的影响，目前 Page Cache 完全就是 Linux 内核的一个自动的行为，只要读写磁盘文件，只要有空闲的内存，就会被用作 Page Cache。所以，判断容器真实的内存使用量，不能用 Memory Cgroup 里的 `memory.usage_in_bytes`，而需要用 `memory.stat` 里的 rss 值。这个很像用 free 命令查看节点的可用内存，不能看"free"字段下的值，而要看除去 Page Cache 之后的"available"字段下的值。

> 正是 Page Cache 内存的这种 Cache 的特性，对于那些有频繁磁盘访问容器，往往会看到它的内存使用率一直接近容器内存的限制值（`memory.limit_in_bytes`）。但是这时候，并不需要担心它内存的不够， 在判断一个容器的内存使用状况的时候，可以把 Page Cache 这部分内存使用量忽略，而更多的考虑容器中 RSS 的内存使用量。

在考虑内核的情况下，计算公式：`memory.usage_in_bytes = memory.stat[rss] + memory.stat[cache] + memory.kmem.usage_in_bytes`，`memory.kmem.usage_in_bytes 表示该 memcg 内核内存使用量`。

- `container_memory_usage_bytes == container_memory_rss + container_memory_cache + kernel memory`
- `container_memory_working_set_bytes = container_memory_usage_bytes – total_inactive_file（未激活的匿名缓存页）`

## Swap

Swap 空间就是一块磁盘空间，当内存写满的时候，可以把内存中不常用的数据暂时写到这个 Swap 空间上。这样一来，内存空间就可以释放出来，用来满足新的内存申请的需求。它的好处是可以**应对一些瞬时突发的内存增大需求**，不至于因为内存一时不够而触发 OOM Killer，导致进程被杀死。



### 创建指定大小的 Swap 空间

```shell
#! /bin/bash
fallocate -l 20G ./swapfile
dd if=/dev/zero of=./swapfile bs=1024 count=20971520
chmod 600 ./swapfile
mkswap ./swapfile
swapon swapfile
```

启用 Swap 空间后会导致 Memory Cgroup 对内存的限制失去作用，如果一个容器中的程序发生了内存泄漏（Memory leak），本来 Memory Cgroup 可以及时杀死这个进程，让它不影响整个节点中的其他应用程序。结果现在这个内存泄漏的进程没被杀死，还会不断地读写 Swap 磁盘，反而影响了整个节点的性能。

### swappiness

proc 文件系统下的 (swappiness)[https://www.kernel.org/doc/Documentation/sysctl/vm.txt] 这个参数 (`/proc/sys/vm/swappiness`) **决定系统将会有多频繁地使用交换分区**。一个较高的值会使得内核更频繁地使用交换分区，而一个较低的取值，则代表着内核会尽量避免使用交换分区。swappiness 的取值范围是 0–100，缺省值 60。

> 在有磁盘文件访问时，Linux 会尽量把系统的空闲内存用作 Page Cache 来提高文件的读写性能。在没有打开 Swap 空间的情况下，一旦内存不够，就只能把 Page Cache 释放了，而 RSS 内存是不能释放的。在 RSS 里的内存，大部分都是没有对应磁盘文件的内存，比如用 `malloc()` 申请得到的内存，这种内存也被称为**匿名内存（Anonymous memory）**。当 Swap 空间打开后，可以写入 Swap 空间的，就是这些匿名内存。

所以在开启 Swap 空间，并且内存紧张时，Linux 系统如何决定是先释放 Page Cache，还是先把匿名内存释放并写入到 Swap 空间里。简单分析如下：

1. 优先释放 Page Cache：那么有频繁的文件读写操作时，系统的性能就会下降
2. 优先将匿名内存释放并写入  Swap：如果释放的匿名内存马上要使用，就需要从 Swap 空间读回内存，这会让 Swap（磁盘）的读写频繁，导致系统性能下降

因此，在释放内存的时候，需要平衡 Page Cache 的释放和匿名内存的释放，而 swappiness，就是用来定义这个平衡的参数。swappiness 的取值范围是 0-100，**它不是一个百分比，更像是一个权重**，用来定义 Page Cache 内存和匿名内存的释放的一个比例。

具体的比例如下：

| swappiness的取值 | 匿名内存与 Page Cache 的比例 |            说明             |
| :--------------: | :--------------------------: | :-------------------------: |
|       100        |           100:100            |         等比例释放          |
|    60（缺省）    |            60:140            |     Page Cache 释放优先     |
|        0         |            0:200             | 并不会禁止匿名内存写入 Swap |

zone 是 Linux 划分物理内存的一个区域，其中包含 3 个水位线（water mark）用来警示空闲内存的紧张程度，保存在 `/proc/zoneinfo` 文件中。

- 在宿主机级别：当空闲内存少于内存一个 zone 的 "high water mark" 中的值的时候，Linux 还是会把匿名内存写入到 Swap 空间后释放内存，即使 swappiness 的值设置为0。
- 在 Memory Cgroup 控制组级别：当 `memory.swappiness` = 0 的时候，对匿名页的回收是始终禁止的，也就是始终都不会使用 Swap 空间。这时 Linux 系统不会再去比较 free 内存和 zone 里的 high water mark 的值，再决定一个 Memory Cgroup 中的匿名内存要不要回收了。

通过宿主机的 `swappiness` 和 `memory.swappiness` 这 2 个参数让需要使用 Swap 空间的容器和不需要 Swap 的容器，同时运行在同一个宿主机上。

# 容器存储

## 容器文件系统

在容器里运行 `df` 命令，看到在容器中根目录 (`/`) 的文件系统类型是"overlay"，而不是普通 Linux 节点上看到的 Ext4 或者 XFS 之类常见的文件系统。

**为了有效地减少磁盘上冗余的镜像数据，同时减少冗余的镜像数据在网络上的传输，需要一种针对于容器的文件系统，这类文件系统称为 UnionFS**。

UnionFS 这类文件系统实现的主要功能是把多个目录（处于不同的分区）一起挂载（mount）在一个目录下。这种多目录挂载的方式，正好可以解决容器镜像的问题。

## Overlay

UnionFS 类似的有很多种实现，包括在 Docker 里最早使用的 AUFS，在 Linux 内核 3.18 版本中，OverlayFS 代码正式合入 Linux 内核的主分支。在这之后，OverlayFS 也就逐渐成为各个主流 Linux 发行版本里缺省使用的容器文件系统了。

```bash
#!/bin/bash

# cleanup
umount ./merged
rm upper lower merged work -r

# prepare
mkdir upper lower merged work
echo "I'm from lower!" > lower/in_lower.txt
echo "I'm from upper!" > upper/in_upper.txt

# `in_both` is in both directories
echo "I'm from lower!" > lower/in_both.txt
echo "I'm from upper!" > upper/in_both.txt

sudo mount -t overlay overlay -o lowerdir=./lower,upperdir=./upper,workdir=./work ./merged
```

OverlayFS 的一个 mount 命令牵涉到四类目录，分别是 lower，upper，merged 和 work，如下图所示。

![overlayfs](/resources/overlayfs.webp)

从下往上看：

1. 首先，最下面的 "`lower/(ro)`"，是被 mount 两层目录中底下的这层（lowerdir），这一层的文件是不能被修改的，OverlayFS 是支持多个 lowerdir 的。
2. 然后，是中间的 "`uppder/(rw)`"，是被 mount 两层目录中上面的这层 （upperdir）。在 OverlayFS 中，如果有文件的创建，修改，删除操作会在这一层反映出来，它是可读写的。
3. 接着，是最上面的 "`merged/`" ，是挂载点（mount point）目录，也是用户看到的目录，用户的实际文件操作在这里进行。
4. 最后，还有一个 "`work/`"，是一个存放临时文件的目录，OverlayFS 中如果有文件修改，就会在中间过程中临时存放文件到这里。

也就是说，OverlayFS 会 mount 两层目录，分别是 lower 层（容器镜像中的文件，对于容器是只读的）和 upper 层（存放容器对文件系统里的所有改动，是可读写的），这两层目录中的文件都会映射到挂载点上。从挂载点的视角看，upper 层的文件会覆盖 lower 层的文件，比如 "`in_both.txt`" 这个文件，在 lower 层和 upper 层都有，但是挂载点 `merged/` 里看到的只是 upper 层里的 "`in_both.txt`"。

如果在挂载点  `merged/` 目录里做文件操作，具体包括新建、删除和修改这三种。

1. 新建文件，这个文件会出现在 `upper/` 目录中。
2. 删除文件，
   1. 如果删除 "`in_upper.txt`"，那么这个文件会在 `upper/` 目录中消失。
   2. 如果删除 "`in_lower.txt`", 在 `lower/` 目录里的 "`in_lower.txt`" 文件不会有变化，只是在 `upper/` 目录中增加了一个特殊文件来告诉 OverlayFS，"`in_lower.txt`'文件不能出现在 `merged/` 目录中，表示它已经被删除。
3. 修改文件，
   1. 如果删除 "`in_upper.txt`"，那么这个文件会在 `upper/` 目录中的内容直接更新。
   2. 如果修改 "`in_lower.txt`"，会在 `upper/` 目录中新建一个 "`in_lower.txt`"文件，包含更新的内容，而在 `lower/` 中的原来的实际文件 "`in_lower.txt`" 不会改变。

从宿主机的角度看，upperdir 就是一个目录，如果容器不断往容器文件系统中写入数据，实际上就是往宿主机的磁盘上写数据，这些数据存在于宿主机的磁盘目录中，就会导致宿主机上的磁盘被写满。这样影响的就不止是容器本身了，而是整个宿主机了。

对于容器来说，如果有大量的写操作是不建议写入容器文件系统的，一般是需要给容器挂载一个 volume，用来满足大量的文件读写。或者给容器的 upperdir 目录做一个容量限制，即对宿主机上文件系统中的一个目录做容量限制。

## XFS Quota

在 Linux 系统里的 XFS 文件系统缺省都有 Quota 的特性，这个特性可以为 Linux 系统里的一个用户（user），一个用户组（group）或者一个项目（project）来限制它们使用文件系统的额度（quota），也就是限制它们可以写入文件系统的文件总量。

1. 要使用 XFS Quota 特性，必须**在文件系统挂载时**加上对应的 Quota 选项，比如配置 Project Quota，那么挂载参数就是 "`pquota`"。
2. 给一个指定的目录打上一个 Project ID，这个 ID 最终是写到目录对应的 inode 上，这个步骤可以使用 XFS 文件系统自带的工具 `xfs_quota` 来完成。一旦目录打上这个 ID 之后，在这个目录下的新建的文件和目录也都会继承这个 ID。
3. 再次使用 `xfs_quota` 工具对相应的 Project ID 做 Quota 限制。

inode 是文件系统中用来描述一个文件或者一个目录的元数据，里面包含文件大小，数据块的位置，文件所属用户 / 组，文件读写属性以及其他一些属性

> 对于根目录来说，这个参数必须作为一个内核启动的参数 "`rootflags=pquota`"，这样设置就可以保证根目录在启动挂载的时候，带上 XFS Quota 的特性并且支持 Project 模式。可以从 `/proc/mounts` 信息里查看某个目录是否开启 project quota 特性。

## blkio Cgroup

- IOPS（Input/Output Operations Per Second）：每秒钟磁盘读写的次数，数值越大，表示性能越好。

- 吞吐量（Throughput）/ 带宽（Bandwidth）：每秒钟磁盘中数据的读取量，一般以 MB/s 为单位。

IOPS 和吞吐量之间的关系：`吞吐量 = 数据块大小 * IOPS`，在 IOPS 固定的情况下，如果读写的每一个数据块越大，那么吞吐量也越大。

[blkio Cgroup](https://www.kernel.org/doc/Documentation/cgroup-v1/blkio-controller.txt) 虚拟文件系统挂载点一般在 "`/sys/fs/cgroup/blkio/`"。在这个目录下创建子目录作为控制组，再把需要做 I/O 限制的进程 pid 写到控制组的 `cgroup.procs` 参数中就可以了。在 blkio Cgroup 中，有四个最主要的参数，它们可以用来限制磁盘 I/O 性能：

1. `blkio.throttle.read_iops_device` ：磁盘读取 IOPS 限制
2. `blkio.throttle.read_bps_device` ：磁盘读取吞吐量限制
3. `blkio.throttle.write_iops_device` ：磁盘写入 IOPS 限制
4. `blkio.throttle.write_bps_device` ：磁盘写入吞吐量限制

```shell
# example

# check device id
ls -l /dev/vdb -l
brw-rw---- 1 root disk 252, 16 Nov 2 08:02 /dev/vdb

# set write throughput 10M/s
echo "252:16 10485760" > $CGROUP_CONTAINER_PATH/blkio.throttle.write_bps_device

# set read throughoup 10M/s
echo "253:0 10485760" > $CGROUP_CONTAINER_PATH/blkio.throttle.read_bps_device
```

在给每个容器都加了 blkio Cgroup 限制后，

- 如果两个容器运行在 Direct I/O 模式下，同时在一个磁盘上写入文件，那么每个容器的写入磁盘的最大吞吐量，是不会互相干扰的。
- 如果两个容器运行在 Buffered I/O 模式，blkio Cgroup，根本不能限制磁盘的吞吐量。

## Direct I/O 和 Buffered I/O

Direct I/O 模式，用户进程如果要写磁盘文件，就会通过 `Linux 内核的文件系统层 (filesystem) -> 块设备层 (block layer) -> 磁盘驱动 -> 磁盘硬件`，这样一路下去写入磁盘。

Buffered I/O 模式，用户进程只是把文件数据写到内存中（Page Cache）就返回了，而 Linux 内核自己有线程会把内存中的数据再写入到磁盘中。

 **在 Linux 里，由于考虑到性能问题，绝大多数的应用都会使用 Buffered I/O 模式**。

![linux io pattern](/resources/linux-io.webp)

Direct I/O 可以通过 blkio Cgroup 来限制磁盘 I/O，但是 Buffered I/O 不能被限制。这是由于 Cgroup v1 架构设计的局限性导致的。因为每一个子系统都是独立的，资源的限制只能在子系统中发生。

如下图所示，进程 pid_y，分别属于 memory Cgroup 和 blkio Cgroup。但是在 blkio Cgroup 对进程 pid_y 做磁盘 I/O 做限制的时候，blkio 子系统是不会去关心 pid_y 用了哪些内存，哪些内存是不是属于 Page Cache，而这些 Page Cache 的页面在刷入磁盘的时候，产生的 I/O 也不会被计算到进程 pid_y 上面。

![example](/resources/pid_y_example.webp)

这就导致了 blkio 在 Cgroups v1 里不能限制 Buffered I/O 。

## Cgroup v2

Cgroup v2 虚拟文件挂载点一般在 `/sys/fs/cgroup/unified`。

Buffered I/O 限速的问题，在 Cgroup V2 得到了解决，这也是促使 Linux 开发者重新设计 Cgroup V2 的原因之一。Cgroup v2 相比 Cgroup v1 做的最大的变动就是**一个进程属于一个控制组，而每个控制组可以定义自己需要的多个子系统**。

如下图所示，进程 pid_y 属于控制组 group2，在 group2 里同时打开了 io 和 memory 子系统 （Cgroup V2 里的 io 子系统就等同于 Cgroup v1 里的 blkio 子系统）。那么，Cgroup 对进程 pid_y 的磁盘 I/O 做限制的时候，就可以考虑到进程 pid_y 写入到 Page Cache 内存的页面了，这样 buffered I/O 的磁盘限速就实现了。

![cgroup_v2_example](/resources/cgroup_v2_example.webp)

```shell
# example

# enable cgroup v2
kernel="cgroup_no_v1=blkio,memory"

# Create a new control group
mkdir -p /sys/fs/cgroup/unified/iotest

# enable the io and memory controller subsystem
echo "+io +memory" > /sys/fs/cgroup/unified/cgroup.subtree_control

# Add current bash pid in iotest control group.
# Then all child processes of the bash will be in iotest group too,
# including the fio
echo $$ >/sys/fs/cgroup/unified/iotest/cgroup.procs

# 256:16 are device major and minor ids, /mnt is on the device.
echo "252:16 wbps=10485760" > /sys/fs/cgroup/unified/iotest/io.max
cd /mnt
#Run the fio in non direct I/O mode
fio -iodepth=1 -rw=write -ioengine=libaio -bs=4k -size=1G -numjobs=1  -name=./fio.test
```


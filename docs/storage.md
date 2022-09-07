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
2. 然后，是中间的 "`uppder/(rw)`"，是被 mount 两层目录中上面的这层 （upperdir）。在 OverlayFS 中，如果有文件的创建，修改，删除操作，那么都会在这一层反映出来，它是可读写的。
3. 接着，是最上面的 "`merged/`" ，是挂载点（mount point）目录，也是用户看到的目录，用户的实际文件操作在这里进行。
4. 最后，还有一个 "`work/`"，是一个存放临时文件的目录，OverlayFS 中如果有文件修改，就会在中间过程中临时存放文件到这里。

也就是说，OverlayFS 会 mount 两层目录，分别是 lower 层和 upper 层，这两层目录中的文件都会映射到挂载点上。从挂载点的视角看，upper 层的文件会覆盖 lower 层的文件，比如 "`in_both.txt`" 这个文件，在 lower 层和 upper 层都有，但是挂载点 `merged/` 里看到的只是 upper 层里的 "`in_both.txt`"。

如果我们在 merged/ 目录里做文件操作，具体包括这三种。第一种，新建文件，这个文件会出现在 upper/ 目录中。第二种是删除文件，如果我们删除"in_upper.txt"，那么这个文件会在 upper/ 目录中消失。如果删除"in_lower.txt", 在 lower/ 目录里的"in_lower.txt"文件不会有变化，只是在 upper/ 目录中增加了一个特殊文件来告诉 OverlayFS，"in_lower.txt'这个文件不能出现在 merged/ 里了，这就表示它已经被删除了。还有一种操作是修改文件，类似如果修改"in_lower.txt"，那么就会在 upper/ 目录中新建一个"in_lower.txt"文件，包含更新的内容，而在 lower/ 中的原来的实际文件"in_lower.txt"不会改变。

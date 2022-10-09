# 常用工具

## 进程

### strace

```shell

# ps -ef | grep c-init-sig
root     15857 14391  0 06:23 pts/0    00:00:00 docker run -it registry/fwd_sig:v1 /c-init-sig
root     15909 15879  0 06:23 pts/0    00:00:00 /c-init-sig
root     15959 15909  0 06:23 pts/0    00:00:00 /c-init-sig
root     16046 14607  0 06:23 pts/3    00:00:00 grep --color=auto c-init-sig

# 15909 是容器内的 init 进程
# strace -p 15909
strace: Process 15909 attached
restart_syscall(<... resuming interrupted read ...>) = ? ERESTART_RESTARTBLOCK (Interrupted by signal)
--- SIGTERM {si_signo=SIGTERM, si_code=SI_USER, si_pid=0, si_uid=0} ---
write(1, "received SIGTERM\n", 17)      = 17
exit_group(0)                           = ?
+++ exited with 0 +++

# 15959 是容器内由 init 进程创建的其他进程
# strace -p 15959
strace: Process 15959 attached
restart_syscall(<... resuming interrupted read ...>) = ?
+++ killed by SIGKILL +++
```

### [pref](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/tools/perf)

在每个 Linux 发行版都有这个工具，安装方式：

1. CentOS：`yum install pref`
2. Ubuntu：`apt install linux-tools-common`

对于定位 CPU Usage 异常，查找高 CPU 使用率情况下的热点函数，`perf` 显然是最有力的工具。在 CPU Usage 增高的节点上找到具体的引起 CPU 增高的函数，然后就可以有针对性地聚焦到那个函数做分析。

通常分为三个步骤：

1. 抓取数据（`pref record`），命令运行结束后会在磁盘的当前目录留下 `perf.data` 文件，记录了所有采样得到的信息。
2. 数据读取（`pref script` 和 `pref report`），把 `perf.data` 转化成分析脚本，然后用 [FlameGraph](https://github.com/brendangregg/FlameGraph.git) 工具来读取脚本，生成火焰图。
3. 异常聚焦，看火焰图 X 轴占比比较大的函数。

```shell
# record
pref record -a -g -p <pid>
## -a 获取所有 CPU Core 上函数运行情况

# record target cpu
perf record -C 32 -g -- sleep 10
## -C 指定只抓取 CPU32 的执行指令
## -g 表示 call-graph enable，也就是记录函数调用关系
## sleep 10 为了让 pref 抓取 10秒钟的数据
```

第一次上手使用时，可以先运行一下 `perf list` 命令，会看到列出了大量的 event，**event 是 perf 工作的基础**，主要有两种：

1. 使用硬件的 PMU 里的 event，
2. 在内核代码中注册的 event。

```shell
 # perf list
…
  branch-instructions OR branches                    [Hardware event]
  branch-misses                                      [Hardware event]
 
  alignment-faults                                   [Software event]
  bpf-output                                         [Software event]
…
 
  block:block_bio_bounce                             [Tracepoint event]
  block:block_bio_complete                           [Tracepoint event]
```

**Hardware event** 来自处理器中的一个 PMU（Performance Monitoring Unit），这些 event 数目不多，都是底层处理器相关的行为，perf 中会命名几个通用的事件，比如 `cpu-cycles`，执行完成的 instructions，Cache 相关的 `cache-misses`。运行一下 `perf stat` ，可以看到在这段时间里这些 Hardware event 发生的数目。

```shell
# perf stat
 Performance counter stats for 'system wide':
 
          58667.77 msec cpu-clock                 #   63.203 CPUs utilized
            258666      context-switches          #    0.004 M/sec
              2554      cpu-migrations            #    0.044 K/sec
             30763      page-faults               #    0.524 K/sec
       21275365299      cycles                    #    0.363 GHz
       24827718023      instructions              #    1.17  insn per cycle
        5402114113      branches                  #   92.080 M/sec
          59862316      branch-misses             #    1.11% of all branches
 
       0.928237838 seconds time elapsed
```

**Software event** 是定义在 Linux 内核代码中的几个特定的事件，比较典型的有进程上下文切换（内核态到用户态的转换）事件 `context-switches`、发生缺页中断的事件 `page-faults` 等。

**Tracepoints event** 在 perf list 中数量众多，因为内核中很多关键函数里都有 Tracepoints。它的实现方式和 Software event 类似，都是在内核函数中注册了 event。这些 Tracepoints 不仅是用在 perf 中，它已经是 Linux 内核 tracing 的标准接口了，ftrace，ebpf 等工具都会用到它。

pref 使用这些 event 的方式有两种，计数和采样。

1. 计数，统计某个 event 在一段时间里发生了多少次（`pref stats`）
2. 采样，按照一定规则抽样（`pref record` **在不加 -e 指定 event 的时候，缺省的 event 是 Hardware event cycles**）

```shell
# perf stat -e page-faults -- sleep 1
## -e 指定要查看的 event 
 
 Performance counter stats for 'sleep 1':
 
                49      page-faults
 
       1.001583032 seconds time elapsed
 
       0.001556000 seconds user
       0.000000000 seconds sys
```

Perf 对 event 的采样有两种模式：

1. 按照 event 的数目（period），比如每发生 10000 次 cycles event 就记录一次 IP、进程等信息，`-c` 参数可以指定每发生多少次，就做一次记录。
2. 定义一个频率（frequency）， `-F` 参数就是指定频率的。

```shell
# perf record  -e cycles -c 10000 -- sleep 1
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.024 MB perf.data (191 samples) ]

# perf record -e cycles -F 99 -- sleep 1
## 指采样每秒钟做 99 次。
```

容器中使用 pref 的注意事项：

1. perf 是和 Linux kernel 一起发布的，版本最好是和 Linux kernel 一样，不然可能无法正常工作
2. 系统调用权限问题，`perf` 通过系统调用 `perf_event_open()` 来完成对 event 的计数或者采样。Docker 使用 [seccomp](https://man7.org/linux/man-pages/man2/seccomp.2.html) 默认禁止这个系统调用，可以在启动容器的命令里，加上参数"`--security-opt seccomp=unconfined`" 来解决。
3. 需要允许容器在没有 `SYS_ADMIN` 这个 capability 的情况下，也可以让 perf 访问 event。在宿主机上设置 `echo -1 > /proc/sys/kernel/perf_event_paranoid`，这样普通的容器里也能执行 perf 。

### ftrace

把 ftrace 的 tracer 设置为 `function_graph`，通过这个办法查看内核函数的调用时间。

```shell

# cd /sys/kernel/debug/tracing
# echo vfs_write >> set_ftrace_filter
# echo xfs_file_write_iter >> set_ftrace_filter
# echo xfs_file_buffered_aio_write >> set_ftrace_filter
# echo iomap_file_buffered_write
# echo iomap_file_buffered_write >> set_ftrace_filter
# echo pagecache_get_page >> set_ftrace_filter
# echo try_to_free_mem_cgroup_pages >> set_ftrace_filter
# echo try_charge >> set_ftrace_filter
# echo mem_cgroup_try_charge >> set_ftrace_filter

# echo function_graph > current_tracer
# echo 1 > tracing_on
```

## 磁盘

### fio

磁盘性能测试工具。

```shell
# fio -direct=1 -iodepth=64 -rw=read -ioengine=libaio -bs=4k -size=10G -numjobs=1  -name=./fio.test
# -direct=1 表示采用非 buffered I/O 文件读写方式，避免文件读写过程中内存缓冲对性能的影响
# -iodepth=64 同时发起 64 个 I/O 请求
# -ioengine-libaio 表示文件读写采用异步 I/O（Async I/O） 的方式，进程可以发起多个 I/O 请求，并且不用阻塞地等待 I/O 的完成，等 I/O 完成之后，进程会收到通知。
# -rw=read 表示读文件测试
# -bs=4k 表示每次读 4KB 大小数块
# -size=10G 表示总共读 10GB 的数据
# -numjobs=1 表示只有一个进程/线程在运行
```

### iostat

查看实际的磁盘写入速度。

```shell
iostat -xz 10
```

### xfs_quota

目录静态容量大小限制。

```shell
# check if project quota feature enabled
cat /proc/mounts | grep prjquota

# make directory
mkdir -p /tmp/xfs_prjquota

# set project id
xfs_quota -x -c 'project -s -p /tmp/xfs_prjquota 101' /

# set quota
xfs_quota -x -c 'limit -p bhard=10m 101' /

# write data
dd if=/dev/zero of=/tmp/xfs_prjquota/test.file bs=1024 count=20000

# check file size
ls -l /tmp/xfs_prjquota/test.file
```

## 网络

### namespace

```shell
# list network namespace
lsns -t net

# list all namespace
lsns

# enter namesapce
nsenter -t <PID> -n <command>
nsenter --target $(docker inspect -f {.State.Pid}) --net
```

### ip

```shell
docker run --init --name lat-test-1 --network none -d registry/latency-test:v1 sleep 36000

# ---Set veth---
pid=$(ps -ef | grep "sleep 36000" | grep -v grep | awk '{print $2}')
echo $pid
# Get network namespace id by pid then link pid and network namesapce id
ln -s /proc/$pid/ns/net /var/run/netns/$pid
 
# Create a pair of veth interfaces
ip link add name veth_host type veth peer name veth_container
# Put one of them in the new net ns
ip link set veth_container netns $pid
 
# In the container, setup veth_container
ip netns exec $pid ip link set veth_container name eth0
ip netns exec $pid ip addr add 172.17.1.2/16 dev eth0
ip netns exec $pid ip link set eth0 up
ip netns exec $pid ip route add default via 172.17.0.1
 
# In the host, set veth_host up
ip link set veth_host up

# set veth_host into docker0 bridge
ip link set veth_host master docker0

# ---Set ipvlan--- 
pid1=$(docker inspect lat-test-1 | grep -i Pid | head -n 1 | awk '{print $2}' | awk -F "," '{print $1}')
echo $pid1
ln -s /proc/$pid1/ns/net /var/run/netns/$pid1
 
ip link add link eth0 ipvt1 type ipvlan mode l2
ip link set dev ipvt1 netns $pid1
 
ip netns exec $pid1 ip link set ipvt1 name eth0
ip netns exec $pid1 ip addr add 172.17.3.2/16 dev eth0
ip netns exec $pid1 ip link set eth0 up
```

### tcpdump

```shell
# container eth0
ip netns exec $pid tcpdump -i eth0 host <target ip addr> -nn

# veth_host
tcpdump -i veth_host host <target ip addr> -nn

# docker0
tcpdump -i docker0 host <target ip addr> -nn

# host eth0
tcpdump -i eth0 host <target ip addr> -nn
```

### [netperf](https://hewlettpackard.github.io/netperf/)

衡量网络性能的工具，它可以提供单向吞吐量和端到端延迟的测试。

TCP_RR 是 netperf 里专门用来测试网络延时的，缺省每次运行 10 秒钟。运行以后，计算平均每秒钟 TCP request/response 的次数，这个次数越高，就说明延时越小。

```shell

# ./netperf -H 192.168.0.194 -t TCP_RR
MIGRATED TCP REQUEST/RESPONSE TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.0.194 () port 0 AF_INET : first burst 0
Local /Remote
Socket Size   Request  Resp.   Elapsed  Trans.
Send   Recv   Size     Size    Time     Rate
bytes  Bytes  bytes    bytes   secs.    per sec
 
16384  131072 1        1       10.00    2504.92
16384  131072
# ./netperf -H 192.168.0.194 -t TCP_RR
MIGRATED TCP REQUEST/RESPONSE TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.0.194 () port 0 AF_INET : first burst 0
Local /Remote
Socket Size   Request  Resp.   Elapsed  Trans.
Send   Recv   Size     Size    Time     Rate
bytes  Bytes  bytes    bytes   secs.    per sec
 
16384  131072 1        1       10.00    2410.14
16384  131072
 
# ./netperf -H 192.168.0.194 -t TCP_RR
MIGRATED TCP REQUEST/RESPONSE TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.0.194 () port 0 AF_INET : first burst 0
Local /Remote
Socket Size   Request  Resp.   Elapsed  Trans.
Send   Recv   Size     Size    Time     Rate
bytes  Bytes  bytes    bytes   secs.    per sec
 
16384  131072 1        1       10.00    2422.81
16384  131072
```

### iperf3

从 iperf3 的输出 "`Retr`" 列里，可以看到有多少重传的数据包。

```shell

# iperf3 -c 192.168.147.51
Connecting to host 192.168.147.51, port 5201
[  5] local 192.168.225.12 port 51700 connected to 192.168.147.51 port 5201
[ ID] Interval           Transfer     Bitrate                        Retr    Cwnd
[  5]   0.00-1.00   sec  1001 MBytes  8.40 Gbits/sec  162    192 KBytes
…
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  9.85 GBytes  8.46 Gbits/sec  162             sender
[  5]   0.00-10.04  sec  9.85 GBytes  8.42 Gbits/sec                  receiver
 
iperf Done.
```

### ipvsadm

查看节点上 IPVS 规则的数目。

```shell
# ipvsadm -L -n | wc -l
79004
```



## 安全

### [capsh](https://man7.org/linux/man-pages/man1/capsh.1.html)

配置 Linux capabilities 的工具。

```shell
# remove CAP_NET_ADMIN linux capabilities
sudo /usr/sbin/capsh --keep=1 --user=root   --drop=cap_net_admin  --   -c './iptables -L;sleep 100'
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
 
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
 
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
iptables: Permission denied (you must be root).

# get process id
ps -ef | grep sleep
root     22603 22275  0 19:44 pts/1    00:00:00 sudo /usr/sbin/capsh --keep=1 --user=root --drop=cap_net_admin -- -c ./iptables -L;sleep 100
root     22604 22603  0 19:44 pts/1    00:00:00 /bin/bash -c ./iptables -L;sleep 100

# check process linux capabilities
cat /proc/22604/status | grep Cap
CapInh:          0000000000000000
CapPrm:          0000003fffffefff
CapEff:          0000003fffffefff # Effective capability sets 
CapBnd:          0000003fffffefff
CapAmb:          0000000000000000
```

### getcap

查看文件的 capabilities。

```shell
getcap $(which ping)
```

### setcap

给文件设置 capabilities。

```shell
setcap -r $(which ping)
```


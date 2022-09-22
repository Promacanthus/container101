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

### pref

```shell
# record
pref record -a -g -p <pid>

# report
pref report
```

### ftrace

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


# 容器进程

使用容器的理想境界是**一个容器只启动一个进程**，但这在现实应用中有时是做不到的。

比如说，在一个容器中除了主进程之外，还会启动辅助进程，做监控或者日志；再比如说，把原来运行在虚拟机的程序移到容器里，这些程序本身就是多进程的。一旦启动了多个进程，在容器里就会出现一个 pid 1，也就是 1 号进程或者 init 进程，然后由这个进程创建出其他的子进程。

## init 进程

1. Linux 操作系统，在系统打开电源，执行 `BIOS/boot-loader` 之后，就会由 `boot-loader` 负责加载 Linux 内核，内核执行文件一般会放在 `/boot` 目录下，文件名类似 `vmlinuz`*。
2. 在内核完成操作系统的各种初始化之后，内核需要执行的第一个用户态程就是 `init` 进程。内核代码启动 1 号进程的时候，在没有外面参数指定程序路径的情况下，会从几个缺省路径（`/sbin/init`,`/etc/init`,`/bin/init`,`/bin/sh`）尝试执行 1 号进程的代码，从内核态切换到用户态。

> 目前主流的 Linux 发行版，都把 `/sbin/init` 作为符号链接指向 `Systemd`（`/sbin/init -> /lib/systemd/systemd`）。
>
> Systemd 是目前最流行的 Linux init 进程，之前还有 SysVinit、UpStart 等 Linux init 进程。它们最基本的功能是创建出 Linux 系统中其他所有的进程，并且管理这些进程。

一旦容器建立自己的 Pid Namespace（进程命名空间），这个 Namespace 里的进程号也是从 1 开始标记的。**1 号进程是第一个用户态的进程，由它直接或者间接创建 Namespace 中的其他进程。**

## Linux 信号

> 运行 kill 命令其实在 Linux 里就是发送一个信号。

```bash
kill -l
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```

信号的编号在所有类 Unix 系统上都是一样的，编号 1-31 共 31 个 POSIX 标准里的信号，编号 32-64 共33个 real-time 信号扩展。**信号（Signal）其实就是 Linux 进程收到的一个通知**。这些通知产生的源头有很多种，通知的类型也有很多种。比如下面这几个典型的场景：

1. 按下键盘“`Ctrl+C`”，当前运行的进程就会收到一个信号 `SIGINT(2)` 而退出；
2. 如果代码写得有问题，导致内存访问出错了，当前的进程就会收到另一个信号 `SIGSEGV(11)`；
3. 执行 `kill` 命令直接向进程发送信号，缺省发送 `SIGTERM(15)`，指定信号类型，如"`kill -9`"发送 `SIGKILL(9)` 信号。

进程在收到信号后，会做相应的处理，对于每一个信号，进程对它的处理都有三个选择。

1. 忽略（Ignore），对这个信号不做任何处理。对于 `SIGKILL(9)` 和 `SIGSTOP(19)` 信号进程不能忽略，它们的主要作用是**为内核和超级用户提供删除任意进程的特权**。
2. 捕获（Catch），让用户进程可以注册针对这个信号的自定义 handler（如 graceful-shutdown）。对于 `SIGKILL(9)` 和 `SIGSTOP(19)` 信号不能有用户自己的处理代码，只能执行系统的缺省（Default）行为。
3. 缺省（Default），Linux 为每个信号都定义了一个缺省的行为，[官方文档](https://man7.org/linux/man-pages/man7/signal.7.html)记录了每个信号的缺省行为。对于大部分的信号而言，应用程序不需要注册自己的 handler，使用系统缺省定义行为就可以了。

### 发送信号

1. 运行 `kill 1` 这个命令把 `SIGTERM(15)` 信号发送给 init 进程，如下图带箭头虚线。在 Linux 实现里，`kill` 命令调用了 `kill()` 系统调用（内核的调用接口）而进入到了内核函数 `sys_kill()`，如下图实线箭头。
2. 内核把信号发送给 init 进程时，调用 `sig_task_ignored()` 这个函数来判断是否要把发送的这个信号忽略掉。如果信号被忽略，那么 init 进程就不能收到指令了。

![signal task ignored](/resources/sig-task-ignored.webp)

```c
kernel/signal.c
static bool sig_task_ignored(struct task_struct *t, int sig, bool force)
{
        void __user *handler;
        handler = sig_handler(t, sig);

        /* SIGKILL and SIGSTOP may not be sent to the global init */
        if (unlikely(is_global_init(t) && sig_kernel_only(sig)))
                return true;

        if (unlikely(t->signal->flags & SIGNAL_UNKILLABLE) &&
            handler == SIG_DFL && !(force && sig_kernel_only(sig)))
                return true;

        /* Only allow kernel generated signals to this kthread */
        if (unlikely((t->flags & PF_KTHREAD) &&
                     (handler == SIG_KTHREAD_KERNEL) && !force))
                return true;

        return sig_handler_ignored(handler, sig);
}
```

init 进程能否收到信号的关键是 `sig_task_ignored()` 这个内核函数的实现，从代码注释看，关键是第二个 if 语句，其中包含 3 个判断条件。

|判断条件|含有|示例|
|---|---|---|
|`unlikely(t->signal->flags & SIGNAL_UNKILLABLE)`|进程是否为 `SIGNAL_UNKILLABLE`|每个 Namespace 的 init 进程建立时会打上 `SIGNAL_UNKILLABLE` 标签|
|`handler == SIG_DFL`|handler 是否为系统缺省值|用户没有针对该信号注册自定义handler，则使用系统缺省 handler|
|`!(force && sig_kernel_only(sig)))`|同一个 Namespace 里发出的信号 force 为 0，只有 `SIGKILL(9)` 和 `SIGSTOP(19)` 时 `sig_kernel_only()`返回 1|比如在容器内执行 kill 命令，则 force 为 0|

```c
kernel/fork.c
if (is_child_reaper(pid)) {
        ns_of_pid(pid)->child_reaper = p;
        p->signal->flags |= SIGNAL_UNKILLABLE;
}

/*
 * is_child_reaper returns true if the pid is the init process of the current namespace. 
 * As this one could be checked before pid_ns->child_reaper is assigned in copy_process, 
 * we check with the pid number.
 */
static inline bool is_child_reaper(struct pid *pid)
{
        return pid->numbers[pid->level].nr == 1;
}
```

**最关键的就是 `handler == SIG_DFL`，内核针对每个 Namespace 里的 init 进程，把只有 default handler 的信号都忽略了**，所以需要为 init 进程注册自定义的 handler 来捕获信号。

> `SIGKILL(9)` 信号是不允许注册自定义 handler 的，所以发送给 init 进程的 `SIGKILL(9)` 信号都会被内核忽略掉，因此 init 进程无法被 `SIGKILL(9)` 信号杀死，而可以被注册了自定义 handler 的 `SIGTERM(15)` 信号杀死。

### 进程处理信号

在Linux下，找到进程的 PID，然后查看`/proc/$PID/status`，包含描述哪些信号被阻止（SigBlk），被忽略（SigIgn）或被捕获（SigCgt）的行。

查看 init 进程的信号注册情况，即 SigCgt Bitmap 的值，对应位为 1 表示该信号注册了自定义 handler，数据保存在 `/proc/1/status`中。

```shell
# cat /proc/1/status
...
SigBlk: 0000000000000000
SigIgn: fffffffe57f0d8fc
SigCgt: 00000000280b2603
...
```

右边的数字是位掩码。如果将其从十六进制转换为二进制，则每个1位代表捕获的信号，从1开始从右到左计数。因此，通过解释SigCgt行，我们可以看到我的init进程正在捕获以下信号：

```shell
00000000280b2603 ==> 101000000010110010011000000011
                     | |       | ||  |  ||       |`->  1 = SIGHUP
                     | |       | ||  |  ||       `-->  2 = SIGINT
                     | |       | ||  |  |`----------> 10 = SIGUSR1
                     | |       | ||  |  `-----------> 11 = SIGSEGV
                     | |       | ||  `--------------> 14 = SIGALRM
                     | |       | |`-----------------> 17 = SIGCHLD
                     | |       | `------------------> 18 = SIGCONT
                     | |       `--------------------> 20 = SIGTSTP
                     | `----------------------------> 28 = SIGWINCH
                     `------------------------------> 30 = SIGPWR
```

- [Golang 程序](https://pkg.go.dev/os/signal#section-directories)：很多信号都注册了自定义 handler，包括 `SIGTERM(15)` 信号，即 bit 15
- Clang 程序：缺省状态下， 没有给任何信号注册自定义 handler
- shell 程序：为 `SIGINT(2)` 和 `SIGCHLD(17)` 注册了自定义 handler，即 bit 2 和 17

**在容器中，1 号进程永远不会响应 `SIGKILL(9)` 和 `SIGSTOP(19)` 这两个特权信号；对于其他的信号，如果用户自己注册了 handler，1 号进程可以响应。**

## 进程状态

> 在 Linux 内核中进程/线程都用 `task_struct{}` 这个结构来表示，是 Linux 里基本的调度单位。

![process status](/resources/process-status.webp)

从上图可以看出来，进程“活着”的时候就只有两个状态：

|状态|描述|解释|示例|
|---|---|---|---|
|运行态|TASK_RUNNING|进程处于运行中或者在 run queue 队列中| ps 命令显示进程状态为 R STAT
|睡眠态|<li>TASK_INTERRUPTIBLE（可被打断）<li>TASK_UNINTERRUPTIBLE（不可被打断）|进程等待某个资源（信号量，磁盘IO等）而进入的状态，进程处于 wait queue 队列中| <li>ps 命令显示进程状态为S <li> ps 命令显示进程状态为D STAT

进程调用 `do_exit()` 退出的时候也有两个状态：

|状态|描述|解释|示例|
|---|---|---|---|
|退出|EXIT_DEAD|进程在真正结束退出的那一瞬间的状态|
|僵尸|EXIT_ZOMBIE|进程在 EXIT_DEAD 前的一个状态，每个进程都会进入这个状态，**只有僵尸进程处于这个状态**| ps 命令显示进程状态为 Z STAT

> 为什么先进入僵尸状态而不是直接消失？是留给父进程一次机会，查看子进程的 PID、终止状态（退出码、终止原因，比如是信号终止还是正常退出等）、资源使用信息。如果子进程直接消失，那么父进程没有机会掌握子进程的具体终止情况。一般情况下，程序逻辑可能会依据子进程的终止情况做出进一步处理：比如 Nginx Master 进程获知 Worker 进程异常退出，则重新拉起来一个 Worker 进程。

进程调用 `do_exit()` 函数后 `task_struct{}` 里的 `mm/shm/sem/files` 等文件资源（`/proc/$pid` 目录中）都已经释放了，只留下了一个 `stask_struct` instance 空壳，这个进程不再响应 `SIGKILL(9)` 和 `SIGTETM(15)` 信号。**这就意味着，残留的僵尸进程，在容器里仍然占据着进程号资源，很有可能会导致新的进程不能运转**。

### 僵尸进程

父进程在创建完子进程之后就不管了，这就是造成子进程变成僵尸进程的原因。在 Linux 中的进程退出之后，如果进入僵尸状态，只能通过父进程调用如下系统调用回收僵尸进程最后的系统资源，如进程号资源：

1. `wait()`：阻塞的调用，如果没有子进程是僵尸进程，这个调用就一直不会返回，整个进程就会被阻塞住。
2. `waitpid()`：非阻塞调用，有一个参数 `WNOHANG`，表示如果在调用的时候没有僵尸进程，那么函数就马上返回。

容器中所有进程的最终父进程，就是 init 进程，由它负责生成容器中的所有其他进程。因此，容器的 init 进程有责任回收容器中的所有僵尸进程。

## 进程数目

一台 Linux 机器上的进程总数目是有限制的。如果超过这个最大值，那么系统就无法创建出新的进程了，这个最大值在 `/proc/sys/kernel/pid_max` 这个参数中看到。

- CPU <= 32: pid_max = 32 * 1024
- CPU > 32:  pid_max = N * 1024

> fork bomb: 在计算机中，通过不断建立新进程来消耗系统中的进程资源，是一种黑客攻击方式。同样，容器中的进程数就会把整个节点的可用进程总数给消耗完。

如果机器的进程资源耗尽，节点本身和节点上运行的容器都将无法工作，会收到如 `Resource temporarily unavailable` 的错误信息，因此需要限制每个容器的最大进程数，通过 pids Cgroups 子系统完成。

在一个容器建立之后，创建容器的服务会在 `/sys/fs/cgroup/pids` 下建立一个子目录（控制组）：

- `pids.max`：限制这个容器中允许的最大进程数目
- `pids.current`：显示这个容器中当前运行的进程数目

## 进程优雅退出

> 退出前的清理通常是在 `SIGTERM(15)` 信号用户注册的 handler 里进行，如清理远端链接，清理本地数据等，可以避免远端和本地发生错误，减少丢包等问题。如果收到 `SIGKILL(9)` 就没有机会执行这些清理工作。

无论是 Kubernetes 中删除 Pod 或是 Docker 停止容器，都是通过 Containerd 服务向容器的 init 进程发送一个 `SIGTERM(15)` 信号，在 init 进程退出后，容器内的其他进程也立刻退出了。需要注意的是，init 进程收到的是 `SIGTERM(15)` 信号，其他进程收到的是 `SIGKILL(9)` 信号。

当 Linux 进程收到 `SIGTERM(15)` 信号并且使进程退出，这时内核处理进程退出的入口点就是 `do_exit()` 函数，它会释放进程的相关资源，比如内存，文件句柄，信号量等等。

![init-sigterm](/resources/init-sigterm.webp)

在做完这些工作之后，它会调用一个 `exit_notify()` 函数，用来通知和这个进程相关的父子进程等。对于容器来说，还要考虑 Pid Namespace 里的其他进程。这里调用的就是 `zap_pid_ns_processes()` 函数，而在这个函数中，如果是处于退出状态的 init 进程，它会向 Namespace 中的其他进程都发送一个 `SIGKILL(9)` 信号。

为了让容器中所有的进程都收到 `SIGTREM(15)` 信号，并进行退出前的清理，可以让容器 init 进程来转发 `SIGTERM(9)` 信号，并且在收到所有子进程退出的 `SIGCHLD(17)` 信号之后， init 进程再退出。比如 Docker Container 里使用的 tini 作为 init 进程，会调用 `sigtimedwait()` 来查看自己收到的信号，然后调用 `kill()` 把信号发给子进程。

> 缺省情况下，tini 转发的信号不会发送给孙子进程，当设置 kill_process_group > 0 且子进程和孙子进程在同一个 process group 时，会将信号转发给孙子进程。

### 信号对应的系统调用

信号是 Linux 进程收到的一个通知，进程对信号的处理包括两个问题，一个进程如何发送信号，另一个进程收到信号后如何处理。

[`kill()`](https://man7.org/linux/man-pages/man2/kill.2.html) 和 [`signal()`](https://man7.org/linux/man-pages/man7/signal.7.html) 系统调用。

```shell
NAME
       kill - send signal to a process

SYNOPSIS
       #include <sys/types.h>
       #include <signal.h>

       int kill(pid_t pid, int sig);

# 参数 pid 表示把信号发送给哪个进程，如 1 就是 init 进程
# 参数 sig 表示要发送的是信号编号，如 15 就是 SIGTERM

NAME
       signal - ANSI C signal handling

SYNOPSIS
       #include <signal.h>
       typedef void (*sighandler_t)(int);
       sighandler_t signal(int signum, sighandler_t handler);

# 参数 signum 表示要发送的信号编号
# 参数 handler 是一个函数指针，用来注册用户的信号 handler，这里的handler 有三个选择，缺省，捕获，忽略
# - 缺省：一般的缺省行为有退出（terminal）、暂停（stop）或忽略（ignore），SIG_DEF 把对应信号恢复为缺省 handler
# - 捕获：注册自定义 handler 处理 SIGTERM 信号 
# - 忽略：注册 SIG_IGN 这个 handler 忽略对信号的处理

# SIGKILL(9) 和 SIGSTOP(19) 是特权信号，使用 signal 注册自定义 hander 会收到 SIG_ERR 报错，不允许捕获。
```

## CPU

Kubernetes 使用 Request CPU 和 Limit CPU 来定义 CPU 相关资源，最后会通过 CPU Cgroup 的配置实现控制容器 CPU 资源的作用。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-name
spec:
    ...
    resources:
      requests:
        memory: "64Mi"
        cpu: "1"
      limits:
        memory: "128Mi"
        cpu: "2"
    ...
```

### CPU 使用的分类

如下运行 top 命令后的输出，`%Cpu(s)`开头的这一行表示 CPU 的各种使用情况。

```shell
top - 09:00:49 up 161 days, 14:02,  0 users,  load average: 0.07, 0.08, 0.03
Tasks: 183 total,   1 running, 182 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.3 sy,  0.0 ni, 99.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   3935.1 total,    168.0 free,    776.3 used,   2990.8 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   2858.4 avail Mem 
```

如下图所示，假设只有一个 CPU 的情况下。

![cpu](/resources/cpu_usage.webp)

|类型|全拼|含义|描述|
|---|---|---|---|
|us|user|Linux 用户态 CPU 时间，不包括低优先级进程的用户态时间（nice值1-19）|普通用户程序只要没有系统调用，消耗的 CPU 都属于 us|
|sy|system|Linux 内核态 CPU 时间|执行系统调用（如读取文件`read()`），进行一些文件系统层的操作，消耗的 CPU 属于 sy|
|ni|nice|低优先级（nice值1-19）的进程用户态 CPU 时间|优先级比较低的进程|
|wa|iowait|等待 Disk I/O 的时间|`read()` 系统调用向 Linux 的 Block Layer 发出 I/O Request 触发真正的磁盘读取操作，进程被设置为 TASK_UNINTERRUPTIBLE，这段时间的消耗属于 wa|
|id|idle|系统处于空闲状态的时间|CPU 上没有需要运行的进程|
|hi|hardware irq|CPU 处理硬中断的时间|机器收到一个网络数据包时，网卡会发出一个中断，CPU 响应中断，然后进入中断服务程序|
|si|soft irq|CPU 处理软中断时间|发生中断后的工作必须完成，如果比较耗时就需要软中断，比如从网卡收数据包的大部分工作，都是通过软中断来处理的|
|st|steal|同一台宿主机上的其他虚拟机抢走的 CPU 时间|在虚拟机中使用的概念|

> **无论是硬中断（hi）或者软中断（si），它们的 CPU 时间都不会计入进程的 CPU 时间，这是因为本身它们在处理的时候就不属于任何一个进程**。

### CPU Cgroup

Cgroups 是对指定进程做计算机资源限制的，CPU Cgroup 是用来限制进程的 CPU 使用的。进程的 CPU 使用包含 2 部分，用户态的 us 和 ni 以及内核态的 sy。对于 wa、hi、si 这些 I/O 或者中断相关的 CPU 使用是不会被 CPU Cgroup 限制的。

每个 Cgroups 子系统都是通过一个虚拟文件系统挂载点的方式，挂到一个缺省的目录下，CPU Cgroup 一般在 Linux 发行版里会放在 `/sys/fs/cgroup/cpu` 这个目录下。在这个子系统的目录下，每个控制组（Control Group） 都是一个子目录，各个控制组之间的关系就是一个树状的层级关系（hierarchy）。

[参考文档](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/resource_management_guide/sec-cpu)

- 完全公平调度程序（CFS） — 一个比例分配调度程序，可根据任务优先级 ∕ 权重或 cgroup 分得的份额，在任务群组（cgroups）间按比例分配 CPU 时间（CPU 带宽）。
- 实时调度程序（RT） — 一个任务调度程序，可对实时任务使用 CPU 的时间进行限定。一般一些嵌入式实时的程序需要实时调度。

|参数|描述|默认值|备注
|---|---|---|
|cgroup.procs|当前控制组限制的进程号||
|cpu.cfs_period_us|CFS 算法的调度周期|100000us=100ms|上限1m，下限100us
|cpu.cfs_quota_us|CFS 算法中一个调度周期里这个控制组被允许的运行时间，这是绝对值| -1 |单位微秒
|cpu.shares|控制组之间的 CPU 分配比例，整个节点 CPU 跑满的时候才能发挥作用|1024表示获得1个CPU的比例|
|cpu.stat|报告 CPU 时间统计|nr_periods — 经过的周期间隔数（cpu.cfs_period_us），nr_throttled — cgroup 中任务被节流的次数（即耗尽所有按配额分得的可用时间后，被禁止运行），throttled_time — cgroup 中任务被节流的时间总计（以纳秒为单位）|
|cpu.rt_period_us|设定在某个时间段中 ，每隔多久，cgroup 对 CPU 资源的存取就要重新分配|单位为微秒us|只可用于实时调度任务
|cpu.rt_runtime_us|在某个时间段中， cgroup 中的任务对 CPU 资源的最长连续访问时间|单位为微秒us|只可用于实时调度任务

> cpu.shares：控制组A（1024），控制组B（3072），那么A与B的比值是1:3,在一台 4 个 CPU 的机器上，当 group3 和 group4 都需要 4 个 CPU 的时候，它们实际分配到的 CPU 分别是这样的：group3 是 1 个，group4 是 3 个。

### Kubernetes 中 CPU 资源限制

- cpu.shares 对应 request 的值，将 request 的值 n  乘以 1024 设置到 cpu.shares，保证即使节点 CPU 都被占满，也能获得的 CPU 数目。
- cpu.cfs_quota_us 对应 limit 的值，将 limit 的值 n 乘以 cpu.cfs_period_us 设置到 cpu.cfs_quota_us，保证 CPU 的使用上限。

### 进程和系统 CPU 使用率

top 工具主要显示了宿主机系统整体的 CPU 使用率，以及单个进程的 CPU 使用率。没有现成的工具可以得到容器 CPU 开销。

#### 进程 CPU 使用率

对于每个进程，在 proc 文件系统中都会每个进程对应的 [stat 文件（`/proc/[pid]/stat`）](https://man7.org/linux/man-pages/man5/proc.5.html)实时输出了进程的状态信息，比如进程的运行态（Running 还是 Sleeping）、父进程 PID、进程优先级、进程使用的内存等等总共 50 多项。

重点关注以下两个指标：

- utime：表示进程的用户态部分在 Linux 调度中获得 CPU 的 ticks
- stime：表示进程的内核态部分在 Linux 调度中获得 CPU 的 ticks

> ticks 是 Linux 操作系统中的一个时间单位。在 Linux 中有自己的时钟，它会周期性地产生中断。每次中断都会触发 Linux 内核去做一次进程调度，而这一次中断就是一个 tick。因为是周期性的中断，比如 1 秒钟 100 次中断，那么一个 tick 作为一个时间单位看的话，也就是 1/100 秒。

utime 和 stime 这两个指标都是累计值，表示从进程开始运行到现在，如果需要计算瞬时值的话，就需要分别求出某一时间间隔（如1秒）内的增量。

#### 系统 CPU 使用率

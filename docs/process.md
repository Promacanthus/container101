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
|运行态|TASK_RUNNING|进程处于运行中或者在 run queue 队列中| ps 命令显示进程状态为 R stat
|睡眠态|<li>TASK_INTERRUPTIBLE（可被打断）<li>TASK_UNINTERRUPTIBLE（不可被打断）|进程等待某个资源（信号量，磁盘IO等）而进入的状态，进程处于 wait queue 队列中| <li>ps 命令显示进程状态为S <li> ps 命令显示进程状态为D stat

进程调用 `do_exit()` 退出的时候也有两个状态：

|状态|描述|
|---|---|
|EXIT_DEAD|进程在真正结束退出的那一瞬间的状态|
|EXIT_ZOMBIE|进程在 EXIT_DEAD 前的一个状态，**僵尸进程处于这个状态**|

## 限制容器中进程数目

一台 Linux 机器上的进程总数目是有限制的。如果超过这个最大值，那么系统就无法创建出新的进程了，这个最大值在 `/proc/sys/kernel/pid_max` 这个参数中看到。

- CPU <= 32: pid_max = 32 * 1024
- CPU > 32:  pid_max = N * 1024

> fork bomb: 在计算机中，通过不断建立新进程来消耗系统中的进程资源，是一种黑客攻击方式。同样，容器中的进程数就会把整个节点的可用进程总数给消耗完。

如果机器的进程资源耗尽，节点本身和节点上运行的容器都将无法工作，因此需要限制每个容器的最大进程数，通过 pids Cgroups 子系统完成。

在一个容器建立之后，创建容器的服务会在 `/sys/fs/cgroup/pids` 下建立一个子目录，就是一个控制组，控制组里最关键的一个文件就是 `pids.max`。向这个文件写入数值就可以限制这个容器中允许的最大进程数目。

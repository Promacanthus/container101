# 常用工具

## strace

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

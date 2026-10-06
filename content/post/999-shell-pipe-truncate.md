+++
date = '2026-06-29T17:13:16+08:00'
draft = true
title = '999 Shell Pipe Truncate'
+++

## 问题描述

考虑一个内容非空的文本文件`config.txt`，文件内容以行为单位，且文件内容不依赖每一行的先后顺序。

由于该文件中可能存在内容重复的行，现需要对其进行去重操作。

一种简单的思路是：首先用`sort`命令对文件内容排序，然后用`uniq`命令进行去重，使用shell的管道符连接上述管线。于是构造如下命令：

```bash
cat config.txt | sort | uniq > config.txt
```

然而，bash的执行结果是`config.txt`文件被清空了。

为了进一步验证，将`config.txt`文件内容恢复之后，重新构造如下命令：

```bash
cat config.txt | tee /dev/stderr | sort | uniq > config.txt
```

结果是控制台`stderr`无内容输出，表明`cat`并没有将数据传递给`sort`。

## 幻想中的命令行为

该命令分为三部分：

1. `cat`读取`config.txt`文件，并将其内容输出到`stdout`；同时，shell将`cat`进程的`stdout`重定向到`sort`进程的`stdin`；
2. `sort`从`stdin`读取数据并排序，直至遇到`EOF`后，才一次性地将所有排序好的数据输出到`stdout`；同时，shell将`sort`进程的`stdout`重定向到`uniq`进程的`stdin`；
3. `uniq`从`stdin`读取数据并去重，并将去重后的内容输出到`stdout`；同时，shell将`uniq`进程的`stdout`重定向到`config.txt`文件。

作为生产者，当`cat`向管道中写入足够多的数据（例如装满buffer）后，`cat`进程被阻塞，`sort`作为消费者进程被唤醒，从管道中取走数据直至无数据可取，然后`sort`被阻塞，`cat`被唤醒，继续向管道中写入下一批数据；二者轮流被唤醒并重复上述动作，直至遇到`EOF`。作为前一个管道的消费者和后一个管道的生产者，`sort`和`uniq`也具有类似行为。

那么，站在bash的角度，该命令将如何执行？这里拆分为了15个步骤：

1. 创建新进程1；
2. 创建匿名管道1；
3. 将新进程1的`stdout`关联到匿名管道1；
4. 在新进程1中执行 `cat`；
5. 创建新进程2；
6. 将新进程2的`stdin`关联到匿名管道1；
7. 创建匿名管道2；
8. 将新进程2的`stdout`关联到匿名管道2；
9. 在新进程2中执行 `sort`；
10. 创建新进程3；
11. 将新进程3的`stdin`关联到匿名管道2；
12. truncate `config.txt`；
13. 将新进程3的`stdout`关联到`config.txt`；
14. 在新进程3中执行 `uniq`；
15. wait新进程1；
16. wait新进程2；
17. wait新进程3；

理论上说，`sort`会维护一个有序的中间状态，直到`cat`读到`EOF`并将其传递给`sort`后，`sort`才会最终完成排序，并开始将排序后的内容传递给`uniq`；因此，即使此时`uniq`才读入第一批数据并对`config.txt`文件进行清空（truncate）和写入操作，也不影响`sort`持有的已完成排序的内容，这些内容将源源不断地传递给`uniq`并写入`config.txt`文件，因此命令执行完毕后`config.txt`文件中应当是排序并去重后的内容；然而实际情况是数据丢失，`config.txt`文件成了空文件。

## 猜测实际的命令行为

上一小节中给出了17个步骤，理论上说这17个步骤应当按顺序执行，但实际情况是步骤12早于步骤4发生，以至于`cat`实际读取是已被清空的文件。

那么，有没有一种可能，是bash先执行步骤1、2、3、5、6、7、8、10、11、12、13、15、16、17，然后才执行步骤4、9、14呢？

## 实验验证

构造`test.sh`脚本：

```bash
!/bin/bash
set -x
cat config.txt | sort | uniq > config.txt
```

使用`strace`执行：

```bash
chmod u+x test.sh
strace -f -o strace_output.txt ./test.sh
```

输出结果如下：

```
52481 execve("./test.sh", ["./test.sh"], 0x7ffc7e51d198 /* 39 vars */) = 0
...
52481 openat(AT_FDCWD, "./test.sh", O_RDONLY) = 3
...
52481 dup2(3, 255)                      = 255
52481 close(3)                          = 0
...
52481 read(255, "#!/bin/bash\nset -x\ncat config.tx"..., 61) = 61
...
52481 pipe2([3, 4], 0)                  = 0
...
52481 clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f48ef56e050) = 52482
...
52481 close(4 <unfinished ...>
52481 <... close resumed>)              = 0
52481 close(4 <unfinished ...>
52481 <... close resumed>)              = -1 EBADF (Bad file descriptor)
52481 pipe2([4, 5], 0)                  = 0
...
52481 clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD <unfinished ...>
52481 <... clone resumed>, child_tidptr=0x7f48ef56e050) = 52483
...
52481 close(3 <unfinished ...>
52481 <... close resumed>)              = 0
52481 close(5 <unfinished ...>
52481 <... close resumed>)              = 0
52481 close(3)                          = -1 EBADF (Bad file descriptor)
52481 close(5)                          = -1 EBADF (Bad file descriptor)
...
52481 clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD <unfinished ...>
52481 <... clone resumed>, child_tidptr=0x7f48ef56e050) = 52484
...
52481 close(4 <unfinished ...>
52481 <... close resumed>)              = 0
...
52481 wait4(-1 <unfinished ...>
52481 <... wait4 resumed>, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 52482
52481 wait4(-1 <unfinished ...>
52481 <... wait4 resumed>, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 52483
52481 wait4(-1 <unfinished ...>
52481 <... wait4 resumed>, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 52484
...
52481 close(4)                          = -1 EBADF (Bad file descriptor)
...
52481 --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=52482, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
52481 wait4(-1, 0x7ffdd36794d8, WNOHANG, NULL) = -1 ECHILD (No child processes)
...
52481 exit_group(0)                     = ?
52481 +++ exited with 0 +++
```

cat

```
...
52482 getpid( <unfinished ...>
52482 <... getpid resumed>)             = 52482
52482 close(255 <unfinished ...>
52482 <... close resumed>)              = 0
...
52482 close(3 <unfinished ...>
52482 <... close resumed>)              = 0
52482 dup2(4, 1 <unfinished ...>
52482 <... dup2 resumed>)               = 1
52482 close(4 <unfinished ...>
52482 <... close resumed>)              = 0
...
52482 write(2, "+ cat config.txt\n", 17 <unfinished ...>
52482 <... write resumed>)              = 17
...
52482 access("/usr/bin/cat", X_OK <unfinished ...>
52482 <... access resumed>)             = 0
...
52482 execve("/usr/bin/cat", ["cat", "config.txt"], 0x5572ce52af90 /* 39 vars */ <unfinished ...>
52482 <... execve resumed>)             = 0
...
52482 openat(AT_FDCWD, "config.txt", O_RDONLY <unfinished ...>
52482 <... openat resumed>)             = 3
52482 fstat(3 <unfinished ...>
52482 <... fstat resumed>, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
52482 fadvise64(3, 0, 0, POSIX_FADV_SEQUENTIAL <unfinished ...>
52482 <... fadvise64 resumed>)          = 0
52482 mmap(NULL, 270336, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0 <unfinished ...>
52482 <... mmap resumed>)               = 0x7f8206321000
52482 read(3 <unfinished ...>
52482 <... read resumed>, "", 262144)   = 0
52482 close(3 <unfinished ...>
52482 <... close resumed>)              = 0
52482 close(1 <unfinished ...>
52482 <... close resumed>)              = 0
52482 close(2 <unfinished ...>
52482 <... close resumed>)              = 0
52482 exit_group(0 <unfinished ...>
52482 <... exit_group resumed>)         = ?
52482 +++ exited with 0 +++
```

sort:

```
...
52483 getpid( <unfinished ...>
52483 <... getpid resumed>)             = 52483
52483 close(255 <unfinished ...>
52483 <... close resumed>)              = 0
...
52483 close(4 <unfinished ...>
52483 <... close resumed>)              = 0
52483 dup2(3, 0 <unfinished ...>
52483 <... dup2 resumed>)               = 0
52483 close(3 <unfinished ...>
52483 <... close resumed>)              = 0
52483 dup2(5, 1 <unfinished ...>
52483 <... dup2 resumed>)               = 1
52483 close(5 <unfinished ...>
52483 <... close resumed>)              = 0
...
52483 write(2, "+ sort\n", 7 <unfinished ...>
52483 <... write resumed>)              = 7
...
52483 access("/usr/bin/sort", X_OK <unfinished ...>
52483 <... access resumed>)             = 0
...
52483 execve("/usr/bin/sort", ["sort"], 0x5572ce52af90 /* 39 vars */ <unfinished ...>
52483 <... execve resumed>)             = 0
...
52483 read(0, "", 131072)               = 0
...
52483 close(0)                          = 0
52483 close(1)                          = 0
52483 close(2 <unfinished ...>
52483 <... close resumed>)              = 0
52483 exit_group(0 <unfinished ...>
52483 <... exit_group resumed>)         = ?
52483 +++ exited with 0 +++
```

uniq:

```
...
52484 getpid( <unfinished ...>
52484 <... getpid resumed>)             = 52484
52484 close(255 <unfinished ...>
52484 <... close resumed>)              = 0
...
52484 dup2(4, 0 <unfinished ...>
52484 <... dup2 resumed>)               = 0
52484 close(4 <unfinished ...>
52484 <... close resumed>)              = 0
...
52484 write(2, "+ uniq\n", 7 <unfinished ...>
52484 <... write resumed>)              = 7
...
52484 access("/usr/bin/uniq", X_OK <unfinished ...>
52484 <... access resumed>)             = 0
...
52484 openat(AT_FDCWD, "config.txt", O_WRONLY|O_CREAT|O_TRUNC, 0666 <unfinished ...>
52484 <... openat resumed>)             = 3
52484 dup2(3, 1 <unfinished ...>
52484 <... dup2 resumed>)               = 1
52484 close(3 <unfinished ...>
52484 <... close resumed>)              = 0
52484 execve("/usr/bin/uniq", ["uniq"], 0x5572ce52af90 /* 39 vars */ <unfinished ...>
52484 <... execve resumed>)             = 0
...
52484 read(0 <unfinished ...>
52484 <... read resumed>, "", 4096)     = 0
52484 close(0 <unfinished ...>
52484 <... close resumed>)              = 0
52484 close(1 <unfinished ...>
52484 <... close resumed>)              = 0
52484 close(2)                          = 0
52484 exit_group(0)                     = ?
52484 +++ exited with 0 +++
```

## 附录A

实验环境：

- OS：Gentoo Linux x86_64
- Kernel: Linux 6.18.35
- CPU: 12th Gen Intel(R) Core(TM) i9-12900H (20) @ 5.00 GHz
- bash: GNU bash, version 5.3.9(1)-release (x86_64-pc-linux-gnu)
- /lib/libc.so.6: GNU C Library (Gentoo 2.43-r2 (patchset 3)) stable release version 2.43. Compiled by GNU CC version 15.2.1 20260214.
- strace: strace -- version 7.0
- cat: cat (GNU coreutils) 9.11 Packaged by Gentoo (9.11-r1 (p0))
- sort: sort (GNU coreutils) 9.11 Packaged by Gentoo (9.11-r1 (p0))
- uniq: uniq (GNU coreutils) 9.11 Packaged by Gentoo (9.11-r1 (p0))

完整的输出结果为：

test.sh

```
52481 execve("./test.sh", ["./test.sh"], 0x7ffc7e51d198 /* 39 vars */) = 0
52481 brk(NULL)                         = 0x5572ce51b000
52481 access("/etc/ld.so.preload", R_OK) = -1 ENOENT (No such file or directory)
52481 openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
52481 fstat(3, {st_mode=S_IFREG|0644, st_size=129098, ...}) = 0
52481 mmap(NULL, 129098, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f48ef81d000
52481 close(3)                          = 0
52481 openat(AT_FDCWD, "/usr/lib64/libreadline.so.8", O_RDONLY|O_CLOEXEC) = 3
52481 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
52481 fstat(3, {st_mode=S_IFREG|0755, st_size=379888, ...}) = 0
52481 mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f48ef81b000
52481 mmap(NULL, 384096, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f48ef7bd000
52481 mmap(0x7f48ef7d3000, 212992, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x16000) = 0x7f48ef7d3000
52481 mmap(0x7f48ef807000, 40960, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x4a000) = 0x7f48ef807000
52481 mmap(0x7f48ef811000, 36864, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x54000) = 0x7f48ef811000
52481 mmap(0x7f48ef81a000, 3168, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f48ef81a000
52481 close(3)                          = 0
52481 openat(AT_FDCWD, "/usr/lib64/libtinfo.so.6", O_RDONLY|O_CLOEXEC) = 3
52481 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
52481 fstat(3, {st_mode=S_IFREG|0755, st_size=240800, ...}) = 0
52481 mmap(NULL, 244344, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f48ef781000
52481 mmap(0x7f48ef788000, 126976, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x7000) = 0x7f48ef788000
52481 mmap(0x7f48ef7a7000, 69632, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x26000) = 0x7f48ef7a7000
52481 mmap(0x7f48ef7b8000, 20480, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x36000) = 0x7f48ef7b8000
52481 close(3)                          = 0
52481 openat(AT_FDCWD, "/usr/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
52481 read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\340v\2\0\0\0\0\0"..., 832) = 832
52481 pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 840, 64) = 840
52481 fstat(3, {st_mode=S_IFREG|0755, st_size=1892624, ...}) = 0
52481 pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 840, 64) = 840
52481 mmap(NULL, 1923952, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f48ef5ab000
52481 mmap(0x7f48ef5cf000, 1388544, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x24000) = 0x7f48ef5cf000
52481 mmap(0x7f48ef722000, 331776, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x177000) = 0x7f48ef722000
52481 mmap(0x7f48ef773000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1c7000) = 0x7f48ef773000
52481 mmap(0x7f48ef779000, 31600, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f48ef779000
52481 close(3)                          = 0
52481 openat(AT_FDCWD, "/usr/lib64/libtinfow.so.6", O_RDONLY|O_CLOEXEC) = 3
52481 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
52481 fstat(3, {st_mode=S_IFREG|0755, st_size=240824, ...}) = 0
52481 mmap(NULL, 244472, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f48ef56f000
52481 mmap(0x7f48ef576000, 126976, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x7000) = 0x7f48ef576000
52481 mmap(0x7f48ef595000, 69632, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x26000) = 0x7f48ef595000
52481 mmap(0x7f48ef5a6000, 20480, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x36000) = 0x7f48ef5a6000
52481 close(3)                          = 0
52481 mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f48ef56d000
52481 arch_prctl(ARCH_SET_FS, 0x7f48ef56dd80) = 0
52481 set_tid_address(0x7f48ef56e3a8)   = 52481
52481 set_robust_list(0x7f48ef56e060, 24) = 0
52481 rseq({cpu_id_start=0, cpu_id=RSEQ_CPU_ID_UNINITIALIZED, rseq_cs=NULL, flags=0, node_id=0, mm_cid=0, slice_ctrl={request=0, granted=0, __reserved=0}, __reserved=0}, 32, 0, 0x53053053) = 0
52481 mprotect(0x7f48ef773000, 16384, PROT_READ) = 0
52481 mprotect(0x7f48ef5a6000, 16384, PROT_READ) = 0
52481 mprotect(0x7f48ef7b8000, 16384, PROT_READ) = 0
52481 mprotect(0x7f48ef811000, 12288, PROT_READ) = 0
52481 mprotect(0x55728e706000, 12288, PROT_READ) = 0
52481 mprotect(0x7f48ef878000, 8192, PROT_READ) = 0
52481 prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
52481 getrandom("\x2e\xd1\xe6\x75\x93\x6e\x8d\xcd", 8, GRND_NONBLOCK) = 8
52481 munmap(0x7f48ef81d000, 129098)    = 0
52481 openat(AT_FDCWD, "/dev/tty", O_RDWR|O_NONBLOCK) = 3
52481 close(3)                          = 0
52481 brk(NULL)                         = 0x5572ce51b000
52481 brk(0x5572ce53c000)               = 0x5572ce53c000
52481 openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
52481 fstat(3, {st_mode=S_IFREG|0644, st_size=6245920, ...}) = 0
52481 mmap(NULL, 6245920, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f48eee00000
52481 close(3)                          = 0
52481 openat(AT_FDCWD, "/usr/lib64/gconv/gconv-modules.cache", O_RDONLY|O_CLOEXEC) = 3
52481 fstat(3, {st_mode=S_IFREG|0644, st_size=27012, ...}) = 0
52481 mmap(NULL, 27012, PROT_READ, MAP_SHARED, 3, 0) = 0x7f48ef836000
52481 close(3)                          = 0
52481 futex(0x7f48ef7786ec, FUTEX_WAKE_PRIVATE, 2147483647) = 0
52481 getresuid([1000], [1000], [1000]) = 0
52481 getresgid([1000], [1000], [1000]) = 0
52481 rt_sigprocmask(SIG_BLOCK, NULL, [], 8) = 0
52481 rt_sigaction(SIGCHLD, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52481 rt_sigaction(SIGCHLD, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f48ef5e8480}, 8) = 0
52481 rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52481 rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52481 rt_sigaction(SIGQUIT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52481 rt_sigaction(SIGQUIT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52481 rt_sigaction(SIGTSTP, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52481 rt_sigaction(SIGTSTP, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52481 rt_sigaction(SIGTTIN, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52481 rt_sigaction(SIGTTIN, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52481 rt_sigaction(SIGTTOU, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52481 rt_sigaction(SIGTTOU, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52481 rt_sigprocmask(SIG_BLOCK, NULL, [], 8) = 0
52481 rt_sigaction(SIGQUIT, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52481 uname({sysname="Linux", nodename="gentoo", ...}) = 0
52481 newfstatat(AT_FDCWD, "/home/bufbrane", {st_mode=S_IFDIR|0755, st_size=582, ...}, 0) = 0
52481 newfstatat(AT_FDCWD, ".", {st_mode=S_IFDIR|0755, st_size=582, ...}, 0) = 0
52481 newfstatat(AT_FDCWD, "/home", {st_mode=S_IFDIR|0755, st_size=16, ...}, 0) = 0
52481 newfstatat(AT_FDCWD, "/home/bufbrane", {st_mode=S_IFDIR|0755, st_size=582, ...}, 0) = 0
52481 newfstatat(AT_FDCWD, "/home/bufbrane", {st_mode=S_IFDIR|0755, st_size=582, ...}, 0) = 0
52481 getpid()                          = 52481
52481 getppid()                         = 52479
52481 getpid()                          = 52481
52481 getppid()                         = 52479
52481 getpid()                          = 52481
52481 getppid()                         = 52479
52481 getpgrp()                         = 52479
52481 ioctl(2, TIOCGPGRP, [52479])      = 0
52481 rt_sigaction(SIGCHLD, {sa_handler=0x55728e664a00, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f48ef5e8480}, 8) = 0
52481 prlimit64(0, RLIMIT_NPROC, NULL, {rlim_cur=127072, rlim_max=127072}) = 0
52481 openat(AT_FDCWD, "/usr/share/locale/locale.alias", O_RDONLY|O_CLOEXEC) = 3
52481 fstat(3, {st_mode=S_IFREG|0644, st_size=2998, ...}) = 0
52481 read(3, "# Locale name alias data base.\n#"..., 4096) = 2998
52481 read(3, "", 4096)                 = 0
52481 close(3)                          = 0
52481 openat(AT_FDCWD, "/usr/share/locale/en_US.UTF-8/LC_MESSAGES/bash.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
52481 openat(AT_FDCWD, "/usr/share/locale/en_US.utf8/LC_MESSAGES/bash.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
52481 openat(AT_FDCWD, "/usr/share/locale/en_US/LC_MESSAGES/bash.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
52481 openat(AT_FDCWD, "/usr/share/locale/en.UTF-8/LC_MESSAGES/bash.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
52481 openat(AT_FDCWD, "/usr/share/locale/en.utf8/LC_MESSAGES/bash.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
52481 openat(AT_FDCWD, "/usr/share/locale/en/LC_MESSAGES/bash.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
52481 rt_sigprocmask(SIG_BLOCK, NULL, [], 8) = 0
52481 openat(AT_FDCWD, "./test.sh", O_RDONLY) = 3
52481 newfstatat(AT_FDCWD, "./test.sh", {st_mode=S_IFREG|0744, st_size=61, ...}, 0) = 0
52481 ioctl(3, TCGETS2, 0x7ffdd367a590) = -1 ENOTTY (Inappropriate ioctl for device)
52481 lseek(3, 0, SEEK_CUR)             = 0
52481 read(3, "#!/bin/bash\nset -x\ncat config.tx"..., 80) = 61
52481 lseek(3, 0, SEEK_SET)             = 0
52481 prlimit64(0, RLIMIT_NOFILE, NULL, {rlim_cur=1024, rlim_max=512*1024}) = 0
52481 fcntl(255, F_GETFD)               = -1 EBADF (Bad file descriptor)
52481 dup2(3, 255)                      = 255
52481 close(3)                          = 0
52481 fcntl(255, F_SETFD, FD_CLOEXEC)   = 0
52481 fcntl(255, F_GETFL)               = 0x8000 (flags O_RDONLY|O_LARGEFILE)
52481 fstat(255, {st_mode=S_IFREG|0744, st_size=61, ...}) = 0
52481 lseek(255, 0, SEEK_CUR)           = 0
52481 read(255, "#!/bin/bash\nset -x\ncat config.tx"..., 61) = 61
52481 rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
52481 fcntl(0, F_GETFD)                 = 0
52481 pipe2([3, 4], 0)                  = 0
52481 rt_sigprocmask(SIG_BLOCK, NULL, [CHLD], 8) = 0
52481 rt_sigprocmask(SIG_BLOCK, [INT TERM CHLD], [CHLD], 8) = 0
52481 rt_sigprocmask(SIG_BLOCK, ~[], [INT TERM CHLD], 8) = 0
52481 clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f48ef56e050) = 52482
52481 rt_sigprocmask(SIG_SETMASK, [INT TERM CHLD], NULL, 8) = 0
52481 rt_sigprocmask(SIG_SETMASK, [CHLD], NULL, 8) = 0
52481 close(4 <unfinished ...>
52481 <... close resumed>)              = 0
52481 close(4 <unfinished ...>
52481 <... close resumed>)              = -1 EBADF (Bad file descriptor)
52481 pipe2([4, 5], 0)                  = 0
52481 rt_sigprocmask(SIG_BLOCK, NULL <unfinished ...>
52481 <... rt_sigprocmask resumed>, [CHLD], 8) = 0
52481 rt_sigprocmask(SIG_BLOCK, [INT TERM CHLD] <unfinished ...>
52481 <... rt_sigprocmask resumed>, [CHLD], 8) = 0
52481 rt_sigprocmask(SIG_BLOCK, ~[] <unfinished ...>
52481 <... rt_sigprocmask resumed>, [INT TERM CHLD], 8) = 0
52481 clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD <unfinished ...>
52481 <... clone resumed>, child_tidptr=0x7f48ef56e050) = 52483
52481 rt_sigprocmask(SIG_SETMASK, [INT TERM CHLD] <unfinished ...>
52481 <... rt_sigprocmask resumed>, NULL, 8) = 0
52481 rt_sigprocmask(SIG_SETMASK, [CHLD] <unfinished ...>
52481 <... rt_sigprocmask resumed>, NULL, 8) = 0
52481 close(3 <unfinished ...>
52481 <... close resumed>)              = 0
52481 close(5 <unfinished ...>
52481 <... close resumed>)              = 0
52481 close(3)                          = -1 EBADF (Bad file descriptor)
52481 close(5)                          = -1 EBADF (Bad file descriptor)
52481 rt_sigprocmask(SIG_BLOCK, NULL <unfinished ...>
52481 <... rt_sigprocmask resumed>, [CHLD], 8) = 0
52481 rt_sigprocmask(SIG_BLOCK, [INT TERM CHLD] <unfinished ...>
52481 <... rt_sigprocmask resumed>, [CHLD], 8) = 0
52481 rt_sigprocmask(SIG_BLOCK, ~[], [INT TERM CHLD], 8) = 0
52481 clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD <unfinished ...>
52481 <... clone resumed>, child_tidptr=0x7f48ef56e050) = 52484
52481 rt_sigprocmask(SIG_SETMASK, [INT TERM CHLD] <unfinished ...>
52481 <... rt_sigprocmask resumed>, NULL, 8) = 0
52481 rt_sigprocmask(SIG_SETMASK, [CHLD] <unfinished ...>
52481 <... rt_sigprocmask resumed>, NULL, 8) = 0
52481 close(4 <unfinished ...>
52481 <... close resumed>)              = 0
52481 rt_sigprocmask(SIG_BLOCK, [CHLD] <unfinished ...>
52481 <... rt_sigprocmask resumed>, [CHLD], 8) = 0
52481 rt_sigprocmask(SIG_SETMASK, [CHLD] <unfinished ...>
52481 <... rt_sigprocmask resumed>, NULL, 8) = 0
52481 rt_sigprocmask(SIG_BLOCK, [CHLD] <unfinished ...>
52481 <... rt_sigprocmask resumed>, [CHLD], 8) = 0
52481 rt_sigaction(SIGINT, {sa_handler=0x55728e661c80, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52481 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52481 wait4(-1 <unfinished ...>
52481 <... wait4 resumed>, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 52482
52481 wait4(-1 <unfinished ...>
52481 <... wait4 resumed>, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 52483
52481 wait4(-1 <unfinished ...>
52481 <... wait4 resumed>, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 52484
52481 rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=0x55728e661c80, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52481 ioctl(2, TIOCGWINSZ, {ws_row=83, ws_col=186, ws_xpixel=2976, ws_ypixel=3154}) = 0
52481 rt_sigprocmask(SIG_SETMASK, [CHLD], NULL, 8) = 0
52481 close(4)                          = -1 EBADF (Bad file descriptor)
52481 rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
52481 --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=52482, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
52481 wait4(-1, 0x7ffdd36794d8, WNOHANG, NULL) = -1 ECHILD (No child processes)
52481 rt_sigreturn({mask=[]})           = 0
52481 read(255, "", 61)                 = 0
52481 exit_group(0)                     = ?
52481 +++ exited with 0 +++
```

cat

```
52482 set_robust_list(0x7f48ef56e060, 24 <unfinished ...>
52482 <... set_robust_list resumed>)    = 0
52482 rt_sigprocmask(SIG_SETMASK, [INT TERM CHLD] <unfinished ...>
52482 <... rt_sigprocmask resumed>, NULL, 8) = 0
52482 getpid( <unfinished ...>
52482 <... getpid resumed>)             = 52482
52482 close(255 <unfinished ...>
52482 <... close resumed>)              = 0
52482 rt_sigprocmask(SIG_SETMASK, [] <unfinished ...>
52482 <... rt_sigprocmask resumed>, NULL, 8) = 0
52482 rt_sigaction(SIGTSTP, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52482 rt_sigaction(SIGTTIN, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52482 rt_sigaction(SIGTTOU, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52482 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52482 close(3 <unfinished ...>
52482 <... close resumed>)              = 0
52482 dup2(4, 1 <unfinished ...>
52482 <... dup2 resumed>)               = 1
52482 close(4 <unfinished ...>
52482 <... close resumed>)              = 0
52482 fstat(2 <unfinished ...>
52482 <... fstat resumed>, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x1), ...}) = 0
52482 write(2, "+ cat config.txt\n", 17 <unfinished ...>
52482 <... write resumed>)              = 17
52482 newfstatat(AT_FDCWD, "." <unfinished ...>
52482 <... newfstatat resumed>, {st_mode=S_IFDIR|0755, st_size=582, ...}, 0) = 0
52482 newfstatat(AT_FDCWD, "/usr/local/sbin/cat" <unfinished ...>
52482 <... newfstatat resumed>, 0x7ffdd3679f20, 0) = -1 ENOENT (No such file or directory)
52482 newfstatat(AT_FDCWD, "/usr/local/bin/cat" <unfinished ...>
52482 <... newfstatat resumed>, 0x7ffdd3679f20, 0) = -1 ENOENT (No such file or directory)
52482 newfstatat(AT_FDCWD, "/usr/bin/cat" <unfinished ...>
52482 <... newfstatat resumed>, {st_mode=S_IFREG|0755, st_size=51584, ...}, 0) = 0
52482 newfstatat(AT_FDCWD, "/usr/bin/cat" <unfinished ...>
52482 <... newfstatat resumed>, {st_mode=S_IFREG|0755, st_size=51584, ...}, 0) = 0
52482 geteuid( <unfinished ...>
52482 <... geteuid resumed>)            = 1000
52482 getegid( <unfinished ...>
52482 <... getegid resumed>)            = 1000
52482 getuid( <unfinished ...>
52482 <... getuid resumed>)             = 1000
52482 getgid( <unfinished ...>
52482 <... getgid resumed>)             = 1000
52482 access("/usr/bin/cat", X_OK <unfinished ...>
52482 <... access resumed>)             = 0
52482 newfstatat(AT_FDCWD, "/usr/bin/cat" <unfinished ...>
52482 <... newfstatat resumed>, {st_mode=S_IFREG|0755, st_size=51584, ...}, 0) = 0
52482 geteuid( <unfinished ...>
52482 <... geteuid resumed>)            = 1000
52482 getegid( <unfinished ...>
52482 <... getegid resumed>)            = 1000
52482 getuid( <unfinished ...>
52482 <... getuid resumed>)             = 1000
52482 getgid( <unfinished ...>
52482 <... getgid resumed>)             = 1000
52482 access("/usr/bin/cat", R_OK <unfinished ...>
52482 <... access resumed>)             = 0
52482 rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52482 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52482 rt_sigaction(SIGQUIT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52482 <... rt_sigaction resumed>, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52482 rt_sigaction(SIGCHLD, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52482 <... rt_sigaction resumed>, {sa_handler=0x55728e664a00, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f48ef5e8480}, 8) = 0
52482 execve("/usr/bin/cat", ["cat", "config.txt"], 0x5572ce52af90 /* 39 vars */ <unfinished ...>
52482 <... execve resumed>)             = 0
52482 brk(NULL <unfinished ...>
52482 <... brk resumed>)                = 0x5631e2360000
52482 access("/etc/ld.so.preload", R_OK <unfinished ...>
52482 <... access resumed>)             = -1 ENOENT (No such file or directory)
52482 openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
52482 fstat(3, {st_mode=S_IFREG|0644, st_size=129098, ...}) = 0
52482 mmap(NULL, 129098, PROT_READ, MAP_PRIVATE, 3, 0 <unfinished ...>
52482 <... mmap resumed>)               = 0x7f820653e000
52482 close(3)                          = 0
52482 openat(AT_FDCWD, "/usr/lib64/libc.so.6", O_RDONLY|O_CLOEXEC <unfinished ...>
52482 <... openat resumed>)             = 3
52482 read(3 <unfinished ...>
52482 <... read resumed>, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\340v\2\0\0\0\0\0"..., 832) = 832
52482 pread64(3 <unfinished ...>
52482 <... pread64 resumed>, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 840, 64) = 840
52482 fstat(3 <unfinished ...>
52482 <... fstat resumed>, {st_mode=S_IFREG|0755, st_size=1892624, ...}) = 0
52482 mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0 <unfinished ...>
52482 <... mmap resumed>)               = 0x7f820653c000
52482 pread64(3 <unfinished ...>
52482 <... pread64 resumed>, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 840, 64) = 840
52482 mmap(NULL, 1923952, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0 <unfinished ...>
52482 <... mmap resumed>)               = 0x7f8206366000
52482 mmap(0x7f820638a000, 1388544, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x24000 <unfinished ...>
52482 <... mmap resumed>)               = 0x7f820638a000
52482 mmap(0x7f82064dd000, 331776, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x177000 <unfinished ...>
52482 <... mmap resumed>)               = 0x7f82064dd000
52482 mmap(0x7f820652e000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1c7000 <unfinished ...>
52482 <... mmap resumed>)               = 0x7f820652e000
52482 mmap(0x7f8206534000, 31600, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0 <unfinished ...>
52482 <... mmap resumed>)               = 0x7f8206534000
52482 close(3 <unfinished ...>
52482 <... close resumed>)              = 0
52482 mmap(NULL, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0 <unfinished ...>
52482 <... mmap resumed>)               = 0x7f8206363000
52482 arch_prctl(ARCH_SET_FS, 0x7f8206363740 <unfinished ...>
52482 <... arch_prctl resumed>)         = 0
52482 set_tid_address(0x7f8206363d68 <unfinished ...>
52482 <... set_tid_address resumed>)    = 52482
52482 set_robust_list(0x7f8206363a20, 24 <unfinished ...>
52482 <... set_robust_list resumed>)    = 0
52482 rseq({cpu_id_start=0, cpu_id=RSEQ_CPU_ID_UNINITIALIZED, rseq_cs=NULL, flags=0, node_id=0, mm_cid=0, slice_ctrl={request=0, granted=0, __reserved=0}, __reserved=0}, 32, 0, 0x53053053 <unfinished ...>
52482 <... rseq resumed>)               = 0
52482 mprotect(0x7f820652e000, 16384, PROT_READ <unfinished ...>
52482 <... mprotect resumed>)           = 0
52482 mprotect(0x5631a914d000, 4096, PROT_READ <unfinished ...>
52482 <... mprotect resumed>)           = 0
52482 mprotect(0x7f8206599000, 8192, PROT_READ <unfinished ...>
52482 <... mprotect resumed>)           = 0
52482 prlimit64(0, RLIMIT_STACK, NULL <unfinished ...>
52482 <... prlimit64 resumed>, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
52482 getrandom( <unfinished ...>
52482 <... getrandom resumed>"\x14\x8a\x7a\x31\x01\x8c\x65\x4b", 8, GRND_NONBLOCK) = 8
52482 munmap(0x7f820653e000, 129098 <unfinished ...>
52482 <... munmap resumed>)             = 0
52482 brk(NULL <unfinished ...>
52482 <... brk resumed>)                = 0x5631e2360000
52482 brk(0x5631e2381000 <unfinished ...>
52482 <... brk resumed>)                = 0x5631e2381000
52482 openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC <unfinished ...>
52482 <... openat resumed>)             = 3
52482 fstat(3 <unfinished ...>
52482 <... fstat resumed>, {st_mode=S_IFREG|0644, st_size=6245920, ...}) = 0
52482 mmap(NULL, 6245920, PROT_READ, MAP_PRIVATE, 3, 0 <unfinished ...>
52482 <... mmap resumed>)               = 0x7f8205c00000
52482 close(3 <unfinished ...>
52482 <... close resumed>)              = 0
52482 fstat(1 <unfinished ...>
52482 <... fstat resumed>, {st_mode=S_IFIFO|0600, st_size=0, ...}) = 0
52482 openat(AT_FDCWD, "config.txt", O_RDONLY <unfinished ...>
52482 <... openat resumed>)             = 3
52482 fstat(3 <unfinished ...>
52482 <... fstat resumed>, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
52482 fadvise64(3, 0, 0, POSIX_FADV_SEQUENTIAL <unfinished ...>
52482 <... fadvise64 resumed>)          = 0
52482 mmap(NULL, 270336, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0 <unfinished ...>
52482 <... mmap resumed>)               = 0x7f8206321000
52482 read(3 <unfinished ...>
52482 <... read resumed>, "", 262144)   = 0
52482 close(3 <unfinished ...>
52482 <... close resumed>)              = 0
52482 close(1 <unfinished ...>
52482 <... close resumed>)              = 0
52482 close(2 <unfinished ...>
52482 <... close resumed>)              = 0
52482 exit_group(0 <unfinished ...>
52482 <... exit_group resumed>)         = ?
52482 +++ exited with 0 +++
```

sort

```
52483 set_robust_list(0x7f48ef56e060, 24 <unfinished ...>
52483 <... set_robust_list resumed>)    = 0
52483 rt_sigprocmask(SIG_SETMASK, [INT TERM CHLD] <unfinished ...>
52483 <... rt_sigprocmask resumed>, NULL, 8) = 0
52483 getpid( <unfinished ...>
52483 <... getpid resumed>)             = 52483
52483 close(255 <unfinished ...>
52483 <... close resumed>)              = 0
52483 rt_sigprocmask(SIG_SETMASK, [] <unfinished ...>
52483 <... rt_sigprocmask resumed>, NULL, 8) = 0
52483 rt_sigaction(SIGTSTP, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52483 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52483 rt_sigaction(SIGTTIN, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52483 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52483 rt_sigaction(SIGTTOU, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52483 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52483 close(4 <unfinished ...>
52483 <... close resumed>)              = 0
52483 dup2(3, 0 <unfinished ...>
52483 <... dup2 resumed>)               = 0
52483 close(3 <unfinished ...>
52483 <... close resumed>)              = 0
52483 dup2(5, 1 <unfinished ...>
52483 <... dup2 resumed>)               = 1
52483 close(5 <unfinished ...>
52483 <... close resumed>)              = 0
52483 fstat(2 <unfinished ...>
52483 <... fstat resumed>, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x1), ...}) = 0
52483 write(2, "+ sort\n", 7 <unfinished ...>
52483 <... write resumed>)              = 7
52483 newfstatat(AT_FDCWD, "." <unfinished ...>
52483 <... newfstatat resumed>, {st_mode=S_IFDIR|0755, st_size=582, ...}, 0) = 0
52483 newfstatat(AT_FDCWD, "/usr/local/sbin/sort" <unfinished ...>
52483 <... newfstatat resumed>, 0x7ffdd3679f20, 0) = -1 ENOENT (No such file or directory)
52483 newfstatat(AT_FDCWD, "/usr/local/bin/sort" <unfinished ...>
52483 <... newfstatat resumed>, 0x7ffdd3679f20, 0) = -1 ENOENT (No such file or directory)
52483 newfstatat(AT_FDCWD, "/usr/bin/sort" <unfinished ...>
52483 <... newfstatat resumed>, {st_mode=S_IFREG|0755, st_size=129608, ...}, 0) = 0
52483 newfstatat(AT_FDCWD, "/usr/bin/sort" <unfinished ...>
52483 <... newfstatat resumed>, {st_mode=S_IFREG|0755, st_size=129608, ...}, 0) = 0
52483 geteuid( <unfinished ...>
52483 <... geteuid resumed>)            = 1000
52483 getegid( <unfinished ...>
52483 <... getegid resumed>)            = 1000
52483 getuid( <unfinished ...>
52483 <... getuid resumed>)             = 1000
52483 getgid( <unfinished ...>
52483 <... getgid resumed>)             = 1000
52483 access("/usr/bin/sort", X_OK <unfinished ...>
52483 <... access resumed>)             = 0
52483 newfstatat(AT_FDCWD, "/usr/bin/sort" <unfinished ...>
52483 <... newfstatat resumed>, {st_mode=S_IFREG|0755, st_size=129608, ...}, 0) = 0
52483 geteuid( <unfinished ...>
52483 <... geteuid resumed>)            = 1000
52483 getegid( <unfinished ...>
52483 <... getegid resumed>)            = 1000
52483 getuid( <unfinished ...>
52483 <... getuid resumed>)             = 1000
52483 getgid( <unfinished ...>
52483 <... getgid resumed>)             = 1000
52483 access("/usr/bin/sort", R_OK <unfinished ...>
52483 <... access resumed>)             = 0
52483 rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52483 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52483 rt_sigaction(SIGQUIT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52483 rt_sigaction(SIGCHLD, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52483 <... rt_sigaction resumed>, {sa_handler=0x55728e664a00, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f48ef5e8480}, 8) = 0
52483 execve("/usr/bin/sort", ["sort"], 0x5572ce52af90 /* 39 vars */ <unfinished ...>
52483 <... execve resumed>)             = 0
52483 brk(NULL <unfinished ...>
52483 <... brk resumed>)                = 0x5635314b3000
52483 access("/etc/ld.so.preload", R_OK <unfinished ...>
52483 <... access resumed>)             = -1 ENOENT (No such file or directory)
52483 openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC <unfinished ...>
52483 <... openat resumed>)             = 3
52483 fstat(3 <unfinished ...>
52483 <... fstat resumed>, {st_mode=S_IFREG|0644, st_size=129098, ...}) = 0
52483 mmap(NULL, 129098, PROT_READ, MAP_PRIVATE, 3, 0 <unfinished ...>
52483 <... mmap resumed>)               = 0x7f6f1976e000
52483 close(3 <unfinished ...>
52483 <... close resumed>)              = 0
52483 openat(AT_FDCWD, "/usr/lib64/libc.so.6", O_RDONLY|O_CLOEXEC <unfinished ...>
52483 <... openat resumed>)             = 3
52483 read(3 <unfinished ...>
52483 <... read resumed>, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\340v\2\0\0\0\0\0"..., 832) = 832
52483 pread64(3 <unfinished ...>
52483 <... pread64 resumed>, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 840, 64) = 840
52483 fstat(3 <unfinished ...>
52483 <... fstat resumed>, {st_mode=S_IFREG|0755, st_size=1892624, ...}) = 0
52483 mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0 <unfinished ...>
52483 <... mmap resumed>)               = 0x7f6f1976c000
52483 pread64(3 <unfinished ...>
52483 <... pread64 resumed>, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 840, 64) = 840
52483 mmap(NULL, 1923952, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0 <unfinished ...>
52483 <... mmap resumed>)               = 0x7f6f19596000
52483 mmap(0x7f6f195ba000, 1388544, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x24000 <unfinished ...>
52483 <... mmap resumed>)               = 0x7f6f195ba000
52483 mmap(0x7f6f1970d000, 331776, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x177000 <unfinished ...>
52483 <... mmap resumed>)               = 0x7f6f1970d000
52483 mmap(0x7f6f1975e000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1c7000 <unfinished ...>
52483 <... mmap resumed>)               = 0x7f6f1975e000
52483 mmap(0x7f6f19764000, 31600, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0 <unfinished ...>
52483 <... mmap resumed>)               = 0x7f6f19764000
52483 close(3 <unfinished ...>
52483 <... close resumed>)              = 0
52483 mmap(NULL, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0 <unfinished ...>
52483 <... mmap resumed>)               = 0x7f6f19593000
52483 arch_prctl(ARCH_SET_FS, 0x7f6f19593740 <unfinished ...>
52483 <... arch_prctl resumed>)         = 0
52483 set_tid_address(0x7f6f19593d68 <unfinished ...>
52483 <... set_tid_address resumed>)    = 52483
52483 set_robust_list(0x7f6f19593a20, 24 <unfinished ...>
52483 <... set_robust_list resumed>)    = 0
52483 rseq({cpu_id_start=0, cpu_id=RSEQ_CPU_ID_UNINITIALIZED, rseq_cs=NULL, flags=0, node_id=0, mm_cid=0, slice_ctrl={request=0, granted=0, __reserved=0}, __reserved=0}, 32, 0, 0x53053053 <unfinished ...>
52483 <... rseq resumed>)               = 0
52483 mprotect(0x7f6f1975e000, 16384, PROT_READ <unfinished ...>
52483 <... mprotect resumed>)           = 0
52483 mprotect(0x563510d1d000, 4096, PROT_READ <unfinished ...>
52483 <... mprotect resumed>)           = 0
52483 mprotect(0x7f6f197c9000, 8192, PROT_READ <unfinished ...>
52483 <... mprotect resumed>)           = 0
52483 prlimit64(0, RLIMIT_STACK, NULL <unfinished ...>
52483 <... prlimit64 resumed>, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
52483 getrandom( <unfinished ...>
52483 <... getrandom resumed>"\x78\xe9\x98\xd1\xbd\x2f\xad\xde", 8, GRND_NONBLOCK) = 8
52483 munmap(0x7f6f1976e000, 129098 <unfinished ...>
52483 <... munmap resumed>)             = 0
52483 brk(NULL <unfinished ...>
52483 <... brk resumed>)                = 0x5635314b3000
52483 brk(0x5635314d4000 <unfinished ...>
52483 <... brk resumed>)                = 0x5635314d4000
52483 openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC <unfinished ...>
52483 <... openat resumed>)             = 3
52483 fstat(3 <unfinished ...>
52483 <... fstat resumed>, {st_mode=S_IFREG|0644, st_size=6245920, ...}) = 0
52483 mmap(NULL, 6245920, PROT_READ, MAP_PRIVATE, 3, 0 <unfinished ...>
52483 <... mmap resumed>)               = 0x7f6f18e00000
52483 close(3 <unfinished ...>
52483 <... close resumed>)              = 0
52483 rt_sigaction(SIGALRM, NULL <unfinished ...>
52483 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGINT, NULL <unfinished ...>
52483 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGQUIT, NULL <unfinished ...>
52483 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGHUP, NULL <unfinished ...>
52483 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGTERM, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGUSR1, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGUSR2, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGILL, NULL <unfinished ...>
52483 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGTRAP, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigprocmask(SIG_BLOCK, ~[], [], 8) = 0
52483 rt_sigaction(SIGABRT, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
52483 rt_sigaction(SIGBUS, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGFPE, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGSEGV, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGXCPU, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGXFSZ, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGSYS, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGVTALRM, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGPROF, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGIO, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGPWR, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGSTKFLT, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGALRM, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGINT, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGQUIT, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGHUP, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGTERM, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGUSR1, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGUSR2, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGILL, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGTRAP, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigprocmask(SIG_BLOCK, ~[], [], 8) = 0
52483 rt_sigaction(SIGABRT, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
52483 rt_sigaction(SIGBUS, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGFPE, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGSEGV, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGXCPU, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGXFSZ, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGSYS, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGVTALRM, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGPROF, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGIO, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGPWR, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGSTKFLT, {sa_handler=0x563510d068b0, sa_mask=[HUP INT QUIT ILL TRAP ABRT BUS FPE USR1 SEGV USR2 ALRM TERM STKFLT XCPU XFSZ VTALRM PROF IO PWR SYS], sa_flags=SA_RESTORER, sa_restorer=0x7f6f195d3480}, NULL, 8) = 0
52483 rt_sigaction(SIGCHLD, {sa_handler=SIG_DFL, sa_mask=[CHLD], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f6f195d3480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f6f195d3480}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
52483 sched_getscheduler(0)             = 0 (SCHED_OTHER)
52483 openat(AT_FDCWD, "/proc/self/cgroup", O_RDONLY) = 3
52483 fstat(3, {st_mode=S_IFREG|0444, st_size=0, ...}) = 0
52483 read(3, "0::/user.slice/user-1000.slice/s"..., 1024) = 48
52483 close(3)                          = 0
52483 access("/sys/fs/cgroup/cgroup.controllers", F_OK) = 0
52483 openat(AT_FDCWD, "/sys/fs/cgroup/user.slice/user-1000.slice/session-69.scope/cpu.max", O_RDONLY) = 3
52483 fstat(3, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
52483 read(3, "max 100000\n", 4096)     = 11
52483 close(3)                          = 0
52483 openat(AT_FDCWD, "/sys/fs/cgroup/user.slice/user-1000.slice/cpu.max", O_RDONLY) = 3
52483 fstat(3, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
52483 read(3, "max 100000\n", 4096)     = 11
52483 close(3)                          = 0
52483 openat(AT_FDCWD, "/sys/fs/cgroup/user.slice/cpu.max", O_RDONLY) = 3
52483 fstat(3, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
52483 read(3, "max 100000\n", 4096)     = 11
52483 close(3)                          = 0
52483 openat(AT_FDCWD, "/sys/fs/cgroup//cpu.max", O_RDONLY) = -1 ENOENT (No such file or directory)
52483 sched_getaffinity(0, 128, [0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19]) = 8
52483 fadvise64(0, 0, 0, POSIX_FADV_SEQUENTIAL) = -1 ESPIPE (Illegal seek)
52483 fstat(0, {st_mode=S_IFIFO|0600, st_size=0, ...}) = 0
52483 prlimit64(0, RLIMIT_DATA, NULL, {rlim_cur=RLIM64_INFINITY, rlim_max=RLIM64_INFINITY}) = 0
52483 prlimit64(0, RLIMIT_AS, NULL, {rlim_cur=RLIM64_INFINITY, rlim_max=RLIM64_INFINITY}) = 0
52483 prlimit64(0, RLIMIT_RSS, NULL, {rlim_cur=RLIM64_INFINITY, rlim_max=RLIM64_INFINITY}) = 0
52483 sysinfo({uptime=93811, loads=[6240, 3008, 384], totalram=33348038656, freeram=25864019968, sharedram=162242560, bufferram=2834432, totalswap=8589930496, freeswap=8589930496, procs=402, totalhigh=0, freehigh=0, mem_unit=1}) = 0
52483 sysinfo({uptime=93811, loads=[6240, 3008, 384], totalram=33348038656, freeram=25864019968, sharedram=162242560, bufferram=2834432, totalswap=8589930496, freeswap=8589930496, procs=402, totalhigh=0, freehigh=0, mem_unit=1}) = 0
52483 mmap(NULL, 16912384, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6f17ddf000
52483 fstat(0, {st_mode=S_IFIFO|0600, st_size=0, ...}) = 0
52483 read(0, "", 131072)               = 0
52483 munmap(0x7f6f17ddf000, 16912384)  = 0
52483 close(0)                          = 0
52483 close(1)                          = 0
52483 close(2 <unfinished ...>
52483 <... close resumed>)              = 0
52483 exit_group(0 <unfinished ...>
52483 <... exit_group resumed>)         = ?
52483 +++ exited with 0 +++
```

uniq

```
52484 set_robust_list(0x7f48ef56e060, 24 <unfinished ...>
52484 <... set_robust_list resumed>)    = 0
52484 rt_sigprocmask(SIG_SETMASK, [INT TERM CHLD] <unfinished ...>
52484 <... rt_sigprocmask resumed>, NULL, 8) = 0
52484 getpid( <unfinished ...>
52484 <... getpid resumed>)             = 52484
52484 close(255 <unfinished ...>
52484 <... close resumed>)              = 0
52484 rt_sigprocmask(SIG_SETMASK, [] <unfinished ...>
52484 <... rt_sigprocmask resumed>, NULL, 8) = 0
52484 rt_sigaction(SIGTSTP, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52484 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52484 rt_sigaction(SIGTTIN, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52484 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52484 rt_sigaction(SIGTTOU, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52484 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52484 dup2(4, 0 <unfinished ...>
52484 <... dup2 resumed>)               = 0
52484 close(4 <unfinished ...>
52484 <... close resumed>)              = 0
52484 fstat(2 <unfinished ...>
52484 <... fstat resumed>, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x1), ...}) = 0
52484 write(2, "+ uniq\n", 7 <unfinished ...>
52484 <... write resumed>)              = 7
52484 newfstatat(AT_FDCWD, "." <unfinished ...>
52484 <... newfstatat resumed>, {st_mode=S_IFDIR|0755, st_size=582, ...}, 0) = 0
52484 newfstatat(AT_FDCWD, "/usr/local/sbin/uniq" <unfinished ...>
52484 <... newfstatat resumed>, 0x7ffdd3679f20, 0) = -1 ENOENT (No such file or directory)
52484 newfstatat(AT_FDCWD, "/usr/local/bin/uniq" <unfinished ...>
52484 <... newfstatat resumed>, 0x7ffdd3679f20, 0) = -1 ENOENT (No such file or directory)
52484 newfstatat(AT_FDCWD, "/usr/bin/uniq" <unfinished ...>
52484 <... newfstatat resumed>, {st_mode=S_IFREG|0755, st_size=59704, ...}, 0) = 0
52484 newfstatat(AT_FDCWD, "/usr/bin/uniq" <unfinished ...>
52484 <... newfstatat resumed>, {st_mode=S_IFREG|0755, st_size=59704, ...}, 0) = 0
52484 geteuid( <unfinished ...>
52484 <... geteuid resumed>)            = 1000
52484 getegid( <unfinished ...>
52484 <... getegid resumed>)            = 1000
52484 getuid( <unfinished ...>
52484 <... getuid resumed>)             = 1000
52484 getgid( <unfinished ...>
52484 <... getgid resumed>)             = 1000
52484 access("/usr/bin/uniq", X_OK <unfinished ...>
52484 <... access resumed>)             = 0
52484 newfstatat(AT_FDCWD, "/usr/bin/uniq" <unfinished ...>
52484 <... newfstatat resumed>, {st_mode=S_IFREG|0755, st_size=59704, ...}, 0) = 0
52484 geteuid( <unfinished ...>
52484 <... geteuid resumed>)            = 1000
52484 getegid( <unfinished ...>
52484 <... getegid resumed>)            = 1000
52484 getuid( <unfinished ...>
52484 <... getuid resumed>)             = 1000
52484 getgid( <unfinished ...>
52484 <... getgid resumed>)             = 1000
52484 access("/usr/bin/uniq", R_OK <unfinished ...>
52484 <... access resumed>)             = 0
52484 rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52484 <... rt_sigaction resumed>, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52484 rt_sigaction(SIGQUIT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52484 <... rt_sigaction resumed>, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7f48ef5e8480}, 8) = 0
52484 rt_sigaction(SIGCHLD, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f48ef5e8480} <unfinished ...>
52484 <... rt_sigaction resumed>, {sa_handler=0x55728e664a00, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f48ef5e8480}, 8) = 0
52484 openat(AT_FDCWD, "config.txt", O_WRONLY|O_CREAT|O_TRUNC, 0666 <unfinished ...>
52484 <... openat resumed>)             = 3
52484 dup2(3, 1 <unfinished ...>
52484 <... dup2 resumed>)               = 1
52484 close(3 <unfinished ...>
52484 <... close resumed>)              = 0
52484 execve("/usr/bin/uniq", ["uniq"], 0x5572ce52af90 /* 39 vars */ <unfinished ...>
52484 <... execve resumed>)             = 0
52484 brk(NULL <unfinished ...>
52484 <... brk resumed>)                = 0x5635fcbe0000
52484 access("/etc/ld.so.preload", R_OK <unfinished ...>
52484 <... access resumed>)             = -1 ENOENT (No such file or directory)
52484 openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC <unfinished ...>
52484 <... openat resumed>)             = 3
52484 fstat(3 <unfinished ...>
52484 <... fstat resumed>, {st_mode=S_IFREG|0644, st_size=129098, ...}) = 0
52484 mmap(NULL, 129098, PROT_READ, MAP_PRIVATE, 3, 0 <unfinished ...>
52484 <... mmap resumed>)               = 0x7fae26871000
52484 close(3 <unfinished ...>
52484 <... close resumed>)              = 0
52484 openat(AT_FDCWD, "/usr/lib64/libc.so.6", O_RDONLY|O_CLOEXEC <unfinished ...>
52484 <... openat resumed>)             = 3
52484 read(3 <unfinished ...>
52484 <... read resumed>, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\340v\2\0\0\0\0\0"..., 832) = 832
52484 pread64(3 <unfinished ...>
52484 <... pread64 resumed>, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 840, 64) = 840
52484 fstat(3 <unfinished ...>
52484 <... fstat resumed>, {st_mode=S_IFREG|0755, st_size=1892624, ...}) = 0
52484 mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0 <unfinished ...>
52484 <... mmap resumed>)               = 0x7fae2686f000
52484 pread64(3 <unfinished ...>
52484 <... pread64 resumed>, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 840, 64) = 840
52484 mmap(NULL, 1923952, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0 <unfinished ...>
52484 <... mmap resumed>)               = 0x7fae26699000
52484 mmap(0x7fae266bd000, 1388544, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x24000 <unfinished ...>
52484 <... mmap resumed>)               = 0x7fae266bd000
52484 mmap(0x7fae26810000, 331776, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x177000 <unfinished ...>
52484 <... mmap resumed>)               = 0x7fae26810000
52484 mmap(0x7fae26861000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1c7000 <unfinished ...>
52484 <... mmap resumed>)               = 0x7fae26861000
52484 mmap(0x7fae26867000, 31600, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0 <unfinished ...>
52484 <... mmap resumed>)               = 0x7fae26867000
52484 close(3 <unfinished ...>
52484 <... close resumed>)              = 0
52484 mmap(NULL, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0 <unfinished ...>
52484 <... mmap resumed>)               = 0x7fae26696000
52484 arch_prctl(ARCH_SET_FS, 0x7fae26696740 <unfinished ...>
52484 <... arch_prctl resumed>)         = 0
52484 set_tid_address(0x7fae26696d68 <unfinished ...>
52484 <... set_tid_address resumed>)    = 52484
52484 set_robust_list(0x7fae26696a20, 24 <unfinished ...>
52484 <... set_robust_list resumed>)    = 0
52484 rseq({cpu_id_start=0, cpu_id=RSEQ_CPU_ID_UNINITIALIZED, rseq_cs=NULL, flags=0, node_id=0, mm_cid=0, slice_ctrl={request=0, granted=0, __reserved=0}, __reserved=0}, 32, 0, 0x53053053 <unfinished ...>
52484 <... rseq resumed>)               = 0
52484 mprotect(0x7fae26861000, 16384, PROT_READ <unfinished ...>
52484 <... mprotect resumed>)           = 0
52484 mprotect(0x5635c58c0000, 4096, PROT_READ <unfinished ...>
52484 <... mprotect resumed>)           = 0
52484 mprotect(0x7fae268cc000, 8192, PROT_READ <unfinished ...>
52484 <... mprotect resumed>)           = 0
52484 prlimit64(0, RLIMIT_STACK, NULL <unfinished ...>
52484 <... prlimit64 resumed>, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
52484 getrandom( <unfinished ...>
52484 <... getrandom resumed>"\x69\x22\x24\xeb\xd8\xd3\x97\xde", 8, GRND_NONBLOCK) = 8
52484 munmap(0x7fae26871000, 129098 <unfinished ...>
52484 <... munmap resumed>)             = 0
52484 brk(NULL <unfinished ...>
52484 <... brk resumed>)                = 0x5635fcbe0000
52484 brk(0x5635fcc01000 <unfinished ...>
52484 <... brk resumed>)                = 0x5635fcc01000
52484 openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC <unfinished ...>
52484 <... openat resumed>)             = 3
52484 fstat(3 <unfinished ...>
52484 <... fstat resumed>, {st_mode=S_IFREG|0644, st_size=6245920, ...}) = 0
52484 mmap(NULL, 6245920, PROT_READ, MAP_PRIVATE, 3, 0 <unfinished ...>
52484 <... mmap resumed>)               = 0x7fae26000000
52484 close(3 <unfinished ...>
52484 <... close resumed>)              = 0
52484 fadvise64(0, 0, 0, POSIX_FADV_SEQUENTIAL <unfinished ...>
52484 <... fadvise64 resumed>)          = -1 ESPIPE (Illegal seek)
52484 fstat(0 <unfinished ...>
52484 <... fstat resumed>, {st_mode=S_IFIFO|0600, st_size=0, ...}) = 0
52484 read(0 <unfinished ...>
52484 <... read resumed>, "", 4096)     = 0
52484 close(0 <unfinished ...>
52484 <... close resumed>)              = 0
52484 close(1 <unfinished ...>
52484 <... close resumed>)              = 0
52484 close(2)                          = 0
52484 exit_group(0)                     = ?
52484 +++ exited with 0 +++
```

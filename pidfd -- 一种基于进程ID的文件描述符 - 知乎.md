# pidfd -- 一种基于进程ID的文件描述符 - 知乎
## 1 PIDFD 的由来

Linux(准确的是说应该是类 Unix 系统) 习惯于把系统中的对象用文件来表示，不过进程是个例外 --Linux 系统中使用整形类型 PID(process ID) 来表示一个进程（缺省最大是 32767，可通过 sysctl_pid_max 来进行修改）。

这种使用 pid 来表示进程方式存在着一些问题，其中最大问题是 PID 会被重复回收利用：当某个进程退出的时候，它释放的 PID 可能会在一段时间后会重新分配给其他新进程使用。这就会造成一种竞争问题 (称为 race-free process signaling)：例如进程 server 本来是通过进程 A 的 PID 给进程 A 发送信号，但是发送信号前进程 A 退出了，且进程 A 的 PID 号很快就被重复分配给了另外一个进程 B，此时 server 就会错把信号发送给了 B，严重的话会导致 B 进程被迫退出。  
针对这个问题 Linux 内核在 Linux-5.1 引入了一个新系统调用 pidfd_send_signal(2) 以通过操作 / proc/<pid>文件描述符的方式来解决该问题，参考[Toward race-free process signaling](https://lwn.net/Articles/773459/)；

在 Linux-5.2 中为 clone(2) 系统调用引入了 CLONE_PIDFD 标志来让用户更加容器的在进程创建时获取到进程的 pidfd，参考[Rethinking race-free process signaling](https://lwn.net/Articles/784831/) ；

在 Linux-5.3 中引入了一个新的系统调用 pidfd_open()，以解决旧应用在未使用 CLONE_PIDFD 标志来 clone() 进程时仍然可通过 pidfd_open() 来获取到该进程的 pidfd，参考：[New system calls: pidfd_open() and close_range()](https://lwn.net/Articles/789023/)。

后续的 Linux 版本又陆续完善支持了 pidfd 版本的 waitid()、 /proc/self/fdinfo、pidfd_getfd()、setns() 以及新的系统调用 process_madvise(2) 等等功能。

通这一系列的补丁，目前内核不仅能够比较完整的支持以前通过 PID 所完成的功能，还通过 pidfd 新引入了其他特。

## 2 PIDFD 基本原理

Pidfd 本质上是文件描述符，但是它在形式上没有一个对应的实际可见的文件路径 (dentry)，而是采用 anon_inode(匿名节点) 方式来实现。

与 PID(process ID) 的实现不同，pidfd 是通过专有的 pidfd open 函数 "打开" 目标进程 dst task，然后返回 dst task 对应的文件描述符 pidfd；在当前进程未主动调用 close() 函数关闭 pidfd 前不论 dst task 是否退出，pidfd 对于当前任务都是有效的，这就避免了 PID"race-free process signaling" 的问题。

### 2.1 PIDFD 的数据结构

Pidfd 的本质是一个文件描述符，因而 inode(这里使用的是匿名节点 anon_inode) 和 struct file 结构是其基础结构；Linux 内核为这类文件定义了 pidfd_fops 作为它专有的 file_operations，这样就为 pidfd 这类文件提供了它专属的文件行为。

有了 inode，struct file 和 file_operations 还不够，还得将 pidfd 与进程信息关联起来才能达到 pidfd 设计的目标，这是通过文件私有成员 file->private_data 来实现的。pidfd 是通过 clone() 或者专有的 open() 函数创建的。在这个创建过程中会直接或者间接传入目标进程的 PID，然后创建流程中 Linux 内核会调用通过函数 get_pid(struct pid \*pid) 将 PID 转换为 struct pid 指针并存放在 file->private_data 成员中；此外，引入了 pidfd 后为了实现与目标进程的同步机制，Linux 在 struct pid 中新增了等待队列 wait_pidfd 用于挂接同步在目标进程各个 pidfd 上的进程。

### 2.2 PIDFD 的运转

有了前面的数据结构作为铺垫，接下来 Linux 中通过 PIDFD 来与目标进程进行交互就水到渠成了。

使用 pidfd 作为参数调用相关的系统调用函数

就当前而言，Linux 与 c 库对 pidfd 的支持已经比较完善，不仅有普通的 poll()、waitid() 等函数支持，内核还新增了针对的 pidfd 特殊接口，如 pidfd_send_signal()、process_madvise() 等等。

系统调用陷入内核态后通过 pidfd 找到对应的 strut file

根据具体的系统调用取出需要的数据结构进行对应操作

这一步中根据具体的系统调用，要么调用文件操作函数 pidfd_fops 挂接具体回调函数，要么通过 file->private_data 取出 struct pid、再转换为对应的 struct task_struct 指针完成特定的指令。

## 3 PIDFD 实际应用案例

### 3.1 通过生成 pidfd

目前生成进程描述符最常用的两种方式是 pidfd_open(2) 和 CLONE_PIDFD 标志的 clone(2) or clone3(2) 函数，此外还可以通过 open() 打开 / proc/<pid>目录来获取，但是这种方式目前已经逐步废弃且功能也受限。下面看看函数 pidfd_open() 的原型：

```c
int pidfd_open(pid_t pid, unsigned int flags);
```

该函数生成一个进程 id 对应的文件描述符；需要注意的是 glibc 中没有提供该函数接口，需要通过 syscall 来实现。  
参考：

### 3.2 通过 pidfd 向进程发送信号 pidfd_send_signal

Linux 为 pidfd 专门定义了一个发送信号的系统调用，其原型如下：

```c
int pidfd_send_signal(int pidfd, int sig, siginfo_t *info, unsigned int flags)
```

该函数向 pidfd 文件描述符指定的进程发送信号 sig，需要注意的是 glibc 中没有提供该函数接口，需要通过 syscall 来实现。

参考：

### 3.3 poll(pidfd) 监控进程退出

在 pidfd 未出现之前，如果不是父子进程关系，进程是无法监控一个另外进程是否退出的；pidfd 机制的出现弥补了这一空缺需求。有了文件描述符，内核可很方便的实现 poll 机制，而 Linux 内核中也确实在 pidfd 文件操作函数 pidfd_fops 中实现了 poll，即 pidfd_poll()。

对于通过 poll() 监控进程退出的例子可以参考：

### 3.4 对其他进程进行 madvise

最后要跟大家介绍的域 pidfd 相关的函数是 process_madvise()。与经典的 madvise() 只能针对本进程不同，process_madvise() 可以针对其他进程进行内存的使用策略调整。

```c
ssize_t process_madvise(int pidfd, const struct iovec *iovec,
                               size_t vlen, int advice,
                               unsigned int flags);
```

该函数的描述如下：The process_madvise() system call is used to give advice or directions to the kernel about the address ranges of another process or of the calling process. It provides the advice for the address ranges described by iovec and vlen. The goal of such advice is to improve system or application performance.

参考：

## 4 pidfd 的限制

1.  目前 pidfd 只能通过进程 PID 而非线程进行使用
2.  glibc 还未提供 pidfd 相关的系统调用符号，只能通过 syscall() + 系统调用号的方式使用
3.  虽然 Linux 内核在很多 PID 使用的地方引入 pidfd 相同的功能接口，但是 pidfd 目前应该还无法完全替代 PID，二者是共存关系
4.  PIDFD 也需要遵守进程之间的权限核命名空间规则 
    [https://zhuanlan.zhihu.com/p/381302990](https://zhuanlan.zhihu.com/p/381302990)

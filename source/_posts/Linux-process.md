---
title: Linux process
date: 2018-05-17 19:29:06
tags:
- Linux
categories:
- Linux
---

{% asset_img linux-kernel-map.jpg Linux kernel map %}

### 进程

Linux内核中进程用task_struct结构体表示，称为进程描述符，该结构体相对比较复杂，有几百行代码，记载着该进程相关的所有信息，比如进程地址空间，进程状态，打开的文件等。对内核而言，进程或者线程都称为任务task。内核将所有进程放入一个双向循环链表结构的任务列表(task list)。

{% asset_img task_struct.jpg task_struct %}

Linux内核是抢占式多任务工作模式，进程大致分为两类（两者可相互转化）：
- 守护进程（服务）: daemon,由内核在系统引导过程中启动的进程，和终端无关进程；
- 前台进程：跟终端相关，通过终端启动的进程（用户进程）；

按进程占用资源的多少可以讲进程分为：
- CPU-Bound： CPU密集型（对CPU密集型是对cpu占用率高的进程），非交互；
- IO-Bound： IO密集型（等待I/O时间长的进程），交互；





### 进程的状态

{% asset_img ProcessState.png Process State %}

- TASK_RUNNING
运行态： running
就绪态： ready（可以运行但是没运行）

- TASK_INTERRUPTIBLE & TASK_UNINTERRUPTIBLE

在linux系统中，一个进程无法获得某种资源，如锁（自旋锁、互斥锁、顺序锁、信号量等）、信号、中断，将进入等待状态，同时一个进程也可以根据需要主动进入等待状态。将进程从运行状态迁移到等待状态的方式：

1. wait_event  
2. wait_event_timeout 
3. wait_event_interruptible 
4. wait_event_interruptible_timeout 

1和2函数将进程放人等待队列中，并将当前进程的状态设置为TASK_UNINTERRUPTIBLE，即在等待队列中的进程不可以被信号激活，而只能由中断事件激活；
3和4函数将进程放人等待队列中，并将当前进程的状态设置为TASK_INTERRUPTIBLE，即在等待队列中的进程可以被信号和中断事件激活；
2和4函数会为当前等待进程设置一个定时器，当等待进程在指定的时间内没有被信号或者中断激活时，这个定时器将激活等待进程。 

- TASK_STOPPED
进程被停止执行，当进程接收到SIGSTOP、SIGTTIN、SIGTSTP或者SIGTTOU信号之后就会进入该状态。

- EXIT_ZOMBIE
进程的执行被终止，但是其父进程还没有使用wait()等系统调用来获知它的终止信息，此时进程成为僵尸进程。

- EXIT_DEAD
进程的最终状态。



### 创建新进程

分为三类：
- Linux进程创建
- Linux用户级线程创建
- Linux内核线程创建

#### Linux进程创建

通过fork()及exec()系统调用创建进程。

- fork: 采用复制当前进程的方式来创建子进程，此时子进程与父进程的区别仅在于pid, ppid以及资源统计量(比如挂起的信号)。

- exec：读取可执行文件并载入地址空间执行；一般称之为exec函数族，有一系列exec开头的函数，比如execl, execve等。

fork过程复制资源包括代码段，数据段，堆，栈。fork调用者所在进程便是父进程，新创建的进程便是子进程；在fork调用结束，从内核返回两次，一次继续执行父进程，一次进入执行子进程。

{% asset_img fork_diagram.png FORK %}

进程内存段：

{% asset_img process_address_space.png Process address space %}

exec执行的例子(`ls`)

{% asset_img exec_ls.png exec ls %}



#### Linux用户级线程创建

通过pthread库中的pthread_create()创建线程，也并非”轻量级进程”，在Linux看来线程是一种进程间共享资源的方式，线程可看做是跟其他进程共享资源的进程。

{% asset_img do_fork.jpg %}

fork, vfork,clone根据不同参数调用do_fork：

- pthread_create: flags参数为 CLONE_VM, CLONE_FS, CLONE_FILES, CLONE_SIGHAND
- fork: flags参数为 SIGCHLD
- vfork: flags参数为 CLONE_VFORK, CLONE_VM, SIGCHLD

所以进程与线程最大的区别在于资源是否共享，线程间共享的资源主要包括内存地址空间，文件系统，已打开文件，信号等信息， 如下图蓝色部分的flags便是线程创建过程所必需的参数。

{% asset_img clone_flags.jpg %}

{% asset_img process_and_thread.png %}


#### Linux内核线程创建 
通过kthread_create()创建内核线程，最初线程是停止的，需要使用wake_up_process启动它。它没有独立的地址空间，即mm指向NULL。这样的线程只在内核运行，不会切换到用户空间。所有内核线程都是由kthreadd作为内核线程的祖师爷，衍生而来的。

Linux内核可以看作一个服务进程(管理软硬件资源，响应用户进程的种种合理以及不合理的请求)。内核需要多个执行流并行，为了防止可能的阻塞，支持多线程是必要的。内核线程就是内核的分身，一个分身可以处理一件特定事情。内核线程的调度由内核负责，一个内核线程处于阻塞状态时不影响其他的内核线程，因为其是调度的基本单位。内核线程是直接由内核本身启动的进程。内核线程实际上是将内核函数委托给独立的进程，它与内核中的其他进程"并行"执行。内核线程经常被称之为内核守护进程。

内核线程主要有两种类型：

1. 线程启动后一直等待，直至内核请求线程执行某一特定操作。

2. 线程启动后按周期性间隔运行，检测特定资源的使用，在用量超出或低于预置的限制时采取行动。

#### 总结

Linux使用task_struct来描述进程和线程：

一个进程由于其运行空间的不同, 从而有内核线程和用户进程的区分, 内核线程运行在内核空间, 之所以称之为线程是因为它没有虚拟地址空间, 只能访问内核的代码和数据, 而用户进程则运行在用户空间, 不能直接访问内核的数据但是可以通过中断, 系统调用等方式从用户态陷入内核态，但是内核态只是进程的一种状态, 与内核线程有本质区别。

用户进程运行在用户空间上, 而一些通过共享资源实现的一组进程我们称之为线程组, Linux下内核其实本质上没有线程的概念, Linux下线程其实上是与其他进程共享某些资源的进程而已。但是我们习惯上还是称他们为线程或者轻量级进程。

因此, Linux上进程分3种，内核线程（或者叫核心进程）、用户进程、用户线程, 当然如果更严谨的，你也可以认为用户进程和用户线程都是用户进程。

- 内核线程拥有 进程描述符、PID、进程正文段、核心堆栈

- 用户进程拥有 进程描述符、PID、进程正文段、核心堆栈 、用户空间的数据段和堆栈

- 用户线程拥有 进程描述符、PID、进程正文段、核心堆栈，同父进程共享用户空间的数据段和堆栈


### 进程调度

现在的操作系统都是多任务的，为了能让更多的任务能同时在系统上更好的运行，需要一个管理程序来管理计算机上同时运行的各个任务（也就是进程）。

这个管理程序就是调度程序，它的功能说起来很简单：

1. 决定哪些进程运行，哪些进程等待;

2. 决定每个进程运行多长时间;

此外，为了获得更好的用户体验，运行中的进程还可以立即被其他更紧急的进程打断。总之，调度是一个平衡的过程。一方面，它要保证各个运行的进程能够最大限度的使用CPU(即尽量少的切换进程，进程切换过多，CPU的时间会浪费在切换上)；另一方面，保证各个进程能公平的使用CPU(即防止一个进程长时间独占CPU的情况)。

{% asset_img context_switching.png %}

把进程区分为三类:

 类型 | 描述 | 示例 
:-------:|:-------:|:-------:
交互式进程(interactive process) | 此类进程经常与用户进行交互, 因此需要花费很多时间等待键盘和鼠标操作. 当接受了用户的输入后, 进程必须很快被唤醒, 否则用户会感觉系统反应迟钝 | shell, 文本编辑程序和图形应用程序
批处理进程(batch process) | 此类进程不必与用户交互, 因此经常在后台运行. 因为这样的进程不必很快相应, 因此常受到调度程序的怠慢 | 程序语言的编译程序, 数据库搜索引擎以及科学计算
实时进程(real-time process) | 这些进程由很强的调度需要, 这样的进程绝不会被低优先级的进程阻塞. 并且他们的响应时间要尽可能的短 | 视频音频应用程序, 机器人控制程序以及从物理传感器上收集数据的程序

实时进程：实时进程的优先级是静态设定的，而且始终大于普通进程的优先级。因此只有当runqueue中没有实时进程的情况下，普通进程才能够获得调度。实时进程采用两种调度策略，SCHED_FIFO 和 SCHED_RR，FIFO 采用先进先出的策略，对于所有相同优先级的进程，最先进入runqueue的进程总能优先获得调度；Round Robin采用更加公平的轮转策略，使得相同优先级的实时进程能够轮流获得调度。

调度算法的主要演化：O(n) -> O(1) -> CFS

观看以下两个video：
[Scheduling in Linux: O(n), O(1) Scheduler](https://www.youtube.com/watch?v=vF3KKMI3_1s)
[Completely Fair Scheduling (CFS)](https://www.youtube.com/watch?v=scfDOof9pww)



### 进程相关命令

`pstree, ps, pidof,pgrep, top, htop, glance, pmap, vmstat, dstat, kill,pkill, job, bg, fg, nohup`

### 进程通信机制

- 同一主机上
1. signal（信号）
2. shm: shared memory（分享内存）
3. semophore：信号量，一种计数器

- 不同主机上
1. rpc: remote procedure call(远程过程调用)
2. socket（套接字）: IP和端口号


### references

[Linux内核的整体架构](http://www.wowotech.net/linux_kenrel/11.html)

[Linux进程管理与调度-之-目录导航](https://blog.csdn.net/gatieme/article/details/51456569)

[Linux进程管理-gityuan](http://gityuan.com/2017/08/05/linux-process-fork/)

[Linux内核进程管理架构图](https://wenku.baidu.com/view/5550581f3186bceb18e8bb5a.html?from=search)
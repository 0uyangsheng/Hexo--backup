---
title: Embeded Linux kernel boot process
date: 2018-05-05 10:50:37
tags:
- Linux
categories:
- Linux
---

{% asset_img linux-kernel-map.jpg Linux kernel map %}

## 第一阶段（从跳转kernel image到跑到start_kernel之前）

kernel入口地址在ENTRY(stext)，代码在arch/arm/kernel/head.S。

如何体现的？

在arch/arm/kernel/vmlinux.lds.S中，同时参考编译后生成的System.map文件，System.map是内核的内核符号表，在这里可以找到函数地址，变量地址，包括一些链接过程中的地址定义等等。

其中要做的事情：

- 设置为SVC模式，关闭所有中断。

为什么要设置成SVC模式？

除了用户模式之外的其他6种处理器模式称为特权模式。特权模式下，程序可以访问所有的系统资源（除了特定模式下的影子寄存器），也可以任意地进行处理器模式的切换，特权模式中，除系统模式外，其他5种模式又称为异常模式。
而用户模式下访问的资源受限，故不能使用用户模式，系统模式的优先级低于异常模式，故不使用系统模式，快中断模式、中断模式、中止模式、未定义模式用于特殊场景下由CPU自动切入,故不使用,所以需要使用SVC模式。

为什么要关闭所有中断？

在启动过程中，中断环境并没有完全准备好，也就是中断向量表和中断处理函数并没有完成设置，一旦有中断产生，可能会导致预想不到的问题，或者是程序跑飞。因此，在准备好中断环境之前，需要关闭所有中断。

- 获取CPU ID，提取相应的proc info

这里存在的MMU标识，也就是我们需要在打开MMU之前需要先获取procinfo的原因，因为打开MMU之前需要配置临时内核页表，而配置临时内核页表需要这里的MMU标识来进行设置。

- 验证tags或者dtb

dtb里面存放了各种硬件信息，如果dtb有问题，会导致后续开机过程中读取的设备信息有问题而导致无法开机。
在生成dtb的时候会在头部上添加一个幻数magic，而验证dtb是否合法主要也就是看这个dtb的magic是否和预期的值一致。 

- 创建临时内核页表的页表项

为了打开MMU，内核需要创建一个临时内核页表，用于kenrel启动过程中的打开MMU的过渡阶段。 在打开MMU的过程中，CPU还是按照地址顺序一条接着一条去获取指令，也就是说此时PC指针还是指向这段代码区域的物理地址。
当MMU打开之后，如果没有恒等映射的话，PC指针此时还是指向这段代码区域的物理地址，但实际上CPU会把PC指针的地址当作虚拟地址进行操作，而造成找不到对应的物理地址。


- 使能MMU

- 跳转到start_kernel，也就是跳转到第二阶段，看其中寄存器说明，存的什么值
````
/*
* The following fragment of code is executed with the MMU on in MMU mode,
* and uses absolute addresses; this is not position independent.
*
*  r0  = cp#15 control register
*  r1  = machine ID
*  r2  = atags/dtb pointer
*  r9  = processor ID
*/
    __INIT
__mmap_switched:
    adr    r3, __mmap_switched_data

    ldmia    r3!, {r4, r5, r6, r7}
    cmp    r4, r5                @ Copy data segment if needed
1:    cmpne    r5, r6
    ldrne    fp, [r4], #4
    strne    fp, [r5], #4
    bne    1b

    mov    fp, #0                @ Clear BSS (and zero fp)
1:    cmp    r6, r7
    strcc    fp, [r6],#4
    bcc    1b

ARM(    ldmia    r3, {r4, r5, r6, r7, sp})
THUMB(    ldmia    r3, {r4, r5, r6, r7}    )
THUMB(    ldr    sp, [r3, #16]        )
    str    r9, [r4]            @ Save processor ID
    str    r1, [r5]            @ Save machine type
    str    r2, [r6]            @ Save atags pointer
    bic    r4, r0, #CR_A            @ Clear 'A' bit
    stmia    r7, {r0, r4}            @ Save control register values
    b    start_kernel
ENDPROC(__mmap_switched)
````

## 第二阶段 - start_kernel

以下只剖析个大概，不对细节做太深入的探讨。

代码路径：init/main.c  从start_kernel()开始，挑重点的介绍：

重要函数       |    作用
:-------------:    |  :------------------------:
setup_arch   |   architecture-specific setup ,如arch\arm\mach-sunxi\Sun8i.c 中MACHINE_START(SUNXI, "sun8i") 下相关
mm_init        | memory management
sched_init    |  调度相关
vfs_caches_init  |  file system, including kernfs, sysfs, rootfs, mount tree
rest_init        |  道生一（start_kernel-->cpu_idle），一生二（kernel_init和kthreadd），二生三（即前面0、1和2三个进程），三生万物（1号进程是所有用户态进程的祖先，2号进程是所有内核线程的祖先。可以通过ps命令，查看PID及PPID的关系。

其中大部分的工作是在rest_init中完成，对于Android系统来讲init进程在 system/core/init/init.c实现，下图代表Android系统的boot process：

{% asset_img Android-process.png android boot process %}

设备驱动的init及probe也是在其中实现的，如下面的调用栈：

````
`--> rest_init
    |
    `--> kernel_init
          |
          |--> kernel_init_freeable
          |    |
          |    |--> do_basic_setup
          |    |    |
          |    |    `--> do_initcalls
          |    |        :
          |    |        :--> early init calls
          |    |        :--> core init calls
          |    |        :--> post core init calls
          |    |        :--> arch init calls
          |    |        :    :
          |    |        :    `--> customize_machine
          |    |        :        |
          |    |        :        `--> machine_desc->init_machine
          |    |        :
          |    |        :--> subsys init calls
          |    |        :--> fs init calls
          |    |        :--> device init calls
          |    |        :    :
          |    |        :    `--> module_init() entries
          |    |        :        drivers are probed during driver registration
          |    |        :
          |    |        `--> late init calls
````
从以上也可以看出kernel的启动顺序。



----

References:

[kernel 启动流程](https://blog.csdn.net/ooonebook/article/details/52710290)

[深入淺出 start_kernel()](https://danielmaker.github.io/blog/linux/inside_start_kernel.html)










































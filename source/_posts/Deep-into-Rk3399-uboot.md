---
title: 谈谈 U-boot 启动流程
date: 2018-04-25 11:07:19
tags:
- uboot
categories:
- uboot
---

{% asset_img uboot-fun.jpg Lets kick-off %}

## 前面的话

很自然的疑问，uboot是干什么的？它内部的运行机制是什么？把这两个问题解释清楚也不容易，所谓‘费曼法则’：通过向别人清楚的解说一件事情，来确认自己真的弄懂了这件事。

写技术文章，想深入浅出，非大师（在这个领域有多年的积累和实践）不可为，很显然我没有，本文只是初浅的梳理，大多是资料的索引。目的是熟悉整个框架，出问题时方便debug。

以RK3399为例。

一切得从官方的[README](http://git.denx.de/?p=u-boot.git;a=blob_plain;f=README;hb=HEAD)开始，其中有对uboot完整的描述，从文件目录介绍、代码流程、编译等等各方面都有涉及，建议通读一遍。

然后是Rockchip官方 [opensource](http://opensource.rock-chips.com/wiki_Main_Page) ，其中有开源代码链接，及[boot flow](http://opensource.rock-chips.com/wiki_Boot_option)介绍。

也许看完以上资料，可能还是懵懵懂懂，或许网上还有关于其他平台更为详细的资料介绍呢？search一番，发现samsung S5PV210文档最多，官方资料最详尽，是ARMV7架构，未加入Arm trust Firmware。

简而言之，uboot是在操作系统内核运行之前运行，可以初始化硬件设备、建立内存空间映射图，从而将系统的软硬件环境带到一个合适状态，以便为最终调用操作系统内核准备好正确的环境。

下面，从一些基础讲起，然后到网上分析比较多的Samsung S5PV210 Uboot，最后到RK3399的Uboot相关。

## 基础

- Text、Data、BSS、Heap、Stack的区别

先来看一幅图：

{% asset_img memmap.png Text Data BSS Heap Stack %}

简单来说，Text是存放代码的，Data是存放已经初始化的全局变量，BSS是未初始化或是0的全局变量，Heap是由malloc等分配的堆，Stack是函数的局部变量或是函数返回值。

对于Uboot来讲，要特别关注链接脚本`arch\arm\cpu\u-boot.lds`，其中各段的分配情况，及代码重定位。

- NorFlash、eMMC、Nand flash、SRAM、SDRAM的区别

| 存储器              |  上电后访问状态                         |     掉电后存储器中数据状态 |
| :-----------:            | :-------------------:                            |  :----------------------------------: |
|Nor                     | 可以读取数据，无法直接写入               | 数据存在                              |
| Nand /eMMC     | 初始化后才能读取写入     | 数据存在                              |
| SRAM                | 可以读取写入数据                          | 数据不存在                              |
| SDRAM             | 上电后没有初始化DDR控制器无法进行数据读写   | 数据不存在      |

那么以上存储器在uboot启动过程中都扮演什么角色呢？

Norflash作为arm处理器程序存储器。可以试想一下，如果程序存储器掉电以后里面的数据没有了。那么你的电脑如何自启动，难道每次开机前都要重新烧写一次代码。在此处可以思考一个问题，在上电后norflash可以看作一个可以随机读取的只读存储器。但是我们运行的程序，一般情况下.text段（代码段）是只读（ok），.rodata（只读数据段）是只读（也ok）。那么问题来了，对于.data段（数据段）和.bss段(未初始化的全局变量和静态变量)在程序运行的过程中变量的值是需要改变的（改变一个变量的值在底层硬件完成操作<在相应的地址（变量在物理地址上存储地址）上写入数据>），很可惜Norflash只能直接读取无法直接进行写操作。（重要！  怎么解决这个问题？  这时就需要SRAM  因为SRAM上电后就可以直接去读写，下面我就解释下SRAM的功能和作用。

SRAM特性：掉电易失（失去电源供电后SRAM里存储的数据不存在了），可以随意<读写>数据。（容量小，程序运行速度快，价格高，一般在SoC里面。).在实际运行时，SRAM可以作为c语言运行时的堆栈空间。把arm处理器的sp（堆栈指针寄存器）设置在sram中，为c语言运行提供环境。关于全局变量的问题，我单独提一下，uboot在重定位前（将uboot镜像从flash搬运到ddr中继续运行前），无论是汇编还是c程序中没有定义全局变量。只是定义了一个结构体指针gd_t *gd_ptr用于存储uboot镜像的长度，重定位地址等信息，将gd_ptr的地址存储在r9中，r9中存储的地址值为sram顶端减去一个sizeof（gd_t ）。（存储在sram里就可以随意读写了嘛    后面分析uboot代码时我会详细讲解）。

SDRAM特性：掉电易失（失去电源供电后SDRAM里存储的数据不存在了），上电后没有初始化DDR控制器，无法进行数据读写。既然需要使用大容量的SDRAM，必须配置ddr时钟和ddr控制器的寄存器。这一步在哪完成呢？（思考一下） 没错就是在norflash和SRAM搭建的程序运行环境中完成。完成什么呢？  1.完成对处理器时钟的初始化 2. DDR的初始化  3.给gd_t *gd_ptr赋值 （用于存储uboot镜像的长度，重定位地址，重定位偏移量等信息）。在uboot搬运到DDR中运行前进行最小系统的初始化，之后就将uboot搬运到ddr中进行运行。（重要！此时Norfalsh和SRAM的任务就完成了（这俩就没用了），现在uboot就在ddr中运行了）。

Nand/SPI/eMMC：它们的I/O接口并没有随机存取外部地址总线，无法向SRAM随机访问地址上的数据，它必须以区块性的方式进行读取，而norflash带有SRAM接口，有足够的地址引脚来寻址，可以很容易地<读取>其内部的每一个字节。所以通常的方式是一上电的过程中自动缓存Nand中的Uboot数据到SRAM中，然后在SRAM中执行，在其中完成DDR、NAND的初始化，这时可以将代码搬到SDRAM中运行。

- 编译地址、运行地址

编译地址： 32位的处理器，它的每一条指令是4个字节，以4个字节存储顺序，进行顺序执行，CPU是顺序执行的，只要没发生什么跳转，它会顺序进行执行行， 编译器会对每一条指令分配一个编译地址，这是编译器分配的，在编译过程中分配的地址，我们称之为编译地址。

运行地址：是指程序指令真正运行的地址，是由用户指定的，用户将运行地址烧录到哪里，哪里就是运行的地址。

在relocation过程中，不是简单的将编译地址完整按顺序的搬到SDRAM中的运行地址空间的，而是会有一定的计算重新排布，参考[uboot的relocation原理详细分析](https://blog.csdn.net/skyflying2012/article/details/37660265)。

好，接下来我们来看看不带Trust 但支持SPL功能传统Uboot的启动模式，以S5PV210为例。

## S5PV210 U-Boot Boot Flow

一图胜千言。参考三星官方[Internal ROM Booting](https://github.com/limingth/ARM-Resources/blob/master/tiny210/Datasheet/S5PV210_iROM_ApplicationNote_Preliminary_20091126.pdf)

{% asset_img v210.png v210 boot flow %}

{% asset_img uboot2.jpg BL01 boot flow%}

{% asset_img uboot3.jpg BL02 boot flow%}

{% asset_img uboot1.jpg v210启动总结%}

是不是有一种感觉，SPL有点多余？

这个主要原因是对于一些SOC来说，它的内部SRAM可能会比较小，小到无法装载下一个完整的uboot镜像，那么就需要spl，它主要负责初始化外部RAM和环境，并加载真正的uboot镜像到外部RAM中来执行。所以由此来看，SPL应该是一个非常小的loader程序，可以运行于SOC的内部SRAM中，它的主要功能就是加载真正的uboot并运行之。

## Rk3399 U-boot Boot Flow

还是一图胜千言。

{% asset_img 3399-uboot.png Rk3399 boot flow %} 

从上图可以看出：

Boot Flow 1 is typical Rockchip boot flow with Rockchip miniloader（即是RK特殊定制的方式）；

Boot Flow 2 is used for most SoCs with U-Boot TPL for ddr init and SPL for trust(ATF/OP-TEE) load and run into next stage（常规的做法）；

Boot Flow 3 is use only for RK3399 with SPL ATF support（常规的做法）；

Boot Flow 4 is used for armv7 SoCs which do not support trust（不带trust，和上面三星v210类似）；

再加上ATF后的理解：

{% asset_img 3399-boot.png Rk3399 boot flow %} 

## 写在后面

其实还有很多主题可以完善，如 到底是如何relocation的，如何boot Linux image，uboot是如何编译的，driver的架构如何，ATF里面的psci如何操作休眠唤醒的。

待…………


参考：
[深入理解uboot 2016 - 基础篇（处理器启动流程分析](https://blog.csdn.net/kernel_yx/article/details/53045424)
[tiny210(s5pv210)上电启动流程（BL0-BL2）](https://blog.csdn.net/ooonebook/article/details/52916202)
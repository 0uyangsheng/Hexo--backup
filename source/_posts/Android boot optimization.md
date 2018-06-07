---
title: Android开机启动优化
date: 2018-05-14 20:37:32
tags:
- Android
categories:
- Android
---

{% asset_img fast.jpg Fast android %}

## 开机流程

如下图：

{% asset_img boot.png Android boot process%}

## Linux优化  

可以通过添加打印module init的log，来check每个module初始化时的时间。从而找到花费时间比较多的module:
````
--- a/init/main.c
+++ b/init/main.c
@@ -785,7 +785,7 @@ int __init_or_module
do_one_initcall(initcall_t fn)
        if (initcall_blacklisted(fn))
                return
-EPERM;

-      if (initcall_debug)
+      if (1)
                ret =
do_one_initcall_debug(fn);

````
优化方案：
1. 通过一个比gzip更快的方式去解压内核镜像；
2. 去掉系统中一些不必要的log打印；
3. 去掉一些系统中不需要的驱动模块；
4. 启动时即以最大频率（cpu/DDR）且多核一起跑；


## Android优化

查看时间：

`adb logcat -v threadtime -b events > logcat_envents.txt`

`adb logcat -v threadtime > logcat.txt`

具体到全志H5平台，查看LOG发现：

````
01-01 03:00:23.233  1503  1503 I auditd  : type=2000 audit(0.0:1): initialized

01-01 03:00:24.730  1503  1503 I auditd  : type=1403 audit(0.0:2): policy loaded auid=4294967295 ses=4294967295

01-01 03:00:31.590  1520  1520 I boot_progress_start: 8846 //systemclock.uptimemillis(),开机到当前时间，毫秒。

//Zygote 进程preload 开始时间 32bit zygote

01-01 03:00:34.168  1520  1520 I boot_progress_preload_start: 11425

//Zygote 进程preload 结束时间32bit zygot

01-01 03:00:37.394  1520  1520 I boot_progress_preload_end: 14650

//System server 开始运行时间

01-01 03:00:37.780  1964  1964 I boot_progress_system_run: 15037

//Package Scan 开始

01-01 03:00:38.553  1964  1964 I boot_progress_pms_start: 15810

//System 目录开始scan

01-01 03:00:38.931  1964  1964 I boot_progress_pms_system_scan_start: 16188

//data 目录开始scan

01-01 03:00:42.210  1964  1964 I boot_progress_pms_data_scan_start: 19467

//package scan 结束时间

01-01 03:00:42.230  1964  1964 I boot_progress_pms_scan_end: 19487

//package manager ready

01-01 03:00:42.727  1964  1964 I boot_progress_pms_ready: 19984

//Activity manager ready，这个事件之后便会启动home Activity。

01-01 03:00:45.432  1964  1964 I boot_progress_ams_ready: 22689

//HomeActivity 启动完毕，系统将检查目前所有的window是否画完，如果所有的window（包括wallpaper， Keyguard 等）都已经画好，系统会设置属性service.bootanim.exit值为1.并且trigger下面的event。

01-01 03:00:49.393  1964  2003 I boot_progress_enable_screen: 26650

//SF设置service.bootanim.exit属性值为1，标志系统要结束开机动画了，可以用来跟踪开机动画结尾部分消耗的时间

01-01 03:00:49.545  1525  1527 I sf_stop_bootanim: 26801
01-01 03:00:49.579  1964  2020 I wm_boot_animation_done: 26836

// 打开Launcher APP
01-01 03:00:50.516  1964  2429 I am_create_activity: [0,14996279,5,tv.lfstrm.smotreshka_launcher/tv.lfstrm.mediaapp_launcher.MainActivity,android.intent.action.MAIN,NULL,NULL,268435712]

````

用bootchart 图形化显示Android启动过程，如图

{% asset_img bootchart.png Allwinner H5 android7.1 bootchart%}

#### 定制本地服务

Init程序的log信息位于kernel Log中，通过检索“init starting”，我们可以找到init进程启动了哪些本地服务，如：

````
[    5.632951] init: Starting service 'logd-reinit'...
[    5.635827] init: Starting service 'zygote'...
[    5.637221] init: Starting service 'netd'...
[    5.643302] init: Starting service 'healthd'...
[    5.644695] init: Starting service 'lmkd'...
[    5.646006] init: Starting service 'servicemanager'...
[    5.647547] init: Starting service 'surfaceflinger'...
[    5.752483] init: Starting service 'console'...
[    6.154863] init: Starting service 'audioserver'...
[    6.156551] init: Starting service 'cameraserver'...
[    6.158068] init: Starting service 'displayd'...
[    6.160115] init: Starting service 'drm'...
[    6.161186] init: Starting service 'gpio'...
[    6.162746] init: Starting service 'installd'...
[    6.164462] init: Starting service 'keystore'...
[    6.165992] init: Starting service 'mediacodec'...
[    6.167700] init: Starting service 'mediadrm'...
[    6.169339] init: Starting service 'mediaextractor'...
[    6.171560] init: Starting service 'media'...
[    6.172651] init: Starting service 'multi_ir'...
[    6.174182] init: Starting service 'ril-daemon'...
[    6.175224] init: Starting service 'systemmix'...
[    6.399317] init: Starting service 'adbd'...
[    6.733571] init: Starting service 'bootanim'...

````

Init进程解析init.rc及init.xxx.rc之类的文件，启动一些本地服务，如果我们的设备中没有电话模块、蓝牙模块，我们可以将这些没用的本地服务在init.rc里注释掉。笔者做了对比，去掉几个本地服务与没有去掉本地服务，二者在开机时间上几乎没有减少多少，这也可以理解，因为本地服务就是几个程序，少执行和多执行几个程序对于总体开机时间没有多大影响，不过，去掉没有使用的本地服务，对整个系统性能来说，会有微不足道的提升。

优化建议：
1. 去掉开机动画服务(service bootanim /system/bin/bootanimation)可以一定程度上提高系统的启动速度。
2. 可以通过在execute_one_command函数中统计测量 ，比如大于100ms的命令打印出来，再分析定位原因，这里命令执行时间长基本算BUG。

#### preloaded classes & resources

全志H5所花费时间：

````
01-01 03:00:12.869  1520  1520 I Zygote  : Preloading classes...
01-01 03:00:14.982  1520  1520 I Zygote  : ...preloaded 4162 classes in 2113ms.
01-01 03:00:15.683  1520  1520 I Zygote  : Preloading resources...
01-01 03:00:15.879  1520  1520 I Zygote  : ...preloaded 114 resources in 196ms.
01-01 03:00:15.887  1520  1520 I Zygote  : ...preloaded 41 resources in 8ms.
````

Android系统为了提高应用程序的启动速度，会在Zygote进程初始化过程中加载一些常用的java class和资源文件到进程的内存中，从而共享常用的class和resourse资源。

preloaded-classes list（`frameworks\base\preloaded-classes`）中预加载的类位于dalvik zygote进程的heap中。在zygote衍生一个新的dalvik进程后，新进程只需加载heap中没有预加载的类（这些后加载进来的类成为该进程所private独有的），这样便加快了应用程序的启动速度。实际上这是一种以空间换时间的办法，因为几乎没有一个应用程序能够使用到所有的预加载类，必定有很多类对于该应用程序来说是冗余的。但是也正如Google所说，智能手机开机远没有启动应用程序频繁——用户开机一次，但直到下次再开机之前可能要运行多个应用程序。因此牺牲一点启动时间来换取应用程序加载时的较快速度是合算的。

preloaded-classes list已经是Google Android工程师使用众多测试工具分析，加以手动微调后形成的最优化预加载列表，涵盖了智能机上最长见的应用类型所需要的各种类。很难想象我们自己能够有什么手段能够获得比这样更优的一个预加载列表。所以，除非你的Android系统是被移植到非智能手机设备上使用（例如MID、EBOOK，可以不需要Telephony相关的类），不建议去“优化”preloaded-classes list。

优化建议：
1. preloadClasses()与preloadResources()可以放到两个线程里面跑。
2. 修改zygote的nice值，及thread priority。

#### 定制Android系统服务

由Android的启动过程可知，init进程启动了app_process作为zygote，在app_process里启动了Dalvik虚拟机，然后加载执行了第一个Java程序ZygoteInit作为Dalvik主线程，在ZygoteInit里fork了第一个Java程序SystemServer，在SystemServer里启动了大量的Android的核心服务，通常来说这些服务一般不要去动，如果我们的设备里没有使用过某些服务，并且将来也明确不使用，可以将其去掉。

SystemServer启动了哪些Android服务：

````
PowerManagerService：电源管理服务  
ActivityManagerService：最核心服务之一，Activity管理服务
PackageManagerService：程序包管理服务 
WindowManagerService：窗口管理服务  
BluetoothService：蓝牙服务  
WifiP2pService：Wifi点对点直联服务  
WifiService：WIFI服务  
ConnectivityService：网络连接状态服务  
MountService：磁盘加载服务，通常也mountd和vold服务结合 
AudioService：AudioFlinger上层的封装的音量控制管理服务  
UsbService：USB Host和device管理服务 

````
优化这些services其实就是剔除我们不需要的一些services，而且不仅仅是修改SystemServer.java的问题，任何使用到被优化剔除掉的服务的代码都必须加以修改，否则系统肯定是起不来的。这样工作量大，而且难度也不小，并且有一定风险。因此对这些services的优化要慎之又慎。

#### PackageManagerService扫描、检查APK安装包信息

PMS对/system/framework，/system/app，/data/app，/data/app-private目录中的APK扫描耗费了大量的时间，如果预置的三方应用很多，这样启动的时间就会越长。

优化建议：

1. /system/app下的应用，如果是预置应用，在Android.mk建议加上LOCAL_DEX_PREOPT := true控制，在/system/vendor下的预置应用，如果此应用编译时间比较长的，也使用上LOCAL_DEX_PREOPT := true

2. 尽量减少data区内置app的数量，这个会严重影响开机速度，特别是第一次的开机速度。放在system的app  尽量生成odex  这样会加快开机速度。

#### Readahead

因为IO慢的原因（cpu与存储类设备如emmc通讯），有两段耗时的地方：1. Zygote的preload 资源和class；2. PackageManagerService的包扫描。可以采用Linux上使用较多的readahead机制，大概原理是：

1. 统计开机过程中，读取的块数据信息，记录下来保存；

2. 再次开机，通过记录下来的块数据读取信息，直接起一个服务，预先开始读，zygote或packagemanagerservice要读文件的时候，文件数据已经在cache中了。

这样主要IO时间，跑到readahead进程去了。


## reference：

[深入浅出 - Android系统移植与平台开发（六）－ 为Android启动加速](https://blog.csdn.net/mr_raptor/article/details/8006721)

[Android开机速度优化简单回顾——readahead](https://blog.csdn.net/lqxandroid2012/article/details/54317298)

[android 5.1.1开机优化(framework层)](https://blog.csdn.net/xxm282828/article/details/49095839)

[Linux性能调优指南](https://legacy.gitbook.com/book/lihz1990/transoflptg/details)

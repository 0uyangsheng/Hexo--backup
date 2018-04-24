---
title: 从头开始移植Ubuntu系统到ARM平台（基于全志H3）
date: 2018-04-20 16:35:17
tags:
- Linux
categories:
- Linux
---

{% asset_img linux.jpg %}

### 起点

在ARM SOC上移植Ubuntu系统并不是一件容易的事情，要对镜像文件的组成，系统的启动顺序非常熟悉。在网上searching一番，发现一个很神奇的网站->[Armbian](https://docs.armbian.com/Developer-Guide_Build-Preparation/),这个网站开发者在大量的开发板上做了移植工作，包括allwinner H3/H5、Rockchip RK3328、amlogic S905x等，并且有详细的文档，以及开源的编译系统。

其实，参考Armbian的文档，即可搭建好完整编译环境，并针对你的板子（前提是里面已经支持的SoC）修改uboot、kernel、编译脚本即可定制，都是开源的。我尝试在H3平台上搭建，并成功制作了镜像，进到了系统。

### 编译系统

如果需要定制一些功能，如添加、删除一些脚本，应用程序，就必须完整看懂整个编译逻辑。Armbian上介绍是：

{% asset_img process.png %}

从[khadas开发板网站](http://docs.khadas.com/social/MapoutBuildUbuntuFromScratch/) 上有更形象的图（但未必准确，尤其是对Initramfs与inittrd的理解）：

{% asset_img map.png %}

对照编译脚本来看：

从根目录`compile.sh`开始，进入`main.sh`，主要工作在后者里完成，关键步骤：

```
1# Check and install dependencies, directory structure and settings
prepare_host
...........
2# 下载uboot及kernel
fetch_from_repo "$BOOTSOURCE" "$BOOTDIR" "$BOOTBRANCH" "yes"
fetch_from_repo "$KERNELSOURCE" "$KERNELDIR" "$KERNELBRANCH" "yes"
...........
3# Compile u-boot if packed .deb does not exist
compile_uboot
...........
4# Compile kernel if packed .deb does not exist
compile_kernel
...........
5# create board support package /create desktop package / build additional packages
create_board_package
create_desktop_package
chroot_build_packages
...........
6# Starting rootfs and image building process
debootstrap_ng
...........
7# make the image
prepare_partitions
create_image
```

### 后记

1. 里面有非常多的细节，要花些时间看明白，尤其要对shell scripts比较熟悉。
2. ramfs、Initramfs、ramdisk、inittrd、rootfs、tmpfs的区别，参考[ramfs-rootfs-initramfs](https://www.kernel.org/doc/Documentation/filesystems/ramfs-rootfs-initramfs.txt)。看了半天貌似也没看懂。简单来说，ramdisk是一种基于ram的块设备，ramfs是一种基于ram的文件系统，开发ramfs的目的是因为ramdisk浪费了太多的内存cache页。initrd是init ramdisk的缩写，initramfs是init ramfs的缩写。名称里加了init前缀，代表它们具有了引导内核启动的功能。
3. 自己做的过程中，添加了一个自动挂载硬盘的功能，通过udev，但发现FAT32可以挂载，NTFS不能，网上searching一番并多次尝试，最终找到可行解决方案。参考如下[自动挂载](https://serverfault.com/a/767079)。也有第三方的解决方案，如autofs, HAL, udisks, udisks2, usbmount，并未尝试。另，一些有用的调试命令：

```
udevadm info /dev/sda1 --此命令可以查看相关设备的udev属性，依据此来写rules。
udevadm monitor --udev  --观察 uevent事件
blkid   fdisk   lsblk
```

以上只是一些索引记录，供参考。





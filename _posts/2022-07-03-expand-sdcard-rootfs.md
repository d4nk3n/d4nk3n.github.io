---
title: "拓展SD卡的根分区"
math: true
categories: 
  - "Computer"
tags: 
  - "D1"
  - "Dev board"
  - "SD Card"
---

## 前言

作为（购买）开发板爱好者，我一直想搞个带MMU的RISC-V板子。之前有一个叫[Nezha](https://linux-sunxi.org/Allwinner_Nezha)就挺不错，不过它价格比树莓派贵不少，而性能一言难尽（用的全志的D1-H@1GHz，平头哥C906架构，单核单发顺序5段流水）。众所周知等等党永远不亏，后来Sipeed发布了一个叫[Lichee RV Dock](https://linux-sunxi.org/Sipeed_Lichee_RV)的小板子，只需要140块钱就能买个能启动Linux的开发板，还是比较划算的。

不过这种开发板一般文档和配套软件很有限，我烧写了Sipeed官网提供的Tina Linux镜像（好像不能用dd烧写，要用它专门的软件），启动之后过段时间会因为无线网卡有关的问题直接卡死，而且没有包管理器也不太好用。好在有人在GitHub上分享了生成Debian镜像的脚本，还提供了预编译好的[镜像](https://github.com/tmolteno/d1_build/releases/tag/v0.3.1)，所以我准备用这个现成的Debian镜像试试看。

烧写之后运行Debian过程中没有遇到什么问题（就是性能捉急，有点卡），不过预编译的镜像是给8G的SD卡用的，烧写到16G的卡上会浪费一半空间。所以我想试试看怎么拓展它的根分区。

![](/uploads/img/posts/2022-07-03-expand-sdcard-rootfs/uname-neofetch.png)
_显示系统信息_

## 拓展SD卡根分区

一般SD卡名字都叫`mmcblk0`（如果只有一张卡的话），可以用`sudo fdisk /dev/mmcblk0`然后输入命令`p`查看它的分区。

目前我的SD卡有3个分区：首先是`mmcblk0p1`，挂载到`/boot`分区，包含了Linux内核镜像等内容；然后是挂载到根目录的`mmcblk0p2`，里面是根目录下的文件；最后是swap分区。考虑到这个开发板内存只有512M，整个交换空间还是有必要的。

为了拓展空间，就需要把交换分区挪到SD卡末尾的空间去，再拓展根分区的空间。步骤如下：

1. 先输入`sudo swapoff -a`关闭交换分区；
2. 把swap分区删了；
3. 计算好给swap预留的大小，拓展`mmcblk0p2`的大小；
4. 拓展根分区；
5. 重新创建swap分区；
6. 使用`sudo mkswap /dev/mmcblk0p3`初始化swap分区；
7. 写入`/etc/fstab`文件设置swap信息；
8. 输入`sudo swapon -a`开启交换分区；
9. 如果输入`df -h`没有观察到rootfs空间变化的话，可以试试`sudo resize2fs /dev/mmcblk0p2`。

修改分区可以用`fdisk`来完成，如果你有Ubuntu等桌面系统，还可以用Disks等桌面软件操作。

我使用的`fdisk`创建分区时跟网上查到的教程不同，需要在创建之后再指定分区类型。

用到的`fdisk`命令：
* `m`：显示帮助
* `p`：显示分区信息
* `d`：删除分区
* `n`：创建一个新分区，默认是`Linux filesystem`
* `t`：修改分区类型，输入编号指定类型，也可以输入别名（比如`swap`）指定
* `w`：应用更改
* `q`：退出

最后的分区效果:

```
rv@lichee:/$ sudo fdisk /dev/mmcblk0

Welcome to fdisk (util-linux 2.38).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

This disk is currently in use - repartitioning is probably a bad idea.
It's recommended to umount all file systems, and swapoff all swap
partitions on this disk.


Command (m for help): p

Disk /dev/mmcblk0: 14.84 GiB, 15931539456 bytes, 31116288 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 097C6CDB-3A07-4AFD-BEDC-93BF1530C575

Device            Start      End  Sectors  Size Type
/dev/mmcblk0p1    81920   204799   122880   60M Linux filesystem
/dev/mmcblk0p2   204800 28968959 28764160 13.7G Linux filesystem
/dev/mmcblk0p3 28968960 31116254  2147295    1G Linux swap
```

`/ect/fstab`文件内容如下：
```
# <device>        <dir>        <type>        <options>            <dump> <pass>
/dev/mmcblk0p1    /boot        ext2          rw,defaults,noatime  1      1
/dev/mmcblk0p2    /            ext4          rw,defaults,noatime  1      1
/dev/mmcblk0p3    none         swap          sw                   0      0
```

## 其他

* 树莓派的[raspi-config](https://github.com/RPi-Distro/raspi-config)提供有拓展rootfs的脚本，如果要批量操作考虑可以考虑；
* 这个开发板其实可以驱动4K显示器，不过我觉得用APT装软件都有点慢……
* 平头哥开源了C906的RTL代码，不过像是代码生成器生成的（也能读），而且也并不包含C906的所有部分（比如向量拓展），我想开发时Debug有点用；
* 为什么全志D1和平头哥C906的手册都需要注册才能下载？
* 如果你想找个文档更丰富，性能更好的RISC-V开发板，可以等等BeagleV。

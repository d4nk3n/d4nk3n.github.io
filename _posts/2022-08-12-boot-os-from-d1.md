---
title: "在全志D1上启动操作系统"
math: true
categories: 
  - "Computer"
tags: 
  - "D1"
  - "Dev board"
  - "RISC V"
---

## 1. 前言

之前提到，[Sipeed Lichee RV](https://linux-sunxi.org/Sipeed_Lichee_RV)看起来是一个性价比很高的RISC-V开发板，虽然性能相对孱弱，但起码能启动像Debian这样的Linux发行版，其实也够用。

当然我想除了启动Debian或者Arch Linux之外，肯定也有人想在这个开发板上启动一个自己编写的OS，毕竟在QEMU上运行跟在物理机上还是不一样的。之前有人在Reddit提出了[这个问题](https://www.reddit.com/r/RISCV/comments/ufdcg9/booting_on_licheerv_d1/)，不过看回复的样子，他们也不太清楚。

## 2. 全志D1芯片的启动过程

首先先搞懂这个Allwinner D1芯片的启动流程，根据Datasheet（可在全志官网或者<https://linux-sunxi.org/D1>查看）和[Sunxi-Allwinner_Nezha](https://linux-sunxi.org/Allwinner_Nezha)可以大致了解芯片的启动过程。

### 2.1 BROM

CPU启动后，首先从地址0x0的位置取指令，这个位置是CPU内部的片上存储Boot ROM(BROM)，地址范围是从0x0开始的48KiB空间。启动后检查开发板上的FEL按键是否按下，如果按下则进入FEL模式（手动升级模式或者说Recovery模式），没有按下则尝试从SD卡加载BOOT0，加载的位置为SD卡的第16或者256扇区，不过一般GPT分区模式可能会使用第16扇区，所以最好把BOOT0烧写到256扇区去。

BROM的源代码没有公开，不过对BROM的一些逆向工程可以在[GitHub](https://github.com/smaeul/sunxi-blobs/tree/master/sun20iw1p1/rvbrom)上找到。

### 2.2 BOOT0

BOOT0用来初始化DDR和一些外设，然后加载下一级Bootloader，这个在启动U-Boot的时候也作为SPL(Secondary Program Loader)，目前版本的BOOT0从SD卡中加载一个包含了OpenSBI和U-Boot的TOC1格式的镜像，BOOT0会尝试从SD卡的16MiB的位置加载TOC1镜像，加载失败则会进入FEL模式。

BOOT0的代码也在GitHub上：<https://github.com/smaeul/sun20i_d1_spl>

### 2.3 OpenSBI和U-Boot

BOOT0尝试加载的TOC1镜像里包含了OpenSBI和U-Boot，首先运行的是OpenSBI(Open Supervisor Binary Interface)，可以近似理解为固件，提供了一些环境调用，比如控制台收发字符，关机等功能，之后的U-Boot可以初始化更多外设，可以从网络或者U盘等设备加载OS。

## 3. 使用XFEL启动

可不可以不从SD卡启动呢？之前提到这个开发板有一个FEL模式，有一个非官方的工具[xfel](https://github.com/xboot/xfel)也可以完成DDR和串口初始化、加载文件到内存等功能，这样就不需要向SD卡烧写BOOT0、OpenSBI和U-Boot了。

现在已经有人已经移植了MIT的xv6到D1上(<https://github.com/michaelengel/xv6-d1>)，编译好之后，可以通过xfel初始化内存，然后将其加载到开发板的DDR3内存起始地址0x40000000处：

```
PS C:\> xfel ddr d1
Initial ddr controller succeeded
PS C:\> xfel write 0x40000000 .\kernel.bin
100% [================================================] 1.010 MB, 461.421 KB/s
PS C:\> xfel exec 0x40000000
```

之后可以从串口看到xv6打印的信息，如果发现输出混乱，是因为输出的换号符号只有LF，可以在串口终端设置里补上CR符号。

![](/uploads/img/posts/2022-08-12-boot-os-from-d1/xfel-start-xv6.png){: width="75%" .shadow }
_启动xv6_

xv6运行在RISC-V的M模式下，可以直接加载运行。如果你的OS运行在S模式，也可以用xfel先加载编译好的OpenSBI和OS到内存，再执行命令运行OpenSBI。

注意Lichee RV开发板的内存大小为512MB，对应的物理地址从`0x40000000`开始。

## 4. 使用OpenSBI和U-Boot

这部分介绍编译Bootloader的过程。这部分的命令参考了项目<https://github.com/tmolteno/d1_build/>的内容。

### 4.1 安装编译依赖
```bash
# Install the prerequisites
sudo apt update
sudo apt install -y autoconf automake autotools-dev curl \
                    python3-setuptools python3 libmpc-dev \
                    libmpfr-dev libgmp-dev gawk build-essential \
                    bison flex texinfo gperf libtool patchutils \
                    bc zlib1g-dev libexpat-dev swig libssl-dev \
                    python3-distutils python3-dev git \
                    gcc-riscv64-linux-gnu g++-riscv64-linux-gnu
```

### 4.2 编译BOOT0
```bash
# Build SPL for D1
git clone --branch mainline https://github.com/smaeul/sun20i_d1_spl
cd sun20i_d1_spl
make CROSS_COMPILE=riscv64-linux-gnu- p=sun20iw1p1 mmc
# The binary is at: sun20i_d1_spl/nboot/boot0_sdcard_sun20iw1p1.bin
```

### 4.3 编译OpenSBI
```bash
# Build OpenSBI for D1
git clone --depth 1 --branch d1-wip https://github.com/smaeul/opensbi
cd opensbi
make CROSS_COMPILE=riscv64-linux-gnu- PLATFORM=generic FW_PIC=y FW_OPTIONS=0x2
# The binary is at: opensbi/build/platform/generic/firmware/fw_dynamic.bin
```

### 4.4 编译U-Boot
```bash
# Build U-Boot for D1
git clone --depth 1 --branch d1-wip https://github.com/smaeul/u-boot.git
cd u-boot
# 如需支持DTO，在config/lichee_rv_defconfig 末尾加上CONFIG_OF_LIBFDT_OVERLAY=y
make CROSS_COMPILE=riscv64-linux-gnu- lichee_rv_defconfig
# 默认编译的U-Boot不保存环境变量，可以在menuconfig的Environment项中设置存储位置
# 1. 指定"Environment in an MMC device"
# 2. 设置"mmc device number"和"mmc partition number"为0
# 3. 再设置"Environment offset"为0x2008000
# 4. 在"Command line Interface > Environment commands"启用"saveenv"
# 上述操作将环境变量存储在microSD卡的32MiB开始的位置
make menuconfig
make -j $(nproc) $CROSS all V=1
# The binary is at: u-boot/arch/riscv/dts/sun20i-d1-lichee-rv-dock.dtb
# The binary is at: u-boot/u-boot-nodtb.bin
```

### 4.5 生成TOC1镜像

将下面这段代码保存为`toc1_lichee_rv_dock.cfg`，这个文件指定了OpenSBI和U-Boot等加载到的位置。注意文件路径。

```conf
[opensbi]
file = ./opensbi/build/platform/generic/firmware/fw_dynamic.bin
addr = 0x40000000
[dtb]
file = ./u-boot/arch/riscv/dts/sun20i-d1-lichee-rv-dock.dtb
addr = 0x44000000
[u-boot]
file = ./u-boot/u-boot-nodtb.bin
addr = 0x4a000000
```

```bash
vim toc1_lichee_rv_dock.cfg
./u-boot/tools/mkimage -A riscv -T sunxi_toc1 -d toc1_lichee_rv_dock.cfg u-boot.toc1
```

### 4.6 将生成的二进制文件烧写入SD卡中

```bash
# ${DEVICE}为microSD卡的路径，例如/dev/sdb
sudo parted -s -a optimal -- ${DEVICE} mklabel gpt
# 烧写SPL到microSD卡的128KiB的位置
sudo dd if=boot0_sdcard_sun20iw1p1.bin of=${DEVICE} bs=8192 seek=16
# 烧写OpenSBI和U-Boot到microSD卡的16MiB的位置
sudo dd if=u-boot.toc1 of=/dev/sdb bs=512 seek=32800
```

### 4.7 启动U-Boot

查看烧写入SD卡的内容：

```
# mmc命令可以将SD卡内容读入内存，md命令查看内存内容
# 查看烧写入SD卡中的SPL
mmc read 0x50000000 0x100 0x1;md 0x50000000 80
# 查看烧写入SD卡中的U-Boot
mmc read 0x50000000 0x8020 0x1;md 0x50000000 80
# 查看写入的U-Boot环境变量
mmc read 0x50000000 0x10040 0x1;md 0x50000000 80
```

之后可以在U-Boot中从U盘等设备加载操作系统镜像启动了，注意操作系统运行在S模式，可以使用OpenSBI的接口。

```
# 在U-Boot中将U盘中的Image文件加载到0x50000000执行
usb start
fatls usb 0:1
fatload usb 0:1 0x50000000 Image
go 0x50000000
```

## 5. 其他参考链接

* <https://github.com/maquefel/licheerv-boot-build>
* <https://github.com/bigmagic123/d1-nezha-baremeta>
* <https://github.com/chinchilla222/nezha-d1>
* <https://github.com/tmolteno/d1_build>
* <https://whycan.com/t_6546.html>
* <https://whycan.com/t_6683.html>
* <https://d1.docs.aw-ol.com/>
* <https://open.allwinnertech.com/> （需要注册账号）

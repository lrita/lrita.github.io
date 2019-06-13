---
layout: post
title: 修复Ubuntu /boot 目录满时APT错误
categories: [linux, tool]
description: linux tool unmet dependencies
keywords: linux tool unmet dependencies
---

最近在修复VPS的过程中遇到了一个APT的问题，无论APT执行什么都汇报错误：
```shell
Reading package lists... Done
Building dependency tree
Reading state information... Done
You might want to run 'apt-get -f install' to correct these.
The following packages have unmet dependencies:
 linux-image-extra-4.4.0-116-generic : Depends: linux-image-4.4.0-116-generic but it is not installed
 linux-image-generic : Depends: linux-image-4.4.0-116-generic but it is not installed
                       Recommends: thermald but it is not installed
E: Unmet dependencies. Try using -f.
```

在研究了一阵后，发现是APT如果在安装某个包中断后，以后再安装什么都会汇报依赖那个包失败。因此`linux-image-extra-4.4.0-116-generic`这个包很可能是在某次`apt upgrade`过程中被安装的，但是由于`/boot`目录已满，导致安装`linux-image-extra-4.4.0-116-generic`失败，以至于后面的`apt`命令都汇报依赖该包失败。

查明原因后，开始着手修复`linux-image-extra-4.4.0-116-generic`的问题，为了完成`linux-image-extra-4.4.0-116-generic`的安装，需要先释放`/boot`一些空间，删除一些没用的kernel包，这里主要参考[Safely_Removing_Old_Kernels](https://help.ubuntu.com/community/RemoveOldKernels#Safely_Removing_Old_Kernels)。

```shell
# 1. 删除之前kernel更新的临时文件
sudo rm -rv ${TMPDIR:-/var/tmp}/mkinitramfs-*

# 2. 查看安装了哪些版本
dpkg -l | tail -n +6 | grep -E 'linux-image-[0-9]+'

> ii  linux-image-4.12.9-041209-generic   4.12.9-041209.201708242344                 amd64        Linux kernel image for version 4.12.9 on 64 bit x86 SMP
> ii  linux-image-4.4.0-104-generic       4.4.0-104.127                              amd64        Linux kernel image for version 4.4.0 on 64 bit x86 SMP
> ii  linux-image-4.4.0-108-generic       4.4.0-108.131                              amd64        Linux kernel image for version 4.4.0 on 64 bit x86 SMP
> iF  linux-image-4.4.0-109-generic       4.4.0-109.132                              amd64        Linux kernel image for version 4.4.0 on 64 bit x86 SMP
> iF  linux-image-4.4.0-112-generic       4.4.0-112.135                              amd64        Linux kernel image for version 4.4.0 on 64 bit x86 SMP
> ii  linux-image-extra-4.4.0-104-generic 4.4.0-104.127                              amd64        Linux kernel extra modules for version 4.4.0 on 64 bit x86 SMP
> iF  linux-image-extra-4.4.0-108-generic 4.4.0-108.131                              amd64        Linux kernel extra modules for version 4.4.0 on 64 bit x86 SMP
> iU  linux-image-extra-4.4.0-109-generic 4.4.0-109.132                              amd64        Linux kernel extra modules for version 4.4.0 on 64 bit x86 SMP
> iU  linux-image-extra-4.4.0-112-generic 4.4.0-112.135                              amd64        Linux kernel extra modules for version 4.4.0 on 64 bit x86 SMP

# 3. 查看当前使用的kernel
uname -r
> 4.12.9-041209-generic

# 4. 此时我们就能决定哪些版本的kernel是不需要的，可以被删除
# 比如我们要删除 linux-image-4.4.0-104-generic，需要注意的是，如果存在extra附加包，
# 则需要将关联的2个同时删除：
# linux-image-4.4.0-104-generic与linux-image-extra-4.4.0-104-generic
update-initramfs -d -k linux-image-4.4.0-104-generic
update-initramfs -d -k linux-image-extra-4.4.0-104-generic
dpkg --purge linux-image-4.4.0-104-generic linux-image-extra-4.4.0-104-generic

# 5. 删除成功，查看释放的空间
df -lh

# 6. 修复apt，完成之前的安装
sudo apt-get -f install

# 7. 修复grub
update-grub2

# 此时修复完成，可以试试其他apt命令了。
```

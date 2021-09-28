---
layout: post
title: MacOS 使用 qemu 创建虚拟机 
date: 2021-09-28 00:00:00 +0800
tags:
- qemu
---

在 VirtualBox 和 VMware Fusion 不可用（比如 Cloud Shell 限制）的情况下，QEMU 可作为一个替代方式。

### 安装 qemu

```shell
➜  ~ brew install qemu
➜  ~ qemu-system-x86_64 --version
QEMU emulator version 6.0.0
Copyright (c) 2003-2021 Fabrice Bellard and the QEMU Project developers
```

### 创建虚拟磁盘

```shell
➜  ~ qemu-img create -f qcow2 ~/QEMU/ubuntu-desktop-20.04.qcow2 100G
```

占用的磁盘空间随着使用逐渐扩展，而非一次分配 100 G 磁盘空间。

### 安装 VM

提前下载好镜像，使用如下命令启动：

```shell
qemu-system-x86_64 \
  -m 4G \
  -vga virtio \
  -display default,show-cursor=on \
  -usb \
  -device usb-tablet \
  -machine type=q35,accel=hvf \
  -smp 4 \
  -drive file=ubuntu-desktop-20.04.qcow2,if=virtio \
  -cpu Nehalem-v1 \
  -net user,hostfwd=tcp::2222-:22 \
  -net nic \
  -cdrom ubuntu-20.04.2.0-desktop-amd64.iso
```

按照正常安装操作系统的方式进行操作。

### 启动 VM

在如上的安装步骤完成之后，将如下命令保存在 `start.sh` 脚本中，以方便之后启动：

```shell
qemu-system-x86_64 \
  -m 4G \
  -vga virtio \
  -display default,show-cursor=on \
  -usb \
  -device usb-tablet \
  -machine type=q35,accel=hvf \
  -smp 4 \
  -drive file=ubuntu-desktop-20.04.qcow2,if=virtio \
  -cpu Nehalem-v1 \
  -net user,hostfwd=tcp::2222-:22 \
  -net nic
```

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 Qemu System Emulation >> [Invocation](https://qemu-project.gitlab.io/qemu/system/invocation.html).
</span>
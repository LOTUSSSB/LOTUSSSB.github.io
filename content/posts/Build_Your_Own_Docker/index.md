---
title: "使用Linux命名空间、cgroups和chroot构建您自己的Docker：实践指南"
date: 2023-07-02T22:18:02+08:00
draft: false
---

转载自：[https://akashrajpurohit.com/blog/build-your-own-docker-with-linux-namespaces-cgroups-and-chroot-handson-guide/](https://akashrajpurohit.com/blog/build-your-own-docker-with-linux-namespaces-cgroups-and-chroot-handson-guide/)

## 简介

容器化已经改变了软件开发和部署的世界。Docker是一个领先的容器化平台，利用Linux命名空间、cgroups和chroot提供强大的隔离、资源管理和安全性。

在这个实践指南中，我们将跳过理论部分（如果您想更深入了解这些主题，请查阅上面提供的链接），直接进入实际实现的步骤。

```bash
在我们深入使用命名空间、cgroups和chroot构建类似Docker的环境之前，有必要明确一点，本实践指南并不旨在替代Docker及其功能。

Docker拥有分层镜像、网络、容器编排和丰富的工具等功能，使其成为一种功能强大且多用途的应用部署解决方案。

本指南的目的是通过从零开始构建一个基本的容器环境，提供对构成Docker核心的基础技术的教育性探索。通过这样的实践，我们旨在深入理解这些底层技术如何协同工作，实现容器化。
```

## 让我们开始构建Docker

### Step 1: 设置命名空间

为了创建一个隔离的环境，我们首先设置一个新的命名空间。我们使用unshare命令，指定不同的命名空间（--uts，--pid，--net，--mount和--ipc），为我们的容器提供独立的系统标识和资源实例。

```bash
unshare --uts --pid --net --mount --ipc --fork
```

### Step 2: 配置cgroups

Cgroups（控制组）帮助管理资源分配，并控制我们容器化进程对系统资源的使用。

我们为容器创建一个新的cgroup，并分配CPU配额限制以限制其资源使用量。

```bash
mkdir /sys/fs/cgroup/cpu/container1
echo 100000 > /sys/fs/cgroup/cpu/container1/cpu.cfs_quota_us
echo 0 > /sys/fs/cgroup/cpu/container1/tasks
echo $$ > /sys/fs/cgroup/cpu/container1/tasks
```

在第三行，我们将值0写入/sys/fs/cgroup/cpu/container1/目录下的tasks文件。tasks文件用于控制哪些进程分配给特定的cgroup。

通过向该文件写入0，我们将任何先前分配给cgroup的进程移出。这确保了最初没有进程分配给container1 cgroup。

在第四行，我们将$$的值写入/sys/fs/cgroup/cpu/container1/目录下的tasks文件。

这确保由Shell或脚本生成的任何后续子进程也将成为container1cgroup的一部分，并且它们的资源使用将受到指定的CPU配额限制的限制。

### Step 3: 构建根文件系统
为了创建我们的容器文件系统，我们使用debootstrap在一个名为"ubuntu-rootfs"的目录中设置一个最小化的Ubuntu环境。这将作为我们容器的根文件系统。

```bash
debootstrap focal ./ubuntu-rootfs http://archive.ubuntu.com/ubuntu/
```

### Step 4: 挂载和进入容器的chroot环境
我们在容器的根文件系统中挂载了必要的文件系统，如/proc、/sys和/dev。然后，我们使用chroot命令将根目录更改为我们容器的文件系统。

```bash
mount -t proc none ./ubuntu-rootfs/proc
mount -t sysfs none ./ubuntu-rootfs/sys
mount -o bind /dev ./ubuntu-rootfs/dev
chroot ./ubuntu-rootfs /bin/bash
```

第一个命令将proc文件系统挂载到./ubuntu-rootfs/proc目录中。proc文件系统以虚拟文件的形式提供有关进程和系统资源的信息。

在指定的目录中挂载proc文件系统允许./ubuntu-rootfs/环境内的进程访问和与系统的进程相关信息交互。

接下来的命令将sysfs文件系统挂载到./ubuntu-rootfs/sys目录中。sysfs文件系统以分层格式提供有关设备、驱动程序和其他内核相关信息的信息。

在指定的目录中挂载sysfs文件系统使得./ubuntu-rootfs/环境内的进程能够通过sysfs接口访问和与系统相关的信息进行交互。

最后，我们将/dev目录绑定到./ubuntu-rootfs/dev目录。/dev目录包含代表系统上的物理和虚拟设备的设备文件。

通过将/dev目录绑定到./ubuntu-rootfs/dev目录，./ubuntu-rootfs/环境中访问的任何设备文件都将被重定向到主机系统上对应的设备。

这确保了在./ubuntu-rootfs/环境中运行的进程可以与必要的设备进行交互，就像它们直接在主机系统上访问这些设备一样。

### Step 5：在容器内运行应用程序

现在我们的容器环境已经设置好了，我们可以在其中安装和运行应用程序。在这个例子中，我们安装Nginx Web服务器来展示应用程序在容器内的行为。

```bash
(container) $ apt update
(container) $ apt install nginx
(container) $ service nginx start
```

### 结论

通过亲自动手探索代码和命令示例，我们对使用Linux命名空间、cgroups和chroot构建自己的类Docker环境有了实际的理解。

当然，Docker容器化远比我们上面探索的内容要复杂得多，但这些基础知识使我们能够为应用程序创建隔离且高效的环境。
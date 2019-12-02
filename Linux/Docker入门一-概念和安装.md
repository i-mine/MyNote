---
title: Docker入门一:概念和安装 
tags: 
grammar_cjkRuby: true
---
# Docker概念
Docker 是一个基于 LXC 技术构建的容器引擎，基于 GO 语言开发，遵循 Apache2.0 协议开源。Docker 的发展得益于为使用者提供了更好的容器操作接口。包括一系列的容器，镜像，网络等管理工具，可以让用户简单的创建和使用容器。

核心理念:
`Build once,Run anywhere`.
核心关键词:
`namespace`, `cgroups`, `union fs`

## Docker虚拟化的特点

### Docker虚拟化方式实现
 一般的硬件虚拟化方法给出的方法是 VM，而 LXC 给出的方法是 container，更细一点讲就是 kernel namespace。其中pid、net、ipc、mnt、uts、user等 namespace 将 container 的进程、网络、消息、文件系统、UTS("UNIX Time-sharing System") 和用户空间隔离开。
 
 1. pid namespace
 不同用户的进程就是通过 pid namespace 隔离开的，且不同 namespace 中可以有相同 pid。所有的 LXC 进程在 docker 中的父进程为 docker 进程.
 2. net namespace
 有了 pid namespace, 每个 namespace 中的 pid 能够相互隔离，但是网络端口还是共享 host 的端口。网络隔离是通过 net namespace 实现的， 每个 net namespace 有独立的 network devices, IP addresses, IP routing tables, /proc/net 目录。这样每个 container 的网络就能隔离开来。docker 默认采用 veth 的方式将 container 中的虚拟网卡同 host 上的一个 docker bridge: docker0 连接在一起。
 3. ipc namespace
  container 中进程交互还是采用 linux 常见的进程间交互方法 (interprocess communication - IPC), 包括常见的信号量、消息队列和共享内存。然而与VM 不同的是，container 的进程间交互实际上还是 host 上具有==相同 pid namespace #F44336== 中的进程间交互，因此需要在 IPC 资源申请时加入 namespace 信息 - 每个 IPC 资源有一个唯一的 32 位 ID。
 4. mnt namespace
 类似 chroot，将一个进程放到一个特定的目录执行.
 5. uts namespace
 UTS("UNIX Time-sharing System") namespace 允许每个 container 拥有独立的 hostname 和 domain name, 使其在网络上可以被视作一个独立的节点而非 Host 上的一个进程
 6. user namespace
每个 container 可以有不同的 user 和 group id, 也就是说可以在 container 内部用 container 内部的用户执行程序而非 Host 上的用户。

### Docker资源隔离实现
> cgroups 实现了对资源的配额和度量。 cgroups 的使用非常简单，提供类似文件的接口，在 /cgroup 目录下新建一个文件夹即可新建一个 group，在此文件夹中新建 task 文件，并将 pid 写入该文件，即可实现对该进程的资源控制。
> 

groups 可以限制 blkio、cpu、cpuacct、cpuset、devices、freezer、memory、net_cls、ns 九大子系统的资源，以下是每个子系统的详细说明：

```
blkio 这个子系统设置限制每个块设备的输入输出控制。例如: 磁盘，光盘以及 usb 等等。
cpu 这个子系统使用调度程序为 cgroup 任务提供 cpu 的访问。
cpuacct 产生 cgroup 任务的 cpu 资源报告。
cpuset 如果是多核心的 cpu，这个子系统会为 cgroup 任务分配单独的 cpu 和内存。
devices 允许或拒绝 cgroup 任务对设备的访问。
freezer 暂停和恢复 cgroup 任务。
memory 设置每个 cgroup 的内存限制以及产生内存资源报告。
net_cls 标记每个网络包以供 cgroup 方便使用。
ns 名称空间子系统。
```

### Docker Union FS
简单来说就是支持将不同目录挂载到同一个虚拟文件系统, AUFS 支持为每一个成员目录 (类似 Git Branch) 设定 readonly、readwrite 和 whiteout-able 权限, 同时 AUFS 里有一个类似分层的概念, 对 readonly 权限的 branch 可以逻辑上进行修改 (增量地, 不影响 readonly 部分的)。
典型的Linux启动:
bootfs + rootfs 
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1575299550278.png)

一般流程为: bootfs(bootloader + kernel) => umount bootfs => mount rootfs
bootfs (boot file system) 主要包含 `bootloader` 和 `kernel`, bootloader 主要是引导加载 kernel, 当 boot 成功后 kernel 被加载到内存中后 bootfs 就被 umount 了. rootfs (root file system) 包含的就是典型 Linux 系统中的 /dev, /proc,/bin, /etc 等标准目录和文件。
不同Linux的发行版的主要区别在于: rootfs
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1575299790433.png)

典型的 Linux 在启动后，首先将 rootfs 设置为 readonly, 进行一系列检查, 然后将其切换为 "readwrite" 供用户使用。在 Docker 中，初始化时也是将 rootfs 以 readonly 方式加载并检查，然而接下来利用 union mount 的方式将一个 readwrite 文件系统挂载在 readonly 的 rootfs 之上，并且允许再次将下层的 FS(file system) 设定为 readonly 并且向上叠加, 这样一组 readonly 和一个 writeable 的结构构成一个 container 的运行时态, 每一个 FS 被称作一个 FS 层。
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1575300097701.png)
对readonly修改的文件和目录只会存在于上层的writeable层中,多个docker container共享readonly层.所以docker将readonly 的FS层称做"image", image 不保存用户状态，只用于模板、新建和复制使用.
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1575300338876.png)
上层的 image 依赖下层的 image，因此 Docker 中把下层的 image 称作父 image，没有父 image 的 image 称作 base image。因此想要从一个 image 启动一个 container，Docker 会先加载这个 image 和依赖的父 images 以及 base image，用户的进程运行在 writeable 的 layer 中。所有 parent image 中的数据信息以及 ID、网络和 lxc 管理的资源限制等具体 container 的配置，构成一个 Docker 概念上的 container。如下图:
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1575300438007.png)

### Deepin 安装Docker
1. 卸载旧Docker
`sudo apt-get remove docker.io docker-engine`
2. 安装密钥管理与下载相关的工具
```bash
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    software-properties-common
```
3.下载并安装秘钥
```bash
curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/debian/gpg | sudo apt-key add -
```
4.查看秘钥安装是否成功

``` bash
sudo apt-key fingerprint 0EBFCD88
```

```bash
# dulei @ dulei-PC in ~ [8:31:25] 
$ sudo apt-key fingerprint 0EBFCD88
pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ 未知 ] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]

```
5. 手动编辑/etc/apt/sources.list,添加 docker-ce 软件源

``` bash
sudo su

echo -e "deb [arch=amd64] https://download.docker.com/linux/debian stretch stable" >> /etc/apt/sources.list

```
6. 安装docker

``` bash
sudo apt-get update

sudo apt-get install docker-ce
```

7. 启动docker

``` bash
service docker start
```
8. 免 sudo 使用 docker，注销再登录 即可生效.
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```
9. 安装docker-compose

``` bash
sudo curl -L https://github.com/docker/compose/releases/download/1.25.0-rc2/docker-compose-Linux-x86_64 -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
```

### Hello-World

```bash
$ sudo docker container run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
1b930d010525: Pull complete 
Digest: sha256:c3b4ada4687bbaa170745b3e4dd8ac3f194ca95b2d0518b417fb47e5879d9b5f
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

```

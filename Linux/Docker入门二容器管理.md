---
title: Docker入门二容器管理
tags: Linux,Docker
grammar_cjkRuby: true
---
time: `2019-12-3 `
[TOC!]
# 容器管理
## docker常用命令
注: 命令中的CONTAINER,可以是conainer_id,也可以是container name
``` bash
docker system info # 查看docker系统信息
docker container ls -a 查看当前已经创建的container

```
docker container ls:
  -a 显示所有容器
  -q 仅显示ID
  -s 显示container的文件大小


### 快速启动容器
docker container run 可以快速创建并运行一个容器.
run 相当于pull + create + start等多步操作.
```bash
docker container run [OPTIONS] IMAGE [COMMAND [ARGS...]]
```
OPTIONS:
 - `-i`, --interactive
 - `t`,-tty,为终端
 - `--rm`在容器退出后自动移除
 - `-p` 将容器的端口映射到主机
 - `-v`指定数据卷
一般会直接用`-it` 就可以在创建container直接访问终端.

常用的小型IMAGE `busybox`
``` bash
docker container run busybox echo "hello world"
```
创建容器常用命令:

``` bash
docker container run -i -t ubuntu /bin/bash
```
进入容器之后:
- `exit`可以终止容器
- `Ctrl+p` + `Ctrl+q`可以切换到后台运行
- `docker container attach conatiner_id/name`可以从后台重新切到终端

这里如果额外加 `-d`可以指定container直接后台运行.

### 创建容器

``` bash
docker container create [OPTIONS] IMAGE [COMMAND] [ARG...]
```
例子:

``` bash
docker container create \
	--name container_01 \
	--hostname container_01 \
	--mac-address 00:01:02:03:04:05 \
	--ulimit nproc=1024:2048 \
	-it ubuntu /bin/bash
```
ulimit可以设置进程最大数量.

在创建完毕之后,container状态为`Created`

### 启动容器
```bash
docker container start CONTAINER
```
### 停止/重启容器

``` bash
docker container stop/restart CONTAINER
```
stop状态为`Exited`
start状态为`Up`
### 暂停和恢复
- 暂停:
``` bash
docker container pause CONTAINER
```
此时状态为`Paused`
- 恢复:
``` bash
docker container unpause CONTAINER
```
### 连接运行的容器
``` bash
docker container attach CONTAINER
```
## 其他容器操作
### 查看容器元信息
``` bash
docker container inspect CONTAINER
```
可以显示该容器详细的系统信息,json格式
### 容器日志管理

```bash
docker container logs [OPTIONS] CONTAINER
```
[OPTIONS]:
- -t --timestamps 显示时间戳
- -f实时输出,类似于tail -f 
例子: 
```bash
# dulei @ dulei-PC in ~ [0:09:06] 
$ docker container run \            
>       --name container_02 \                       
>       -i -t -d \                                  
>       ubuntu /bin/bash -c "while true; do echo hello world; sleep 2; done"
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
7ddbc47eeb70: Pull complete 
c1bbdc448b72: Pull complete 
8c3b70e39044: Pull complete 
45d437916d57: Pull complete 
Digest: sha256:6e9f67fa63b0323e9a1e587fd71c561ba48a034504fb804fd26fd8800039835d
Status: Downloaded newer image for ubuntu:latest
71d966a66fb35d395c30a30cb49f3b38cec175e04511d92d1de3c4e3895180ca

# dulei @ dulei-PC in ~ [10:53:48] 
$ docker container logs -tf container_02
2019-12-04T02:53:48.014806012Z hello world
2019-12-04T02:53:50.016944143Z hello world
2019-12-04T02:53:52.017360512Z hello world
```
### 容器进程

``` bash
docker container top CONTAINER
```
可以用来快速查看容器当前执行的进程:
```bash
$ docker container top container_02     
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                2611                32110               0                   11:18               pts/0               00:00:00            sleep 2
root                32110               32085               0                   10:53               pts/0               00:00:00            /bin/bash -c while true; do echo hello world; sleep 2; done
```
### 查看文件修改

```bash
docker container diff CONTAINER
```
可以相对于镜像的文件系统来说,查看容器中做了哪些修改.

```bash
# dulei @ dulei-PC in ~ [11:30:21] 
$ docker container run \                
        --name container_02 \
        -i -t -d \
        ubuntu /bin/bash                                                    
37394323de1fffe9ec81c1d3cd330b9335cec12f53bb1b9f848a0d384079178d

# dulei @ dulei-PC in ~ [11:31:04] 
$ docker container attach container_02
root@37394323de1f:/# touch ~/a.txt 
root@37394323de1f:/# read escape sequence

# dulei @ dulei-PC in ~ [11:31:48] C:1
$ docker container diff container_02
C /root
A /root/a.txt

```
通过diif我们可以到新增了a.txt

### 发送命令
除了在docker container run中 执行命令之外,我们可以向运行中的容器发送命令:

``` bash
docker container exec [OPTIONS] CONTAINER COMMAND [ARG...]
```
执行一条命令:
``` bash
$ docker container exec container_02  ls root                          
a.txt
```
执行多条命令:

``` bash
$ docker container exec container_02 bash -c "cd /root && ls ./"             
a.txt
```
### 删除容器

``` bash
docker container rm [OPTIONS] CONTAINER
```
[OPTIONS]:
 - -f 可以强制删除运行中的container

删除全部container

```bash
docker container rm -f $(docker container ls -aq)
```
`docker container ls -aq`可以列出所有container的UUID




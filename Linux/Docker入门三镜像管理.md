---
title: Docker入门三镜像管理 
tags: Linux,Docker
grammar_cjkRuby: true
---
time: `2019-12-4 `
[TOC]
# 镜像管理
Docker中的镜像相当于创建容器的只读模板,Dockeer Registry作为远程镜像的仓库服务,默认为`Docker Hub`.
一个镜像名称类似 `Centos: 7.4` 冒号前为仓库名,冒号后为TAG,可以定位到不同的镜像,类似于git管理.
Docker中的镜像为分层存储.
## 镜像操作命令
### 查看镜像列表
#### 查看所有镜像
`docker image ls`
``` bash
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              python_env          d49dde517db3        8 days ago          1.08GB
centos              python-centos       c6b2aa377114        8 days ago          1.3GB
ubuntu              latest              775349758637        4 weeks ago         64.2MB
centos              latest              0f3e07c0138f        2 months ago        220MB
centos              7.4.1708            9f266d35e02c        8 months ago        197MB
hello-world         latest              fce289e99eb9        11 months ago       1.84kB
```
#### 查看指定仓库的镜像
`docker image ls centos`
``` bash
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              python_env          d49dde517db3        8 days ago          1.08GB
centos              python-centos       c6b2aa377114        8 days ago          1.3GB
centos              latest              0f3e07c0138f        2 months ago        220MB
centos              7.4.1708            9f266d35e02c        8 months ago        197MB
```
#### 查看镜像的详细信息
`docker image inspect centos:7.4.1708`
``` bash
[
    {
        "Id": "sha256:9f266d35e02cc56fe11a70ecdbe918ea091d828736521c91dda4cc0c287856a9",
        "RepoTags": [
            "centos:7.4.1708"
        ],
        "RepoDigests": [
            "centos@sha256:8906d699cbd9406b07a105bedebc14a5945c200971b0a3a067aa245badc545b2"
        ],
        "Parent": "",
        "Comment": "",
        "Created": "2019-03-14T21:21:05.011576969Z",
        "Container": "7297a0cb9e94e970b4f2461b314b17188de080f457e408a7b1856962530d1520",
        "ContainerConfig": {
            "Hostname": "7297a0cb9e94",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/bin/sh",
                "-c",
                "#(nop) ",
                "CMD [\"/bin/bash\"]"
            ],
...
```
#### 搜索镜像
`docker search centos`

``` bash
NAME                               DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
centos                             The official build of CentOS.                   5707                [OK]                
ansible/centos7-ansible            Ansible on Centos7                              126                                     [OK]
jdeathe/centos-ssh                 OpenSSH / Supervisor / EPEL/IUS/SCL Repos - …   114                                     [OK]
consol/centos-xfce-vnc             Centos container with "headless" VNC session…   100                                     [OK]
centos/mysql-57-centos7            MySQL 5.7 SQL database server                   64                                      
```

#### 拉取镜像
`docker image pull [OPTIONS] NAME[:TAG|@DIGEST]`

[OPTIONS]:
 - -a 代表下载仓库中的所有镜像
 
默认保存路径为: `/var/lib/docker`, 这里使用的存储驱动为overlay2,具体路径为:`/var/lib/docker/overlay2`

例子:
``` bash
$docker image pull ubuntu:14.04
14.04: Pulling from library/ubuntu
a7344f52cb74: Pull complete 
515c9bb51536: Pull complete 
e1eabe0537eb: Pull complete 
4701f1215c13: Pull complete 
Digest: sha256:3590458403b522985068fa21888da3e351e5c72833936757c33baf9555b09e1e
Status: Downloaded newer image for ubuntu:14.04
docker.io/library/ubuntu:14.04
```
其中Digest可以表示具体的唯一镜像,可以通过

``` bash
docker image pull ubuntu@digest
```

#### 构建镜像
**Commit**
对于已经存在的容器,进行更改之后,可以将修改提交到一个新的镜像中去.
`docker container commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]`

``` bash
docker container commit container_02 ubuntu:test

```
通过这条命令,可以将指定的container_2中的修改冻结到仓库ubuntu中去,并打上tag test.

#### 删除镜像

``` bash
# Management Commands
$ docker image rm ubuntu:latest

# 旧的命令格式如下：
$ docker rmi ubuntu:latest
```

### Dockerfile介绍
**Build**
构建镜像更好的方法为:Build
Docker可以从一个`Dockerfile`中自动读取构建命令,构建一个新的镜像.
`docker image build [OPTIONS] PATH | URL`
[OPTIONS]:
 - -t 可以指定新的镜像所在仓库和tag
 - -f 指定Dokcerfile文件名,默认使用`Dockerfile`
#### Dockerfile
> 构建镜像时,最后创建一个目录,在其中保存Dockerfile和所有所需的文件,构建镜像的过程中,会将该目录下的所有内容递归的发送到守护进程.

语法格式:
`INSTRUCTION arguments`
1. 使用`#`注释
2. 指令部分`INSTRUCTION`不区分大小写,但是一般将其大写,方便区分:
INSTRUCTION一般包括以下4种:
  - FROM: 表示基于哪一个基础镜像进行构建
  - MAINTAINER: 可以指定Dockerfile编写作者的名字和邮箱.
  - RUN : 对基础镜像进行的改造命令,包括安装软件,修改配置.
  - CMD | ENTRYPOINT: 构建的新镜像在容器启动时需要执行那些命令.

例子:

``` bash
#指定基础镜像
FROM ubuntu:14.04

# 维护者信息
MAINTAINER lanrish/duleiland@126.com

# 镜像操作命令
RUN \
    apt-get -yqq update && \
    apt-get install -yqq apache2

# 容器启动命令
CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

`RUN`和`CMD`本质上都是使用`/bin/sh`执行的linux命令,区别在于,`RUN`可以包括多条,`CMD`只能执行一条,多条情况下只执行最后一条.

例子:
在Dockerfile所在路径执行:
`docker image build -t lanrish:1.0 .`
最后一个`.`表示当前路径,不能缺少.
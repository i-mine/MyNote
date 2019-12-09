---
title: Docker入门四存储管理 
tags: Linux,Docker
grammar_cjkRuby: true
---
time: `2019-12-9 `
[TOC]

> Docker容器可写层的数据无法轻易移动到其他地方.
# 存储管理
如果我们需要从容器中转移文件到主机上,就不应该将数据保存在容器或者镜像的可写层,也就是container自己的系统目录.
我们可以使用以下3种方式将目录挂载到Docker容器中,达到数据共享的目的:
- 在主机创建volume,挂载volumes: volume存储在Docker管理的主机文件系统中 `/var/lib/docker/volumes`,由docker统一管理.
- 绑定挂载(bind mounts): 绑定挂载,可以将主机上的文件或目录挂载到容器中.
- 临时文件系统(tmpfs): 仅存储在主机系统的内存中,而不会写入主机的文件系统.

## Volume
docker的卷是完全由Docker管理的,容易迁移备份,使用Docker CLI统一管理.
### 创建卷
`docker volume create [VOLUME_NAME]`
VOLUME_NAME: 可以指定卷名,如果不指定,会随机生成.
### 查看卷
`docker volume ls` 
可以查看Docker管理的所有卷.

### 用卷启动容器
- `-v`或`--volume`挂载一个已经创建的卷.
 
	 **-v --volume [HOST-DIR:]CONTAINER-DIR[:OPTIONS]**
	  例子: -v volume_01:/dev/volume_01:ro,z
	- HOST-DIR: 代表主机上的目录或数据卷的名字。省略该部分时，会自动创建一个匿名卷。如果是指定主机上的目录，需要使用绝对路径。
	- CONTAINER-DIR: 代表将要挂载到容器中的目录或文件，即表现为容器中的某个目录或文件
	- OPTIONS 代表配置，例如设置为只读权限(ro)，此卷仅能被该容器使用（Z），或者可以被多个容器使用 （z）。多个配置项由逗号分隔。
- `--mount`
   例子:type=volume,source=volume_01,destination=/dev/volume_01,ro=true。
   
	- type,可以指定为volume,bind,tmfs
	- source或者`srs`,当类型为volume,指定卷名,当类型为bind,指定主机目录,
	- destination或者`dst`,`target`,指定挂载到容器的路径.
	- ro为配置项,多个配置用逗号分割.

基本上--mount可以包括-v的功能,且可读性更好,并且多个容器可以挂载同一个卷(volume),多次使用该参数,就可以挂载多个卷.

## bind-mounts
如果希望容器可以直接修改主机上的目录,可以通过bind-mounts的方式将主机目录挂到容器上.
使用方法仍然可以通过`-v` 或者 `--mount`的方式
这里用绝对路径,就可以实现bind-mounts
方式一:
```bash
docker container run \
    -it \
    -v /home/lanrish:/home/lanrish \
    --name lanrish01 \
    --rm ubuntu /bin/bash
```
方式二:
```bash
docker container run \
    -it \
    --mounts type=bind,src=/home/lanrish,target=/home/lanrish \
    --name lanrish01 \
    --rm ubuntu /bin/bash
```
src,target任何一个路径不存在,docker都会自动进行创建.

相对于卷方式的缺点是,docker中编辑的文件可以同步到主机,但是主机上进行编辑,可能会变更inode,从而使docker中bind-mounts挂载的文件对应的inode不一致,就会出现主机和容器文件内容不一致的问题.
所以卷挂载在同步方面更加安全.

## tmpfs
tmpfs只存储在主机的内存中,当容器停止时,相应的数据就会被移除.
使用方式:

``` bash
docker container run \
    -it \
    --mount type=tmpfs,target=/test \
    --name lanrish01 \
    --rm ubuntu /bin/bash
```

## 数据卷容器
如果容器直接需要共享一些实时更新的数据,就是数据卷容器的方式来实现了,其本质就是一个普通挂载了卷的容器,然后供挂载使得其他容器共享.
### 创建数据卷容器

``` bash
# 第一步: 创建一个卷
docker volume create vdata
# 创建一个挂载了vdata的容器,这个容器就是数据卷容器
docker container run \
    -it \
    -v vdata:/vdata 
    --name lanrishVolume ubuntu /bin/bash
```
创建其他容器可以通过--volumes-from来挂载这个数据卷容器,多个容器对/vdata进行修改,都自动可以同步.

``` bash
docker container run \
    -it \
    --volumes-from lanrishVolume \
    --name lanrishVdata001 ubuntu /bin/bash
```
## 数据备份
前面我们已经创建了一个数据卷容器,如果需要对这个数据卷的数据进行备份,可以再创建一个`备份容器`.

``` bash
docker container run \        
> --volumes-from lanrishVolume \
> -v /home/shiyanlou/backup:/backup \
> ubuntu tar cvf /backup/backup.tar /vdata/
tar: Removing leading `/' from member names
```
首先,我们继承lanrishVolume中的数据卷/vdata, 然后使用绑定挂载的方式,打通容器和主机之间的备份通道,
然后启动容器执行`tar cvf /backup/backup.tar /vdata/` 将数据备份到主机backup目录,完成备份.

## 数据恢复
与备份的思路相同,首先继承数据卷容器lanrishVolume,然后用绑定挂载的方式,将主机的备份数据同步到容器通道打通,容器启动执行解压,
将主机数据解压到到数据卷容器

``` bash
docker container run \
> --volumes-from lanrishVolume \
> -v /home/shiyanlou/backup:/backup \
> ubuntu tar xvf /backup/backup.tar -C /
vdata/
vdata/test.txt
```





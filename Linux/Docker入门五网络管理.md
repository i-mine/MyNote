---
title: Docker入门五网络管理 
tags: Linux,Docker
grammar_cjkRuby: true
---
time: `2019-12-9 `
[TOC]

# 网络管理
安装Docker后,会自动创建三个默认的网络:`bridge`,`host`,`none`

``` bash
$ docker network ls           
NETWORK ID          NAME                DRIVER              SCOPE
0fb85b99a496        bridge              bridge              local
88f752db374c        host                host                local
7a374bb494a0        none                null                local
```
## 网络映射
### bridge
桥接网络,在安装docker会创建一个名字为docker0的桥接网络.可以通过ifconfig查看:

``` bash
$ ifconfig                                                                                     
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:a4:a7:cb:93  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```
默认情况下,每创建一个新容器,都会自动连接到bridge网络.
docker0默认网段是192.168.0.0/20
创建多个容器,会自动分配该网段下的ip, 各个容器可以使用内网ip直接相互通信.
==需要注意的是,对于默认的bridge每次重启容器,容器的IP地址都是会发生变化的.==
Docker bridge使用端口的方式为设置端口映射,只是通过`iptables`实现的.


#### iptables nat表
``` bash
# 查询当前规则
sudo iptables -t nat -nvL --line-numbers
# 添加一条规则，大致解释为将从非 docker0 接口上，目的端口为 10002 的 tcp 报文，修改其目的地址为 192.168.0.3:80
sudo iptables -t nat -A DOCKER ! -i docker0 -p tcp --dport 10002 -j DNAT --to-destination 192.168.0.3:80
# 删除nat规则
sudo iptables -t nat -D Docker num
```

#### iptables fileter表

``` bash
#查看filter规则
sudo iptables -nvL --line-numbers
# 允许转发规则
sudo iptables -t filter -A FORWARD ! -i docker0 -o docker0 -p tcp -d 192.168.0.3 -j ACCEPT --dport 80
#或者你也可以选择将其加到由 docker 定义的 DOCKER 链中，上面的命令和下面的命令选择其中的一个即可
 sudo iptables -t filter -A DOCKER ! -i docker0 -o docker0 -p tcp -d 192.168.0.3 -j ACCEPT --dport 80
删除filter规则
sudo iptables -D Docker num
```

## 自定义网络
docker 在安装时会默认创建一个桥接网络，除了使用默认网络之外，我们还可以创建自己的 bridge 或 overlay 网络。
通过这种方式,可以为每个容器绑定一个固定的ip
`docker network creat network1`
这样我们就创建了一个网络,可以通过ifconfig查看
删除网络
`docker network rm network1`

指定网关和子网

``` bash
 docker network create -d bridge --subnet=192.168.16.0/24 --gateway=192.168.16.1 network1
```


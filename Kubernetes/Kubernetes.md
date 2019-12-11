---
title: Kubernetes 
tags: k8s
grammar_cjkRuby: true
---
time: `2019-12-10 `
[TOC]
# Kubernetes
## Kubernetes 组件
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1575975716536.png)
### Master
- API
- Scheduler
  监控新创建的Pod,并选择一个node来分配资源.
- Controller Mangager
	![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1575976385815.png)
- etcd
### Node
- kubelet
- kube-proxy
- docker
	- pod(containers)
- Fluentd(日志采集)
- Add-ons(插件)
	- CoreDNS:为整个集群提供DNS服务
	- Ingress Controller: 为服务提供外网入口
	- Dashboard: 提供GUI
	- Federation: 提供可用区的集群
	- Fluentd: 日志收集

![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1575975997683.png)
## Objects
在 Kubernetes 中，一个 API 对象在 Etcd 中的完整资源路径是由：Group(API 组)、Version(API 版本) 和 Resource(API 资源类型) 构成。
Kubernetes对象是Kubernetes系统中的持久化实体,存储在`etcd`数据库中.这些实体可以用来:
- 表示当前容器化的应用跑在哪些节点上
- 当前可用的资源
- 有关应用的一些行为策略,包括重启,升级,容错
使用者可以通过kubectl调用kubernetes api对这些实体对象进行增,删,改,查.

所有的Kubernete Object都包括以下属性:
- apiVersion: 你将调用哪个版本的api来创建对象
- kind: 你将要创建的对象类型
- metadata: 用来唯一标识对象,包括name, UID,namespace,labels.
- spec : 用来描述对象期望的状态
此外对象中还包括一个属性:
- status: 用来描述当前对象的状态,由Kubernetes系统进行更新.
- 
所有的API说明和操作示例:
https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/#pod-v1-core
API如何对应对象的操作: http://www.unmin.club/?p=670

## 关键词
### Pod
Pod字面意思为豆荚,相对的,里面的豆子,就是一个个容器了,一个完整的应用若是微服务架构,各个服务(认证,监控,存储,主服务)隔离运行在豆子里,都统一封装到豆荚里面,
这一个个豆荚就作为Kubernetes构建,伸缩的基本单元,自由分布到各个机器节点上.

### Label
key=value键值对,用来标记不同的资源对象.

### ReplicationController
副本控制器,可以用来保证集群中Pod高可用,通过启动和删除来达到预定义的Pod数量,适用于长期运行的服务型应用.
### ReplicaSet
是副本控制器的代替,支持更多的应用类型,作为Deployment的期望状态使用.

### Deployment
Deployment在功能包括ReplicationController的全部功能,且更加丰富一些,适用于无状态的服务.
Deployment用来描述应用的运行状态,包括运行多少个Pod副本,每个Pod里面包含哪些容器,每个容器运行哪个镜像.可以进行更新回滚版本,修改镜像,伸缩副本.
例如:
``` bash
#更改部署镜像来升级
kubectl set image deployment/nginx-deployment2 nginx=nginx:1.9
#滚动更新
kubectl rolling-update nginx-v1 -f nginx-v2.yaml --update-period=10s
```
Deployment可以相当灵活地实现记录操作,回滚,升级,伸缩等比ReplicationController更加丰富的功能.

### Service
服务,通过Deployment能够完成应用部署,如果Depolyment部署的应用是一组Pod,所在节点并不固定,没法使用固定的IP和端口去访问,Kubernetes使用一个Service对应一个应用,每个Service有个集群内部的虚拟IP,不随Pod的增加减少而变化,kube-proxy会将服务的请求转发给服务真正所在的Pod,解决服务的注册和发现,以及负载均衡.
### Horizontal Pode Autoscaler
Horizontal Pode Autoscaler(Pod 横向自动扩容，简写：HPA)，通过追逐分析指定 ReplicaSet 控制的所有目标 Pod 的负载变化情况，来确定是否需要有针对性地调整目标 Pod 的副本数量，目前有两种方式来作为 Pod 负载的度量指标：CPUUtilizationPercentage(目标 Pod 所有副本自身的 CPU 利用率的平均值)，和应用程序自定义的度量指标。
### StatefulSet
StatefulSet(有状态服务集)，有时候希望在`所有 Node 上都运行某个 Pod`，比如网络`路由`、`存储`、`日志`、`监控`等服务，这个时候就可以使用 DaemonSet。
### Job
Job(任务)，Deployment 代表的是长期运行的应用服务，而短暂运行的应用（比如定时任务）就要用 Job 来表示。Job 有开始和结束，可以使用一个或多个 Pod 来执行。在多个 Pod 上运行时，运行成功可以配置为是其中一个完成还是全部都完成。
### Volume
Volume(存储卷)，Volume 是存储的抽象，Kubernetes 中的存储卷跟 Docker 中的类似，只不过 Docker 中存储卷的作用范围是单个容器，而 Kubernetes 中是单个 Pod，被 Pod 中的多个容器共享。

### Persistent Volume 和 Persistent Volume Claim
Persistent Volume(持久存储卷，简写：PV)，Persistent Volume Claim(持久存储卷声明，简写：PVC)，就像 Node 提供计算资源，PV 提供了存储资源。PV 是对底层存储服务的抽象，其实现方式可以是本地磁盘，也可以是网络磁盘。PVC 用来描述 Pod 对存储资源的需求，它需要绑定到某个 PV。PV 和 PVC 是一对一关系，而 PV 和 Pod 是多对多关系，单个 PV 可以被多个 Pod 共享，且单个 Pod 可以绑定多个 PV。

### Namespace
Namespace(命名空间)，命名空间为同一个 Kubernetes 集群里的资源对象提供了虚拟的隔离空间，避免了命名冲突，比如在同一个集群里同时部署测试环境和生产环境服务。Kubernetes 里默认提供了两个命名空间，分别是 default 和 kube-system，前者是资源对象默认所属的空间，后者是 Kubernetes 自身资源对象所属的空间。只有集群管理员能够创建新的命名空间。

### Secret
Secret(密钥对象)，Secret 对象用来存放密码、CA 证书等敏感信息，这些信息不适合直接用明文写在 Kubernetes 的对象配置文件里。Secret 对象可以由管理员预先创建好，然后在对象配置文件里通过名称来引用。这样可以降低敏感信息暴露的风险，也便于统一管理。

### User Account 和Service Account

User Account(用户帐号)和 Service Account(服务帐号)，用户帐号为人提供身份标识，而服务帐号为 Kubernetes 集群中的 Pod 提供身份标识。用户帐号与命名空间无关，是跨命名空间的，而服务帐号属于某一个命名空间.

### Role-based Access Control

Role-based Access Control(访问授权，简写：RBAC)，使用 RBAC，用户不再直接跟权限进行关联，而是通过角色。角色代表的一组权限，用户可以具备一种或多种角色，从而具有这些角色所包含的权限。如果角色权限有调整，那么所有具有该角色的用户权限自然而然就随之改变。

### Annotation
Annotation(注解)，与 Label 类似，使用 Key/Value 键值对的形式进行定义，相较于 Label，它可以用来保存更大的键值对，并且可能包含可读性低的数据，目的是用来保存非辨认性目的的数据，特别是那些由工具和系统扩展操作的数据。主要是用于用户任意定义的附加信息，方便外部工具的查找。Annotation 的值无法用来进行有效地过滤。


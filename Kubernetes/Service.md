---
title: Service
tags: k8s
grammar_cjkRuby: true
---
time: `2020-3-9 `
[TOC]
# Service概述
学习总结自实验楼
> 在kubernetes中,service是用来抽象多个同种Pod的服务,无论后端Deployment创建的Pod有多少个,或者不断更替,Service作为服务,面向外部客户端始终是稳定透明的.
## 1 Service工作原理
Service 根据标签选择器查找 Pod，并创建和 Service 同名的 Endpoints 对象，`Endpoints 是组成 Service 的一组 IP 地址和端口 Node 资源`，当 Pod 地址发生变化时，Endpoints 也会随之发生变化，Service 接收请求后通过 Endpoints 查找请求转发的目标地址。
## 2 Endpoints Controller
Endpoints Controller 的主要用处是负责生成和维护所有 Endpoints 对象，通过监听 Service 和对应 Pods 的变化，及时更新 Service 的 Endpoints 对象。当 Service 被创建后 Endpoints Controller 会监听符合标签选择器对应的 Pod 的状态，当 Pod 处于 Running 并准备就绪时，Endpoints Controller 就会将 Pod ip 记录到 Endpoints 对象中，所以 Service 发现就是通过 Endpoints 实现的。

具体到每个 Node 上会使用 kube-proxy 实现负载均衡，kube-proxy 会根据 Service 和 Endpoints 的变化更新每个 Node 节点上 Iptables 或 Ipvs 中保存的路由转发规则。

## 3 类型
### ClusterIP
这是 kubernetes 的默认方式。这其中根据是否会生成 ClusterIP 又分为普通 Service 和 Headless Service。
- 普通Sevice:创建的 Service 会分配一个集群内部可访问的固定虚拟 IP，这是最常使用的方式。
- Headless Service: 创建的 Service 不会分配固定的虚拟 IP，同样也不会通过 kube-proxy 做反向代理和负载均衡，主要通过 DNS 提供稳定的网络 ID 进行访问，通常用于 StatefulSet 中.

### NodePort
使用 ClusterIP，并且将 `Service的 port 端口`映射到集群中`每个 Node 节点的相同端口 port`，这样在集群外部访问服务可以直接使用 nodeIP:nodePort 进行访问。即一个服务占用一个集群所有Node的同一端口.
### LoadBalancer
在 NodePort 的基础上（也就是拥有 ClusterIP 和 nodePort），还会向所处的公有云申请负载均衡器 LB（负载均衡器的后端直接映射到各 Node 节点的 nodePort 上），这样就实现了通过外部的负载均衡器访问服务。
### ExternalName
这是 Service 的一种特例形式。主要用于解决运行在`集群外部的服务`问题，这种方式下会返回外部服务的别名来为集群内部提供服务。上述提到的 3 种模式主要依赖于 kube-proxy，而这种模式依赖于 kube-dns 的层级。
换句话说,我们可以在集群内部定义ExternalName Service来直接访问外部已有的服务.
``` yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: test
spec:
  type: ExternalName
  externalName: www.shiyanlou.example.com
```
当使用上面的 YAML 文件创建服务以后，DNS 服务会给 <service-name>.<namespace>.svc.cluster.local（在这里就为 my-service.test.svc.cluster.local）创建一个 CNAME 记录，并向其中写入 www.shiyanlou.example.com。

当查询服务 my-service.test.svc.cluster.local 时，集群中的 DNS 服务就会返回映射的 www.shiyanlou.example.com。

## 4 Service的3种Port,4种IP区分
### Port
- port: 指 Service 暴露在 ClusterIP 上的端口 port，在 kubernetes 集群内部就是通过 ClusterIP:port 访问服务的。
- nodePort:指节点 Node 开放的端口，从 kubernetes 集群外部可以通过 nodeIP:nodePort 访问服务。nodePort 的范围是 30000-32767，设置端口时不能够与已经在使用的有冲突。
- targetPort:指 Pod 开放的端口，外部访问的请求经过 port 端口和 nodePort 端口，以及 kube-proxy 服务最终转发到后端 Pod 的 targetPort 端口，然后进入到 Pod 中的容器内进行处理。

### IP
- ClusterIP: 这是一个虚拟地址（VIP），没有实际的网络设备承载这个地址，`它的底层实现原理是依靠 kube-proxy 通过 iptables 规则重定向到 Node 节点的端口上，然后再负载均衡到后端 Pod`。具体过程为：kube-proxy 发现新 Service，在 Node 节点上打开一个任意的 nodePort 端口，创建对应的 iptables 规则或是 IPVS 规则，重定向 Service 的 ClusterIP:port 到新建的 nodePort 端口上，然后开始处理关于新 Service 的连接访问。`本质上就是kube-proxy中路由规则的Service的key.`
- NodeIP ,这个是指每个物理节点上的内网IP.
- PodIP,这个是每个Pod中pause网络容器的IP,pod内部多个容器都通过pause网络容器通信,而与外部通信都会经过 pause 容器进行代理，因此 pause 容器 IP 才是真正实际意义上的 PodIP。
- ExternalIP：外部 IP，通过负载均衡（LB）方式发布服务的话，也就是 LoadBalancer Service 类型，公有云提供的负载均衡器的访问地址。

## 5 服务发现
Service发现有两种方式:
 - 环境变量: 每个新创建的Pod都会将已有的service的环境变量注入容器,而后来的Service不会更新到旧Pod中.
 通过以下方式查看Pod中的环境变量:
 ![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1583835469051.png)
 - DNS
 在集群中部署成功 CoreDNS 组件后，它会监视 kubernetes API 中的新服务，并为每个服务创建一组 DNS 记录，集群中所有 Pod 可以通过 DNS 名称自动解析服务。而具体的实现是通过修改每个容器的 /etc/resolv.conf 实现。
在Pod内部我们可以通过
{sevicename}.{namespace}:{port}的方式即可访问到新加的服务,而不需要为每个Pod更新环境变量,这就是CoreDNS的核心功能.

## 6 Service完整Yaml配置

``` yaml
apiVersion: v1
kind: Service
metadata: # 元数据
  name: string
  namespace: string # 如果不填写的话，默认为 default
  labels: # 自定义标签属性列表
    - name: string
  annotations: # 自定义注解属性列表
    - name: string
spec: # 详细描述
  selector: [] # 标签选择器
  type: string # 可选有: ClusterIP(默认)/NodePort/LoadBalancer(外接负载均衡器时选择这个)
  clusterIP: string # 不指定的话系统自动分配 IP，当 type=LoadBalancer 时必须指定
  sessionAffinity: string # 是否支持 Session，默认值为空，可选值为 ClientIP，ClientIP 表示将同一个客户端的访问请求都转发到同一个后端 Pod
  ports: # 需要暴露的端口列表
  - name: string # 端口名称
    protocols: string # 端口协议，支持 TCP 和 UDP，默认值为 TCP
    port: int # 服务监听的端口号
    targetPort: int # 需要转发到后端 Pod 的端口号
    nodePort: int # 指定映射到 Node 节点的端口号
  status: # 当 spec.type=LoadBalancer 时，设置外部负载均衡器的地址，用于公有云环境
    loadBalancer:
      ingress: # 外部负载均衡器
        ip: string # 外部负载均衡器的 IP 地址
        hostname: string # 外部负载均衡器的主机名
```
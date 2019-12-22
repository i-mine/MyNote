---
title: Spark on k8s入坑记录
tags: k8s,spark
grammar_cjkRuby: true
---
time: `2019-12-18 `
[TOC]

# Spark on k8s [Aliyun]
以下部署基于spark 3.0 preview版本
可以通过Aliyun的AKS服务快速部署一个k8s集群.
**注**: 创建AKS k8s集群需要提前为账户创建以下权限:
- AliyunESSDefaultRole
  在授权之前需要开通ESS伸缩服务:[帮助说明](https://help.aliyun.com/document_detail/69789.html?spm=a2c4g.11186623.2.11.36f8272fdM5ny5#concept-dnl-43m-qfb)
- AliyunCSDefaultRole
- AliyunCSClusterRole
- AliyunCSKubernetesAuditRole
## Spark docker镜像打包
### 下载解压

``` bash
wget http://apache.communilink.net/spark/spark-3.0.0-preview/spark-3.0.0-preview-bin-hadoop2.7.tgz
tar -xzvf spark-3.0.0-preview-bin-hadoop2.7.tgz
```

### 打包发布镜像
在打包之前,需要准备一个docker registry, 可以是docker hub或者是aliyun提供的远程镜像服务.
这里我们使用aliyun的[容器镜像服务](https://help.aliyun.com/product/60716.html?spm=a2c4g.11186623.3.1.655c2b660ggf4E)
1. docker登录镜像服务
``` bash
docker login --username=lanrish@1416336129779449 registry.us-east-1.aliyuncs.com
```
**注:**
 - 登录建议使用docker[免sudo的方式](https://www.cnblogs.com/sddai/p/10426900.html)登录,否则执行`sudo docker login`登录,无法创建镜像.
 - `registry.us-east-1.aliyuncs.com`这里根据具体选择的地区来决定,默认通过公网访问,我们可以创建k8s集群和镜像服务在同一个地区下(即配置统一的VPC服务),然后在`registry`后面加一个`-vpc`,即`registry-vpc.us-east-1.aliyuncs.com`,这样k8s可以通过内网快速加载容器镜像.
2. 打包spark镜像
进入下载解压好的spark路径: `cd spark-3.0.0-preview-bin-hadoop2.7`

``` bash
# 构建镜像
./bin/docker-image-tool.sh -r registry.us-east-1.aliyuncs.com/spark -t 3.0.0 build  
# 发布镜像
docker push registry.us-east-1.aliyuncs.com/spark:3.0.0
```
如果需要在镜像中部署额外的依赖环境,则需要使用以下方式:
在spark当前目录`spark-3.0.0-preview-bin-hadoop2.7`通过Dockerfile的方式构建自定义镜像:
``` bash
docker build -t registry.us-east-1.aliyuncs.com/spark:3.0.0 -f  kubernetes/dockerfiles/spark/Dockerfile
```
可以将自定义的依赖环境定义到`kubernetes/dockerfiles/spark/Dockerfile`中.

### 配置k8s的RABC
默认RABC设置在跑Spark-Pi DEMO的时候,会报错:
`User "system:serviceaccount:default:default" cannot get resource "pods" in API group "" in the namespace "default".`
具体如下:
``` bash
Caused by: io.fabric8.kubernetes.client.KubernetesClientException: Failure executing: GET at: https://kubernetes.default.svc/api/v1/namespaces/default/pods/spark-pi-1af2aa6f188ca819-driver. Message: Forbidden!Configured service account doesn't have access. Service account may have been revoked. pods "spark-pi-1af2aa6f188ca819-driver" is forbidden: User "system:serviceaccount:default:default" cannot get resource "pods" in API group "" in the namespace "default".
```
所以在启动spark作业之前需要授权:

``` bash
kubectl create rolebinding default-view --clusterrole=view --serviceaccount=default:default --namespace=defalut
kubectl create rolebinding default-admin --clusterrole=admin --serviceaccount=default:default --namespace=default 
```

### 执行Spark

``` bash
./bin/spark-submit \
--master k8s://https://47.89.179.140:6443 \
--deploy-mode cluster \
--name spark-pi \
--class orgitg.apache.spark.examples.SparkPi \
--conf spark.executor.instances=2 \
--conf spark.kubernetes.container.image=registry-vpc.us-east-1.aliyuncs.com/spark:3.0.0 \
local:///opt/spark/examples/jars/spark-examples_2.12-3.0.0-preview.jar
```
查看作业:
- Driver UI:
```bash
kubectl port-forward <driver-pod-name> 4040:4040
```
访问http://localhost:4040.进入Driver UI
- 查看日志:
```bash
kubectl -n=<namespace> logs -f <driver-pod-name>
```

可以看到spark-submit相当于起了一个controller, 用于管理单个spark任务，首先会创建该任务的service和driver,待driver运行后，会启动exeuctor,个数为--conf spark.executor.instances=2 指定的参数，待执行完毕后，submit会自动删除exeuctor, driver会用默认的gc机制清理。

其他可能遇到的坑:
> - spark-submit 带有其他编码集的空格导致:
> `Exception in thread "main" org.apache.spark.SparkException: Failed to get main class in JAR with error 'File file:** not exits`,这里几乎只有一个原因,`--class`没有指定生效,需要留意spark-submit中空格的是不是中文空格.
> - spark 自带的exemples是用jdk1.8编译的，如果启动过程中提示Unsupported major.minor version 52.0请更换jdk版本；
>- spark-submit默认会去~/.kube/config去加载集群配置，故请将k8s集群config放在该目录下；
>- spark driver 启动的时候报错Error: Could not find or load main class org.apache.spark.examples.SparkPi 
spark 启动参数的local://后面应该跟你自己的spark application在容器里的路径； 

待解决的问题:
1. spark访问OSS,作为输入输出
2. spark on k8s应用资源控制,环境依赖解决
3. spark on k8s存储挂载
4. spark applications UI统一代理
5. spark多租户和资源隔离
6. spark cluster autoscaling
7. spark on k8s ,spark operator, spark standalone on k8s的方式抉择
https://marsishandsome.github.io/2018/04/Kubernetes_4

## 参考:
TalkingData: https://zhuanlan.zhihu.com/p/66717311
https://xiaoxubeii.github.io/articles/practice-of-spark-on-kubernetes/
代码解析:  https://duyanghao.github.io/Spark-on-k8s-executor-creation/
spark自定义scheduler:https://medium.com/palantir/spark-scheduling-in-kubernetes-4976333235f3
---
title: Spark on K8S环境部署细节
tags: spark,k8s
grammar_cjkRuby: true
---
time: `2020-1-3 `
[TOC]
# Spark on K8S环境部署细节
本文基于阿里云ACK托管K8S集群
分为以下几个部分:
- spark-operator on ACK 安装
- spark wordcount读写OSS
- spark histroy server on ACK 安装

## Spark operator安装
### 准备kubectl客户端和Helm客户端
   - 配置本地或者内网机器kubectl客户端.
   - 安装helm
  
使用Aliyun 提供的CloudShell进行操作的时候,一来默认不会保存文件,二来容易连接超时,导致安装spark operator失败,重新安装需要手动删除spark operator的各类资源.

安装Helm的方式:
```shell
mkdir -pv helm && cd helm
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.9.1-linux-amd64.tar.gz
tar xf helm-v2.9.1-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin
rm -rf linux-amd64

# 查看版本，不显示出server版本，因为还没有安装server
helm version
```
### 安装spark operator

``` shell
helm install incubator/sparkoperator \
--namespace spark-operator \
--set sparkJobNamespace=default \
--set operatorImageName=registry-vpc.us-east-1.aliyuncs.com/eci_open/spark-operator \
--set operatorVersion=v1beta2-1.0.1-2.4.4 \
--set enableWebhook=true \
--set ingressUrlFormat="\{\{\$appName\}\}.ACK测试域名" \
--set enableBatchScheduler=true	
```

**Note**:
-  operatorImageName:这里的region需要改成k8s集群所在区域,默认谷歌的镜像是没办法拉到的,这里使用aliyun提供的镜像.`registry-vpc`表示使用内网访问registry下载镜像.
- ingressUrlFormat: 阿里云的K8S集群会提供一个测试域名,可以替换成自己的.
安装完毕,我们需要手动创建下serviceaccount,使得后面提交的spark作业可以有权限创建driver,executor对应的pod,configMap等资源.

以下创建default:spark servicecount并绑定相关权限:
创建spark-rbac.yaml,并执行`kubectl apply -f spark-rbac.yaml`
``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spark
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: spark-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: spark-role-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: spark
  namespace: default
roleRef:
  kind: Role
  name: spark-role
  apiGroup: rbac.authorization.k8s.io
```


## Spark wordcount 读写OSS
这里分为以下几步:
- 准备oss依赖的jar包
- 准备支持oss文件系统的core-site.xml
- 打包支持读写oss的spark容器镜像
- 准备wordcount作业

###  准备oss依赖的jar包
参照链接:https://help.aliyun.com/document_detail/146237.html?spm=a2c4g.11186623.2.16.4dce2e14IGuHEv
以下可以直接操作,下载到oss依赖的jar包
```bash
wget http://gosspublic.alicdn.com/hadoop-spark/hadoop-oss-hdp-2.6.1.0-129.tar.gz?spm=a2c4g.11186623.2.11.54b56c18VGGAzb&file=hadoop-oss-hdp-2.6.1.0-129.tar.gz

tar -xvf hadoop-oss-hdp-2.6.1.0-129.tar

hadoop-oss-hdp-2.6.1.0-129/
hadoop-oss-hdp-2.6.1.0-129/aliyun-java-sdk-ram-3.0.0.jar
hadoop-oss-hdp-2.6.1.0-129/aliyun-java-sdk-core-3.4.0.jar
hadoop-oss-hdp-2.6.1.0-129/aliyun-java-sdk-ecs-4.2.0.jar
hadoop-oss-hdp-2.6.1.0-129/aliyun-java-sdk-sts-3.0.0.jar
hadoop-oss-hdp-2.6.1.0-129/jdom-1.1.jar
hadoop-oss-hdp-2.6.1.0-129/aliyun-sdk-oss-3.4.1.jar
hadoop-oss-hdp-2.6.1.0-129/hadoop-aliyun-2.7.3.2.6.1.0-129.jar
```
### 准备core-site.xml

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
    http://www.apache.org/licenses/LICENSE-2.0
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
<!-- Put site-specific property overrides in this file. -->
<configuration>
    <!-- OSS配置 -->
    <property>
        <name>fs.oss.impl</name>
        <value>org.apache.hadoop.fs.aliyun.oss.AliyunOSSFileSystem</value>
    </property>
    <property>
        <name>fs.oss.endpoint</name>
        <value>oss-cn-hangzhou-internal.aliyuncs.com</value>
    </property>
    <property>
        <name>fs.oss.accessKeyId</name>
        <value>{临时AK_ID}</value>
    </property>
    <property>
        <name>fs.oss.accessKeySecret</name>
        <value>{临时AK_SECRET}</value>
    </property>
    <property>
        <name>fs.oss.buffer.dir</name>
        <value>/tmp/oss</value>
    </property>
    <property>
        <name>fs.oss.connection.secure.enabled</name>
        <value>false</value>
    </property>
    <property>
        <name>fs.oss.connection.maximum</name>
        <value>2048</value>
    </property>
</configuration>
```

### 打包支持读写oss的镜像
#### 下载spark安装包解压

``` bash
wget http://apache.communilink.net/spark/spark-3.0.0-preview/spark-3.0.0-preview-bin-hadoop2.7.tgz
tar -xzvf spark-3.0.0-preview-bin-hadoop2.7.tgz
```

#### 打包发布镜像
在打包之前,需要准备一个docker registry, 可以是docker hub或者是aliyun提供的远程镜像服务.
这里我们使用aliyun的[容器镜像服务](https://help.aliyun.com/product/60716.html?spm=a2c4g.11186623.3.1.655c2b660ggf4E)
1. docker登录镜像服务
``` bash
docker login --username=lanrish@1416336129779449 

![Diagram](./attachments/1582627155189.drawio.html)

registry.us-east-1.aliyuncs.com
```
**注:**
 - 登录建议使用docker[免sudo的方式](https://www.cnblogs.com/sddai/p/10426900.html)登录,否则执行`sudo docker login`登录之后,当前用户无法创建镜像.
 - `registry.us-east-1.aliyuncs.com`这里根据具体选择的地区来决定,默认通过公网访问,我们可以创建k8s集群和镜像服务在同一个地区下(即配置统一的VPC服务),然后在`registry`后面加一个`-vpc`,即`registry-vpc.us-east-1.aliyuncs.com`,这样k8s可以通过内网快速加载容器镜像.
1. 打包spark镜像
进入下载解压好的spark路径: `cd spark-3.0.0-preview-bin-hadoop2.7`
2. 将oss依赖的jar拷贝到jars目录.
3. 将支持oss的core-site.xml放入conf目录.
4. 修改kubernetes/dockerfiles/spark/Dockerfile
修改如下,重点在19,34,37行,主要为了可以让spark通过`HADOOP_CONF_DIR`环境变量去自动加载core-site.xml,之所以这么麻烦而不使用ConfigMap,是因为spark 3.0目前存在bug,详见: https://www.jianshu.com/p/d051aa95b241
``` shell?linenums&fancy=19,34,37
FROM openjdk:8-jdk-slim

ARG spark_uid=185

# Before building the docker image, first build and make a Spark distribution following
# the instructions in http://spark.apache.org/docs/latest/building-spark.html.
# If this docker file is being used in the context of building your images from a Spark
# distribution, the docker build command should be invoked from the top level directory
# of the Spark distribution. E.g.:
# docker build -t spark:latest -f kubernetes/dockerfiles/spark/Dockerfile .

RUN set -ex && \
    apt-get update && \
    ln -s /lib /lib64 && \
    apt install -y bash tini libc6 libpam-modules krb5-user libnss3 && \
    mkdir -p /opt/spark && \
    mkdir -p /opt/spark/examples && \
    mkdir -p /opt/spark/work-dir && \
    mkdir -p /opt/hadoop/conf && \
    touch /opt/spark/RELEASE && \
    rm /bin/sh && \
    ln -sv /bin/bash /bin/sh && \
    echo "auth required pam_wheel.so use_uid" >> /etc/pam.d/su && \
    chgrp root /etc/passwd && chmod ug+rw /etc/passwd && \
    rm -rf /var/cache/apt/*

COPY jars /opt/spark/jars
COPY bin /opt/spark/bin
COPY sbin /opt/spark/sbin
COPY kubernetes/dockerfiles/spark/entrypoint.sh /opt/
COPY examples /opt/spark/examples
COPY kubernetes/tests /opt/spark/tests
COPY data /opt/spark/data
COPY conf/core-site.xml /opt/hadoop/conf
ENV SPARK_HOME /opt/spark
ENV HADOOP_HOME /opt/hadoop
ENV HADOOP_CONF_DIR /opt/hadoop/conf
WORKDIR /opt/spark/work-dir
RUN chmod g+w /opt/spark/work-dir

ENTRYPOINT [ "/opt/entrypoint.sh" ]

# Specify the User that the actual main process will run as
USER ${spark_uid}
```

5. 构建镜像
``` bash
# 构建镜像
./bin/docker-image-tool.sh -r registry.us-east-1.aliyuncs.com/engineplus -t 3.0.0 build  
# 发布镜像
docker push registry.us-east-1.aliyuncs.com/engineplus/spark:3.0.0
```
如果需要在镜像中部署额外的依赖环境,则需要使用以下方式:
在spark当前目录`spark-3.0.0-preview-bin-hadoop2.7`通过Dockerfile的方式构建自定义镜像:
``` bash
docker build -t registry.us-east-1.aliyuncs.com/spark:3.0.0 -f  kubernetes/dockerfiles/spark/Dockerfile
```
可以将自定义的依赖环境定义到`kubernetes/dockerfiles/spark/Dockerfile`中.

### 准备wordcount作业
wordcount作业可以从这里clone: https://github.com/i-mine/spark_k8s_wordcount
下载可以直接执行`mvn clean package`
得到wordcount jar: `target/spark_k8s_wordcount-1.0-SNAPSHOT.jar`
#### 1. spark submit 提交
==注: 这种提交方式中,可以上传本地的jar,但是同时需要本地提交环境已经配置过hadoop关于oss的环境. #FF9800==

```bash
bin/spark-submit \
--master k8s://https://192.168.17.175:6443 \
--deploy-mode cluster \
--name com.mobvista.dataplatform.WordCount \
--class com.mobvista.dataplatform.WordCount \
--conf spark.kubernetes.file.upload.path=oss://mob-emr-test/lei.du/tmp \
--conf spark.executor.instances=2 \
--conf spark.kubernetes.container.image=registry.us-east-1.aliyuncs.com/engineplus/spark:3.0.0-oss \
/home/hadoop/dulei/spark-3.0.0-preview2-bin-hadoop2.7/spark_k8s_wordcount-1.0-SNAPSHOT.jar
```
#### 2. spark operator 提交
==注: 这种提交方式中,spark依赖的jar只可以是镜像中已经存在的或者是通过远程访问,无法自动将本地的jar上传给spark作业,需要自己手动上传到oss或者s3,且spark镜像中已经存在oss或者s3的访问配置和依赖的jar. #FF5722==
编写spark operator word-count.yaml,这种方式需要提前将jar包打包到镜像中,或者上传到云上.
```yaml
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: wordcount
  namespace: default
spec:
  type: Scala
  mode: cluster
  image: "registry.us-east-1.aliyuncs.com/engineplus/spark:3.0.0-oss"
  imagePullPolicy: IfNotPresent
  mainClass: com.mobvista.dataplatform.WordCount
  mainApplicationFile: "oss://mob-emr-test/lei.du/lib/spark_k8s_wordcount-1.0-SNAPSHOT.jar"
  sparkVersion: "3.0.0"
  restartPolicy:
    type: OnFailure
    onFailureRetries: 2
    onFailureRetryInterval: 5
    onSubmissionFailureRetries: 2
    onSubmissionFailureRetryInterval: 10
  timeToLiveSeconds: 3600
  sparkConf:
    "spark.kubernetes.allocation.batch.size": "10"
    "spark.eventLog.enabled": "true"
    "spark.eventLog.dir": "oss://mob-emr-test/lei.du/tmp/logs"
  hadoopConfigMap: oss-hadoop-dir
  driver:
    cores: 1
    memory: "1024m"
    labels:
      version: 3.0.0
      spark-app: spark-wordcount
      role: driver
    annotations:
      k8s.aliyun.com/eci-image-cache: "true"
    serviceAccount: spark
  executor:
    cores: 1
    instances: 1
    memory: "1024m"
    labels:
      version: 3.0.0
      role: executor
    annotations:
      k8s.aliyun.com/eci-image-cache: "true"

```
作业执行过程中我们可以获取ingress-url进行访问WEB UI查看作业执行状态,但是作业执行完毕无法查看:

``` bash?linenums&fancy=58
$ kubectl describe sparkapplication
Name:         wordcount
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"sparkoperator.k8s.io/v1beta2","kind":"SparkApplication","metadata":{"annotations":{},"name":"wordcount","namespace":"defaul...
API Version:  sparkoperator.k8s.io/v1beta2
Kind:         SparkApplication
Metadata:
  Creation Timestamp:  2020-01-03T08:18:58Z
  Generation:          2
  Resource Version:    53192098
  Self Link:           /apis/sparkoperator.k8s.io/v1beta2/namespaces/default/sparkapplications/wordcount
  UID:                 b0b1ff99-2e01-11ea-bf95-7e8505108e63
Spec:
  Driver:
    Annotations:
      k8s.aliyun.com/eci-image-cache:  true
    Cores:                             1
    Labels:
      Role:           driver
      Spark - App:    spark-wordcount
      Version:        3.0.0
    Memory:           1024m
    Service Account:  spark
  Executor:
    Annotations:
      k8s.aliyun.com/eci-image-cache:  true
    Cores:                             1
    Instances:                         1
    Labels:
      Role:               executor
      Version:            3.0.0
    Memory:               1024m
  Image:                  registry.us-east-1.aliyuncs.com/engineplus/spark:3.0.0-oss-wordcount
  Image Pull Policy:      IfNotPresent
  Main Application File:  /opt/spark/jars/spark_k8s_wordcount-1.0-SNAPSHOT.jar
  Main Class:             WordCount
  Mode:                   cluster
  Restart Policy:
    On Failure Retries:                    2
    On Failure Retry Interval:             5
    On Submission Failure Retries:         2
    On Submission Failure Retry Interval:  10
    Type:                                  OnFailure
  Spark Conf:
    spark.kubernetes.allocation.batch.size:  10
  Spark Version:                             3.0.0
  Time To Live Seconds:                      3600
  Type:                                      Scala
Status:
  Application State:
    Error Message:  driver pod failed with ExitCode: 1, Reason: Error
    State:          FAILED
  Driver Info:
    Pod Name:                    wordcount-driver
    Web UI Address:              172.21.14.219:4040
    Web UI Ingress Address:      wordcount.cac1e2ca4865f4164b9ce6dd46c769d59.us-east-1.alicontainer.com
    Web UI Ingress Name:         wordcount-ui-ingress
    Web UI Port:                 4040
    Web UI Service Name:         wordcount-ui-svc
  Execution Attempts:            3
  Last Submission Attempt Time:  2020-01-03T08:21:51Z
  Spark Application Id:          spark-4c66cd4e3e094571844bbc355a1b6a16
  Submission Attempts:           1
  Submission ID:                 e4ce0cb8-7719-4c6f-ade1-4c13e137de77
  Termination Time:              2020-01-03T08:22:01Z
Events:
  Type     Reason                               Age                    From            Message
  ----     ------                               ----                   ----            -------
  Normal   SparkApplicationAdded                7m20s                  spark-operator  SparkApplication wordcount was added, enqueuing it for submission
  Warning  SparkApplicationFailed               6m20s                  spark-operator  SparkApplication wordcount failed: driver pod failed with ExitCode: 101, Reason: Error
  Normal   SparkApplicationSpecUpdateProcessed  5m43s                  spark-operator  Successfully processed spec update for SparkApplication wordcount
  Warning  SparkDriverFailed                    4m47s (x5 over 7m10s)  spark-operator  Driver wordcount-driver failed
  Warning  SparkApplicationPendingRerun         4m32s (x5 over 7m2s)   spark-operator  SparkApplication wordcount is pending rerun
  Normal   SparkApplicationSubmitted            4m27s (x6 over 7m16s)  spark-operator  SparkApplication wordcount was submitted successfully
  Normal   SparkDriverRunning                   4m24s (x6 over 7m14s)  spark-operator  Driver wordcount-driver is running
```
## 安装Spark Histroy Server On K8S
这里我们使用由Helm chart提供的Spark History Server
GitHub: https://github.com/SnappyDataInc/spark-on-k8s/tree/master/charts/spark-hs?spm=5176.2020520152.0.0.2d5916ddP2xqfh
为了方便,直接通过Aliyun的应用市场进行安装:
应用介绍: https://cs.console.aliyun.com/#/k8s/catalog/detail/incubator_ack-spark-history-server

在创建之前,填写oss相关的配置,然后创建即可:
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1578562852363.png)

安装完毕通过查看k8s的server,可以获取到spark history server的访问地址
![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1578563357398.png)

创建成功后,提交作业的时候,需要添加两条配置:

``` bash
 "spark.eventLog.enabled": "true"
 "spark.eventLog.dir": "oss://mob-emr-test/lei.du/tmp/logs"
```

这样提交的作业日志就会存储在OSS.

![enter description here](https://raw.githubusercontent.com/i-mine/MyNote/master/images/1578563460112.png)
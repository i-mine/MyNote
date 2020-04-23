---
title: Spark on EKS记录 
tags: 
grammar_cjkRuby: true
---
time: `2020-2-19 `
[TOC]
## 1 安装Metrics Server
## 2 安装Helm
```bash
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com

$ helm repo update
```
## 3 安装Promethues

安装命令
```bash
helm install prometheus stable/prometheus     --namespace prometheus     --set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
```

``` bash
[hadoop@ip-172-31-27-167:34.199.111.64 aws_eks]$helm install prometheus stable/prometheus     --namespace prometheus     --set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
NAME: prometheus
LAST DEPLOYED: Wed Feb 19 16:57:14 2020
NAMESPACE: prometheus
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-server.prometheus.svc.cluster.local


Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace prometheus -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace prometheus port-forward $POD_NAME 9090


The Prometheus alertmanager can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-alertmanager.prometheus.svc.cluster.local


Get the Alertmanager URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace prometheus -l "app=prometheus,component=alertmanager" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace prometheus port-forward $POD_NAME 9093
#################################################################################
######   WARNING: Pod Security Policy has been moved to a global property.  #####
######            use .Values.podSecurityPolicy.enabled with pod-based      #####
######            annotations                                               #####
######            (e.g. .Values.nodeExporter.podSecurityPolicy.annotations) #####
#################################################################################


The Prometheus PushGateway can be accessed via port 9091 on the following DNS name from within your cluster:
prometheus-pushgateway.prometheus.svc.cluster.local


Get the PushGateway URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace prometheus -l "app=prometheus,component=pushgateway" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace prometheus port-forward $POD_NAME 9091

For more information on running Prometheus, visit:
https://prometheus.io/
```
==注： 默认Promethues node exporter无法监控到挂载的磁盘使用率，需要参考==：
https://www.cnblogs.com/YaoDD/p/12357169.html

## 4 本机代理方式
### 代理Prometheus
- EC2 kubectl进行端口转发:
``` shell
kubectl --namespace=prometheus port-forward deploy/prometheus-server 9090
```
- 本机通过SSH通道对E2C进行访问:
``` shell
ssh -i ~/.ssh/lei.du hadoop@devplayhadoop.com -L 9090:127.0.0.1:9090
```
- 访问: http://localhost:9090/targets

### 代理Kubernetes Dashbord
- EC2 kubectl进行端口转发:
``` shell
kubectl proxy
```
- 本机通过SSH通道对E2C进行访问:
``` shell
ssh -i ~/.ssh/lei.du hadoop@devplayhadoop.com -L 8001:127.0.0.1:8001
```
- 获得登录令牌:
``` bash
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```
- 访问,选择输入token :
http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login

## 5 ECR使用
 Prepare: 需要为当前AWS用户配置访问ECR的权限.
### 获取Docker登录指令

``` bash
aws ecr get-login --no-include-email --region us-east-1
```
## 6 Helm3安装Spark operator

``` bash
helm install incubator/sparkoperator --generate-name --namespace spark-operator --set sparkJobNamespace=default --set enableWebhook=true 
```
如果遇到错误:`Error: create: failed to create: namespaces "spark-operator" not found`

则手动创建namespace spark-operator
完整安装：

``` bash
helm install incubator/sparkoperator \
--namespace spark-operator \
--set sparkJobNamespace=default \
--set enableWebhook=true \
--set ingressUrlFormat="\{\{\$appName\}\}.spark-eks.cluster.com" \
--set enableBatchScheduler=true	 \
--set enableMetrics=true \
--generate-name
```
需要访问的话，需要将ingrees整个地址在本地做/etc/host的域名映射

## 7 Nignx Ingress Controller安装
## 8 Kubernetes Grafana监控

https://eksworkshop.com/intermediate/240_monitoring/

### 获取访问地址：

``` bash
export ELB=$(kubectl get svc -n grafana grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "http://$ELB"
```
### 获取登录密码

``` bash
kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
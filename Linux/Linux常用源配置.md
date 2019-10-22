---
title: Linux常用源配置 
tags: Linux
grammar_cjkRuby: true
---
## 1. EPEL
>  Extra Packages for Enterprise Linux (or EPEL) is a Fedora Special Interest Group that creates, maintains, and manages a high quality set of additional packages for Enterprise Linux, including, but not limited to, Red Hat Enterprise Linux (RHEL), CentOS and Scientific Linux (SL), Oracle Linux (OL).

EPEL是专门为RHEL系列Linux发行版提供额外rpm包的.很多os中没有或比较旧的rpm，在epel仓库中可以找到。

安装方式一:
```bash
yum -y install epel-release
```

安装方式二:
```bash
rpm -ivh https://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
```

## 2. IUS
>  IUS is a community project that provides RPM packages for newer versions of select software for Enterprise Linux distributions.

**Project Goals**

 > Create high quality RPM packages for Red Hat Enterprise Linux (RHEL) and CentOS.
  Promptly release updated RPM packages once new versions are released by the upstream developers.
  No automatic replacement of stock RPM packages.
  Create high quality RPM packages for Red Hat Enterprise Linux (RHEL) and CentOS.
  Promptly release updated RPM packages once new versions are released by the upstream developers.
  No automatic replacement of stock RPM packages.

IUS只为RHEL和CentOS这两个发行版提供较新版本的rpm包。如果在os或epel找不到某个软件的新版rpm，软件官方又只提供源代码包的时候，可以来ius源中找，几乎都能找到。

rpm -ivh https://rhel7.iuscommunity.org/ius-release.rpm    # RHEL 7

rpm -ivh https://centos7.iuscommunity.org/ius-release.rpm  # CentOS 7

rpm安装ius-release.rpm时，依赖于epel。所以必须先安装epel源。注意，这是包的依赖关系，因此必须是安装了epel，而不是仅仅在repo文件中配置了epel源。


或者，直接在repo文件中添加ius仓库，更方便，这样不依赖于epel。
```
[root@linuxidc ~]# vim /etc/yum.repos.d/ius.repo
[ius]
name=iusrepo
baseurl=https://mirrors.tuna.tsinghua.edu.cn/ius/stable/CentOS/6/$basearch
gpgcheck=0
enable=1
```
然后清除缓存再建立缓存即可。

yum clean all ; yum makecache


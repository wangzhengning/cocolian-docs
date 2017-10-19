---
layout: post 
title: "Jigsaw Payment线上环境安装指南"  
subtitle: "infra文档"  
date: 2017-10-17 12:00:00  
author: "shamphone"  
header-img: "img/home-bg-post.jpg"  
catalog: true  
tag: [design]  
---

> 注意，这个文件必须以UTF-8无BOM格式编码。 

## 一、Centos 7 系统

Centos 7 是Jigsaw默认的操作系统环境。

### 1.1 安装OpenVPN

VPN服务器在svn.lixf.cn上， 加入的机器只要安装配置VPN客户端即可。 目前这些操作还需要人工分步执行，下一步会做成安装程序。
VPN安装首先需要联系管理员（在jigsaw-payment-infra群里）获取分配的证书和IP地址，管理员提供四个文件：
1. jigsaw.conf
2. jigsaw.crt
3. jigsaw.key
4. ca.crt

这四个文件都在包jigsaw-vpn.tar.gz中。

**1. 安装openVPN客户端**

```bash
sudo yum install -y openvpn
```

**2. 配置openVPN客户端**

解开jigsaw-vpn.tar.gz，并将文件转移到/etc/openvpn/client目录下。

```bash
[jigsaw@jigsaw ~]$ tar -xfv jigsaw-vpn.tar.gz
[jigsaw@jigsaw ~]$ sudo mv ~/jigsaw-vpn/*.*  /etc/openvpn/client/
[jigsaw@jigsaw ~]$ sudo ls -al /etc/openvpn/client
total 20
drwxr-x---. 3 root   root     94 Oct 17 23:37 .
drwxr-xr-x. 4 root   root     34 Oct 17 23:37 ..
drwxr-xr-x. 2 jigsaw jigsaw    6 Oct 17 15:07 172.16.2.23
-rw-r--r--. 1 jigsaw jigsaw 1655 Oct 15 17:02 ca.crt
-rw-r--r--. 1 jigsaw jigsaw  192 Oct 17 15:09 jigsaw.conf
-rw-r--r--. 1 jigsaw jigsaw 5360 Oct 15 17:02 jigsaw.crt
-rw-------. 1 jigsaw jigsaw 1704 Oct 15 17:02 jigsaw.key

```

**3. 配置和启动客户端服务**

```bash
sudo systemctl enable openvpn-client@jigsaw
sudo systemctl start openvpn-client@jigsaw
```

这就将openvpn作为系统服务来启动了。

### 1.2 安装Kubernetes

参考： https://kubernetes.io/docs/getting-started-guides/centos/centos_manual_config/

**1. 安装软件**

配置安装源， 创建这个文件

```bash
sudo vi /etc/yum.repos.d/virt7-docker-common-release.repo 
```
内容如下：

```bash
[virt7-docker-common-release]
name=virt7-docker-common-release
baseurl=http://cbs.centos.org/repos/virt7-docker-common-release/x86_64/os/
gpgcheck=0
```

安装Kubernetes, etcd 和 flannel， 同时，作为依赖， docker和cadvisor也会被安装。 
```bash
sudo yum -y install --enablerepo=virt7-docker-common-release kubernetes etcd flannel
```

修改机器名为kube[XX].jigsaw为机器名。 其中[XX]为分配的IP地址尾号，比如kube28.jigsaw。

```bash
sudo hostnamectl --static set-hostname kube[XX].jigsaw
```

修改/etc/hosts 文件，添加master节点

``` bash

172.16.2.24  kube-master.jigsaw  
172.16.2.[XX]  kube[XX].jigsaw

```

修改 /etc/kubernetes/config 文件，内容如下:

```bash
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=false"

# How the replication controller and scheduler find the kube-apiserver
KUBE_MASTER="--master=http://kube-master.jigsaw:8080"
```

关闭防火墙

```bash
sudo setenforce 0
sudo systemctl disable iptables-services firewalld
sudo systemctl stop iptables-services firewalld
```

注意，修改后，需要*重启系统*， 以让设置生效。 


**2. 修改配置文件**
配置 kubelet启动代理， 修改 /etc/kubernetes/kubelet 文件， 

```bash
# The address for the info server to serve on
KUBELET_ADDRESS="--address=0.0.0.0"

# The port for the info server to serve on
KUBELET_PORT="--port=10250"

# You may leave this blank to use the actual hostname
# !!!Check the node number, 将kubeXXX.jigsaw修改为你的机器名!!!
KUBELET_HOSTNAME="--hostname-override=kube[XX].jigsaw"

# Location of the api-server
KUBELET_API_SERVER="--api-servers=http://kube-master.jigsaw:8080"

# Add your own!
KUBELET_ARGS=""
```

配置flannel, 修改 /etc/sysconfig/flanneld 

```bash

# Flanneld configuration options

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="http://jigsaw-master:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/kube-centos/network"

# Any additional options that you want to pass
#FLANNEL_OPTIONS=""
```

启动服务

```bash
for SERVICES in kube-proxy kubelet flanneld docker; do
    sudo systemctl restart $SERVICES
    sudo systemctl enable $SERVICES
    sudo systemctl status $SERVICES
done
```

配置 kubectl

```bash
kubectl config set-cluster default-cluster --server=http://jigsaw-master:8080
kubectl config set-context default-context --cluster=default-cluster --user=default-admin
kubectl config use-context default-context
```

搞定!


## 二、Ubuntu 系统
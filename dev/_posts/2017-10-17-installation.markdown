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

1.1 安装OpenVPN

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

**4. 安装Kubernetes**


## 二、Ubuntu 系统
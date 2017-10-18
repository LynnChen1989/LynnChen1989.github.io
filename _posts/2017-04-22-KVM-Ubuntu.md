---
layout: post
title: KVM部署使用
tags:
    - 虚拟化
    - KVM
categories: 虚拟化
description:  KVM部署使用
---


# 基础安装配置

### 查看CPU是否支持KVM

```
 egrep "(svm|vmx)" /proc/cpuinfo

```

### 安装KVM及相关依赖包

```
sudo apt-get install qemu-kvm qemu virt-manager virt-viewer libvirt-bin bridge-utils
```

### 启用桥接网络

桥接的目的是虚拟机和宿主机的网路环境保持一致，都直接从上层路由器直接分配IP

```
先备份一套原有配置

sudo cp /etc/network/interfaces /etc/network/interfaces-bak


对/etc/network/interfaces 配置文件进行更改

sudo gedit /etc/network/interfaces
# Enabing Bridge networking br0 interface
auto br0
iface br0 inet static
address 192.168.1.130
network 192.168.1.0
netmask 255.255.255.0
broadcast 192.168.1.255
gateway 192.168.1.1
dns-nameservers 223.5.5.5
bridge_ports eth0
bridge_stp off

```

### 重启系统

略


### 检查安装

```
执行： `kvm-ok`命令
    
```


### 创建虚拟机

创建两个目录，一个存放虚拟机，一个存放镜像

```
mkdir -p /home/chenlin/virtual-machine/{kvm,iso}
```

创建一个新的虚拟机

```
virt-install --virt-type kvm --name myserver --vcpus 1 --ram 1024 --cdrom=/home/chenlin/virtual-machine/iso/ubuntu-16.04.1-server-amd64.iso --disk  path=/home/chenlin/virtual-machine/kvm/myserver.qcow2,size=10,format=qcow2 --network network=default --os-type=linux
```

# kvm管理



#



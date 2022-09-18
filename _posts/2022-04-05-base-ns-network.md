---
layout: post
title: 基于NameSpace的组网模型
subtitle: 
categories: K8S基础原理
tags: [Kubernetes]
---


# 组网模型

## 两个Namspace组网

两个NS的组网很简单，一对虚拟网卡，一端放置在一个NS中，简单配置即可通信

### 1、增加一对虚拟网卡

```
# ip link add type veth 
```

 执行1次出现 veth0@veth1和veth1@veth0 一对虚拟网卡
```
# ip link add type veth  
```
执行2次出现 veth2@veth3和veth3@veth2 第二对虚拟网卡, 如果不命名，都是自动编号

也可以给虚拟网卡指定具体的名字，如使用 `ip link add veth001 type veth peer name veth002` 命令
以上命令执行后，在宿主机使用`ip add`命令查看时发现虚拟网卡对是落在宿主机当前的NS中的


### 2、增加两个NS

```
# ip netns add ns1
# ip netns add ns2
```


### 3、把虚拟网卡对的两端veth0/veth1分别放到ns1/ns2中去

```
# ip link set veth0 netns ns1
# ip link set veth1 netns ns2
```
以上命令执行，就把虚拟网卡对的两端veth0放到ns1，veth1放到ns2中去了，此时你在宿主机执行`ip add`就不能再看到这个虚拟网卡对了
我们在ns1中去看看veth0

```
# ip netns exec ns1 ip add
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: veth0@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether a6:8f:38:e9:2f:c2 brd ff:ff:ff:ff:ff:ff link-netnsid 1
```
此时，可以看到veth0这端已经出现在了ns1中，但是为DOWN的状态，通过下面的命令激活网卡，

```
# ip netns exec ns1 ifconfig veth0 172.20.0.2/24 up    
# ip netns exec ns1 ip add
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
17: veth0@if18: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
    link/ether 46:2a:6d:d7:e7:72 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 172.20.0.2/24 brd 172.20.0.255 scope global veth0
       valid_lft forever preferred_lft forever
```
ns2的操作类似

```
# ip netns exec ns2 ip add
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
18: veth1@if17: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 42:c3:d2:e0:e2:ce brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.20.0.3/24 brd 172.20.0.255 scope global veth1
       valid_lft forever preferred_lft forever
    inet6 fe80::40c3:d2ff:fee0:e2ce/64 scope link 
       valid_lft forever preferred_lft forever
```


### 4、测试连通性

```
# ip netns exec ns2 ping 172.20.0.2
PING 172.20.0.2 (172.20.0.2) 56(84) bytes of data.
64 bytes from 172.20.0.2: icmp_seq=1 ttl=64 time=0.077 ms
64 bytes from 172.20.0.2: icmp_seq=2 ttl=64 time=0.050 ms
64 bytes from 172.20.0.2: icmp_seq=3 ttl=64 time=0.064 ms
^C
--- 172.20.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.050/0.063/0.077/0.014 ms
```

## 多个Namspace组网（≥3）

多个ns组网就需要通过宿主机的虚拟网桥进行组网


<img src="/assets/images/k8s/multi-namespace-net.png" width="100%" height="100%" alt="" />


### 1、前提

1、安装 bridge 工具 `yum -y install bridge-utils `


### 2、创建三个NS和三组Veth
```
# Namespace创建
[root@cd.ops.server.snake-test0.30:~]# ip netns add namespace1
[root@cd.ops.server.snake-test0.30:~]# ip netns add namespace2
[root@cd.ops.server.snake-test0.30:~]# ip netns add namespace3
[root@cd.ops.server.snake-test0.30:~]# ip netns ls
namespace3
namespace2
namespace1

# Veth创建, 连续执行三次，自动编号
[root@cd.ops.server.snake-test0.30:~]# ip link add type veth   # 0/1 一对
[root@cd.ops.server.snake-test0.30:~]# ip link add type veth   # 2/3 一对
[root@cd.ops.server.snake-test0.30:~]# ip link add type veth   # 4/5 一对
[root@cd.ops.server.snake-test0.30:~]# ip add
...省略显示...
20: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 6e:7f:a5:f9:4d:08 brd ff:ff:ff:ff:ff:ff
21: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether fe:7d:80:a5:50:01 brd ff:ff:ff:ff:ff:ff
22: veth2@veth3: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 7a:95:2f:ef:a3:02 brd ff:ff:ff:ff:ff:ff
23: veth3@veth2: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 66:9e:1c:73:31:5e brd ff:ff:ff:ff:ff:ff
24: veth4@veth5: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 66:78:e9:06:8a:a6 brd ff:ff:ff:ff:ff:ff
25: veth5@veth4: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 82:71:44:b1:2a:8e brd ff:ff:ff:ff:ff:ff
```

### 3、创建网桥 test-bridge

```
# brctl addbr test-bridge
# brctl show
```

### 4、接入网桥、NS中的网卡配置

(1)把v1,v3,v5 分别放到namespace1，namespace2，namespace3中, 并且在NS中进行网卡配置

```
# 虚拟网卡放入NS
[root@cd.ops.server.snake-test0.30:~]# ip link set veth1 netns namespace1
[root@cd.ops.server.snake-test0.30:~]# ip link set veth3 netns namespace2
[root@cd.ops.server.snake-test0.30:~]# ip link set veth5 netns namespace3

# 网卡配置并激活
[root@cd.ops.server.snake-test0.30:~]# ip netns exec namespace1 ifconfig veth1 172.19.0.2/24 up   
[root@cd.ops.server.snake-test0.30:~]# ip netns exec namespace2 ifconfig veth3 172.19.0.3/24 up     
[root@cd.ops.server.snake-test0.30:~]# ip netns exec namespace3 ifconfig veth5 172.19.0.4/24 up 

```
(2)把v0,v2,v4 接入到网桥中并配置网桥
```

# 接入网桥
[root@cd.ops.server.snake-test0.30:~]# brctl addif test-bridge veth0
[root@cd.ops.server.snake-test0.30:~]# brctl addif test-bridge veth2
[root@cd.ops.server.snake-test0.30:~]# brctl addif test-bridge veth4

# 配置网桥 
# ifconfig test-bridge 172.19.0.1 netmask 255.255.255.0 up
```

（3）在各个NS中配置网关
```
[root@cd.ops.server.snake-test0.30:~]# ip netns exec namespace1 route add default gw 172.19.0.1
[root@cd.ops.server.snake-test0.30:~]# ip netns exec namespace2 route add default gw 172.19.0.1 
[root@cd.ops.server.snake-test0.30:~]# ip netns exec namespace3 route add default gw 172.19.0.1

# 查看一个路由表

[root@cd.ops.server.snake-test0.30:~]# ip netns exec namespace1 route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.19.0.1      0.0.0.0         UG    0      0        0 veth1
172.19.0.0      0.0.0.0         255.255.255.0   U     0      0        0 veth1
```

（4）激活veth peer的宿主机这一端

```
[root@cd.ops.server.snake-test0.30:~]# ifconfig veth0 up
[root@cd.ops.server.snake-test0.30:~]# ifconfig veth2 up  
[root@cd.ops.server.snake-test0.30:~]# ifconfig veth4 up  
```

上面的步骤后，让3个NS和bridge组成了一个子网，bridge上的IP就是这个子网的网关IP,NS中的数据包通过veth设备到达bridge，bridge中的数据包
要转发到eth0上，需要做两个网络接口之间的数据包转发，要开启宿主机的`net.ipv4.ip_forward = 1`内核数据包转发参数。同时允许test-bridge和eth0之间的数据包转发:

```
[root@cd.ops.server.snake-test0.30:~]# iptables -A FORWARD -i test-bridge -o eth0 -j ACCEPT
[root@cd.ops.server.snake-test0.30:~]# iptables -A FORWARD -i eth0 -o  test-bridge -j ACCEPT   
```

### 5、连通性测试

```
[root@cd.ops.server.snake-test0.30:~]# ip netns exec namespace3 ping 172.19.0.2
PING 172.19.0.2 (172.19.0.2) 56(84) bytes of data.
64 bytes from 172.19.0.2: icmp_seq=1 ttl=64 time=0.265 ms
64 bytes from 172.19.0.2: icmp_seq=2 ttl=64 time=0.085 ms
^C
--- 172.19.0.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.085/0.175/0.265/0.090 ms
```

### 6、配置外部网络访问

以上就完成了NS1、NS2、NS3的局域网组网。但是在这个局域网的内部只是可以胡同，还是没法访问外部网络的。进一步通过NAT配置。
与外部网络通信，由于内部172.19的地址是私有，经过物理网卡时时不能识别的，需要对出去的包进行源地址转换，对回来的包的进行目的地址转换。

```
# 出去SNAT源地址转换

iptables -t nat -A POSTROUTING -s 172.19.0.0/24 -o eth0 -j MASQUERADE 
iptables -t nat -A POSTROUTING -s 172.19.0.0/24 -o ens192 -j MASQUERADE  
```
*以上命令eth0根据具体使用的网络设备名字修改*

```
[root@cd.ops.server.snake-test0.30:~]# ip netns exec namespace3 ping -c 2 14.215.177.38
PING 14.215.177.38 (14.215.177.38) 56(84) bytes of data.
64 bytes from 14.215.177.38: icmp_seq=1 ttl=53 time=35.3 ms
64 bytes from 14.215.177.38: icmp_seq=2 ttl=53 time=34.0 ms

--- 14.215.177.38 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 34.050/34.698/35.347/0.674 m
```
回来的数据目标地址转换，举个docker的例子：

```
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
```







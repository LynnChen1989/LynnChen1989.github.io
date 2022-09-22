---
layout: post
title: VXLAN的原理及实验
subtitle: 
categories: K8S基础原理
tags: [Kubernetes]
---
# VXLAN介绍

### 简介
VLAN作为传统的网络隔离技术，在标准定义中VLAN的数量只有4000个左右，无法满足大型数据中心的租户间隔离需求。另外，VLAN的二层范围一般较小且固定，无法支持虚拟机大范围的动态迁移。

VXLAN完美地弥补了VLAN的上述不足，一方面通过VXLAN中的24比特VNI字段，提供多达16M租户的标识能力，远大于VLAN的4000；另一方面，VXLAN本质上在两台交换机之间构建了一条穿越数据中心基础IP网络的虚拟隧道，将数据中心网络虚拟成一个巨型“二层交换机”，满足虚拟机大范围动态迁移的需求。 

虽然从名字上看，VXLAN是VLAN的一种扩展协议，但VXLAN构建虚拟隧道的本领已经与VLAN迥然不同了。

下面就让我们来看下，VXLAN报文到底长啥样。


<img src="/assets/images/vxlan.png" width="100%" height="100%" alt="" />


如上图所示，VTEP对VM发送的原始以太帧（Original L2 Frame）进行了以下“包装”：

**VXLAN Header**

增加VXLAN头（8字节），其中包含24比特的VNI字段，用来定义VXLAN网络中不同的租户。此外，还包含VXLAN Flags（8比特，取值为00001000）和两个保留字段（分别为24比特和8比特）。

**UDP Header**

VXLAN头和原始以太帧一起作为UDP的数据。UDP头中，目的端口号（VXLAN Port）固定为4789，源端口号（UDP Src. Port）是原始以太帧通过哈希算法计算后的值。

**Outer IP Header**

封装外层IP头。其中，源IP地址（Outer Src. IP）为源VM所属VTEP的IP地址，目的IP地址（Outer Dst. IP）为目的VM所属VTEP的IP地址。

**Outer MAC Header**

封装外层以太头。其中，源MAC地址（Src. MAC Addr.）为源VM所属VTEP的MAC地址，目的MAC地址（Dst. MAC Addr.）为到达目的VTEP的路径中下一跳设备的MAC地址。

VXLAN的报文传输必须要知道以下三个信息：

    MAC(src) - VNI - VTEP(dest)

### 基本术语

**VNI（VXLAN Network Identifier，VXLAN网络标识符）**：VXLAN通过VXLAN ID来标识，其长度为24比特。VXLAN 16M个标签数解决了VLAN标签不足的缺点。

**VTEP（VXLAN Tunnel End Point，VXLAN隧道端点）**：VXLAN的边缘设备。VXLAN的相关处理都在VTEP上进行，例如识别以太网数据帧所属的VXLAN、基于VXLAN对数据帧进行二层转发、封装/解封装报文等。VTEP可以是一台独立的物理设备，也可以是虚拟机所在服务器的虚拟交换机。

**VXLAN Tunnel**：两个VTEP之间点到点的逻辑隧道。VTEP为数据帧封装VXLAN头、UDP头、IP头后，通过VXLAN隧道将封装后的报文转发给远端VTEP，远端VTEP对其进行解封装。

**VSI（Virtual Switching Instance，虚拟交换实例）**：VTEP上为一个VXLAN提供二层交换服务的虚拟交换实例。VSI可以看作是VTEP上的一台基于VXLAN进行二层转发的虚拟交换机，它具有传统以太网交换机的所有功能，包括源MAC地址学习、MAC地址老化、泛洪等。VSI与VXLAN一一对应。

**VSI-Interface（VSI的虚拟三层接口）**：类似于Vlan-Interface，用来处理跨VNI即跨VXLAN的流量。VSI-Interface与VSI一一对应，在没有跨VNI流量时可以没有VSI-Interface。

<img src="/assets/images/vxlan-basic.jpg" width="100%" height="100%" alt="" />

# 一、点对点VXLAN

<img src="/assets/images/vxlan-p2p.png" width="100%" height="100%" alt="" />


|        | 节点1     | 节点2                           |
|--------|---------|----------------------|
| IP     | 10.168.0.43 | 10.168.0.53                   |
| 以太网设备  | ens160  | ens192                        |

### 10.168.0.43配置

#### 创建一个名为 vxlan0 ，类型为vxlan的网络接口
```shell
# ip link add vxlan0 type vxlan id 100 dstport 4789 remote 10.168.0.53 local 10.168.0.43 dev ens160
```
，一些重要参数说明： 
+ id 100：指定VNI的值，在1~2的24次方之间 
+ dstport： VTEP通信端口，IANA分配的是4789，Linux默认使用的是8472 
+ remote 10.168.0.53： 对端VTEP的地址 
+ local 10.168.0.43： 当前节点VTEP要使用的IP地址，即当前隧道口的IP地址 
+ dev ens160：当前节点用于VTEP要使用的IP地址，用来获取VTEP IP地址。

#### 为VXLAN网卡配置IP地址并启用
```shell
# ip addr add 172.16.200.2/24 dev vxlan0
# ip link set vxlan0 up
```
#### 查看信息

(1)执行`ip add`输出 
```
11: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 6a:dc:d0:63:73:25 brd ff:ff:ff:ff:ff:ff
    inet 172.16.200.2/24 scope global vxlan0
       valid_lft forever preferred_lft forever
    inet6 fe80::68dc:d0ff:fe63:7325/64 scope link 
       valid_lft forever preferred_lft forever
```

(2)执行`ip route`输出
```shell
172.16.200.0/24 dev vxlan0 proto kernel scope link src 172.16.200.2
```

(3)执行`bridge fdb show dev vxlan0`输出
```
00:00:00:00:00:00 dst 10.168.0.53 via ens160 self permanent
```
这个表项的意思是，默认的VTEP对端地址为 10.168.0.53.换句话说，原始报文经过vxlan0后，会被内核加上VXLAN头部，而外部UDP头部的目的IP地址为被冠上10.168.0.53。
在另外一台机器上（10.168.0.53）上也进行相同的配置，要保证VNI也是100，dstport也是4789，并修改VTEP的local和remote IP地址到相应的值。

### 10.168.0.53配置

```shell
# ip link add vxlan0 type vxlan id 100 dstport 4789 remote 10.168.0.43 local 10.168.0.53 dev ens192 
# ip addr add 172.16.200.3/24 dev vxlan0 
# ip link set vxlan0 up
# ip add
# ip route
# bridge fdb show dev vxlan0        
```

### 测试

(1) 在10.168.0.43上执行 `ping -c 2 172.16.200.3`
```
PING 172.16.200.3 (172.16.200.3) 56(84) bytes of data.
64 bytes from 172.16.200.3: icmp_seq=1 ttl=64 time=0.365 ms
64 bytes from 172.16.200.3: icmp_seq=2 ttl=64 time=0.376 ms

--- 172.16.200.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.365/0.370/0.376/0.020 ms
```
(2) 在10.168.0.53上执行 `ping -c 2 172.16.200.2`

```
PING 172.16.200.2 (172.16.200.2) 56(84) bytes of data.
64 bytes from 172.16.200.2: icmp_seq=1 ttl=64 time=0.471 ms
64 bytes from 172.16.200.2: icmp_seq=2 ttl=64 time=0.727 ms

--- 172.16.200.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.471/0.599/0.727/0.128 ms
```

# 二、多播模式VXLAN

<img src="/assets/images/vxlan-mutil.png" width="100%" height="100%" alt="" />

|        | 节点1     | 节点2                           |节点3 |     
|--------|---------|----------------------| ----|
| IP     | 10.168.0.43 | 10.168.0.53                   | 10.168.0.30|
| 以太网设备  | ens160  | ens192                        | ens192|


要组成同一个VXLAN网络，VTEP必须能够感知到彼此的存在，多播组本身的功能就是把网络中的某个节点组成一个虚拟的组。 **如果VXLAN要使用多播，那么底层网络结构需要支持多播功能**
本实验与前面的实验比较相似，只不过主机之间不是点对点的连接，而是通过多播组成一个虚拟的整体。 虽然加入了一个多播组，但是实际比较简单，和点对点相比就多了参数group。

### 10.168.0.43配置

#### 创建一个名为 vxlan0 ，类型为vxlan的网络接口
```shell
# ip link add vxlan0 type vxlan id 2022 dstport 4789 dev ens160 group 239.1.1.1 
```

它表示将VTEP加入一个多播组，多播组的地址是239.1.1.1，多播地址能够让设备将报文发送给一组设备，属于多播组的设备将被分配一个多播组IP，多播地址范围在224.0.0.0~239.255.255.255。

#### VXLAN接口配置地址并启用

```shell
# ip addr add 172.200.1.2/24 dev vxlan0
# ip link set dev vxlan0 up
```

#### 查看信息

（1）执行`ip add show vxlan0`得到：

```
12: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether d2:e1:4a:d6:40:27 brd ff:ff:ff:ff:ff:ff
    inet 172.200.1.2/24 scope global vxlan0
       valid_lft forever preferred_lft forever
    inet6 fe80::d0e1:4aff:fed6:4027/64 scope link 
       valid_lft forever preferred_lft forever
```
（2）执行`ip route`得到：
```
172.200.1.0/24 dev vxlan0 proto kernel scope link src 172.200.1.2 
```

（3）执行`bridge fdb show dev vxlan0`得到：
```
00:00:00:00:00:00 dst 239.1.1.1 via ens160 self permanent
```

不同的是fdb表项内容，dst字段变成了多播地址239.1.1.1，而不是前面对方的VTEP 地址，意思是原始报文经过vxlan0后，被内核添加伤VXLAN头部，其外部UDP头目的IP地址会被改成多播地址239.1.1.1。
同理，需要为通信的节点进行上诉配置，验证它们是否通过172.200.1.0/24网络互相通信。

### 10.168.0.53配置
```shell
ip link add vxlan0 type vxlan id 2022 dstport 4789 dev ens192 group 239.1.1.1 
ip addr add 172.200.1.3/24 dev vxlan0
ip link set dev vxlan0 up
ip add show vxlan0
ip route
bridge fdb show dev vxlan0
```

### 10.168.0.30配置
```shell
ip link add vxlan0 type vxlan id 2022 dstport 4789 dev ens192 group 239.1.1.1 
ip addr add 172.200.1.4/24 dev vxlan0
ip link set dev vxlan0 up
ip add show vxlan0
ip route
bridge fdb show dev vxlan0

```

### 测试

在10.168.0.43执行，发现172.168.200.0/24已经互通。

```
# ping 172.200.1.2
PING 172.200.1.2 (172.200.1.2) 56(84) bytes of data.
64 bytes from 172.200.1.2: icmp_seq=1 ttl=64 time=0.643 ms
^C
--- 172.200.1.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.643/0.643/0.643/0.000 ms


#  ping 172.200.1.3
PING 172.200.1.3 (172.200.1.3) 56(84) bytes of data.
64 bytes from 172.200.1.3: icmp_seq=1 ttl=64 time=0.074 ms
64 bytes from 172.200.1.3: icmp_seq=2 ttl=64 time=0.076 ms
^C
--- 172.200.1.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.074/0.075/0.076/0.001 ms


# ping 172.200.1.4
PING 172.200.1.4 (172.200.1.4) 56(84) bytes of data.
64 bytes from 172.200.1.4: icmp_seq=1 ttl=64 time=1.29 ms
64 bytes from 172.200.1.4: icmp_seq=2 ttl=64 time=0.393 ms
^C
--- 172.200.1.4 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.393/0.846/1.299/0.453 ms

```

# 三、多播+Bridge模式VXLAN

<img src="/assets/images/vxlan-bridge.png" width="100%" height="100%" alt="" />

前面的方法能够通过多播实现自动话的overlay网络构建，但是通信双方只有一个VTEP，但是在容器应用的领域，每一台宿主机安装好了docker后，会创建一个虚拟网桥，用于容器内部网络的对外通信接口，那么在我们跨主机的容器组网时可以使用网桥将多个虚拟机或容器放到同一个VXLAN网络中，和前面的多播相比，只是多了一块网桥，连接同一个主机上不同容器的 veth pair，这里先用 network namespace 代替容器，其实原理是一样的。创建一个network namespace ，并通过 veth pair将namespace 中的eth0 网卡连接到网桥，同时VXLAN也连接到网桥。
下面我们使用network namespace的方式来模拟多个容器。


|       | 节点1                                                             | 节点2                            |     
|-------|-----------------------------------------------------------------|--------------------------------|
| IP    | 10.168.0.43                                                     | 10.168.0.53                    |
| 以太网设备 | ens160                                                          | ens192                         | 
| 容器    | container_43_001<br/>container_43_002                           | container_53_001               | 
| 容器IP  | container_43_001[172.200.1.2]<br/>container_43_002[172.200.1.3] | container_53_001[172.200.1.99] | 


## 节点1（10.168.0.43）配置

### 首先创建VXLAN，使用多播模式

```shell
# ip link add vxlan0 type vxlan id 2022 dstport 4789 dev ens160 group 233.1.1.1
```
### 创建网桥 bridge2022 ，把vxlan0绑定到上面，并启用它们

```shell
# ip link add bridge2022 type bridge
# ip link set vxlan0 master bridge2022
# ip link set vxlan0 up
# ip link set bridge2022 up
```

### 容器模拟配置（container_43_001、container_43_002）

#### 模拟容器container_43_001（172.200.1.2/24）

```shell

(1)创建NS
# ip netns add container_43_001

(2)创建vethpeer虚拟网卡对
# ip link add veth0 type veth peer name veth1

(3)把vethpeer的veth0这一端接在bridge2022网桥上
# ip link set dev veth0 master bridge2022
# ip link set dev veth0 up

(4)把vethpeer的veth0这一端放入NS
# ip link set dev veth1 netns container_43_001

(5)对container_43_001网络配置
# ip netns exec container_43_001 ip link set lo up
# ip netns exec container_43_001 ip link set veth1 name eth0
# ip netns exec container_43_001 ip addr add 172.200.1.2/24 dev eth0
# ip netns exec container_43_001 ip link set eth0 up

(6)查看container_43_001网络配置
# ip netns exec container_43_001 ip add                              
```

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever       
15: eth0@if16: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether ea:75:c7:e5:16:89 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.200.1.2/24 scope global eth0
       valid_lft forever preferred_lft forever
```        
 	   

#### 模拟容器container_43_002（172.200.1.3/24）
```shell
# ip netns add container_43_002
# ip link add veth2 type veth peer name veth3
# ip link set dev veth2 master bridge2022
# ip link set dev veth2 up
# ip link set dev veth3 netns container_43_002
# ip netns exec container_43_002 ip link set lo up
# ip netns exec container_43_002 ip link set veth3 name eth0
# ip netns exec container_43_002 ip addr add 172.200.1.3/24 dev eth0
# ip netns exec container_43_002 ip link set eth0 up
# ip netns exec container_43_002 ip add    
```

## 节点1（10.168.0.53）配置

### 首先创建VXLAN，使用多播模式

```shell
# ip link add vxlan0 type vxlan id 2022 dstport 4789 dev ens192 group 233.1.1.1
```
### 创建网桥 bridge2022 ，把vxlan0绑定到上面，并启用它们

```shell
# ip link add bridge2022 type bridge
# ip link set vxlan0 master bridge2022
# ip link set vxlan0 up
# ip link set bridge2022 up
```

### 容器模拟配置（container_43_001、container_43_002）

####  模拟容器container_53_001（172.200.1.99/24）
```shell
# ip netns add container_53_001
# ip link add veth0 type veth peer name veth1
# ip link set dev veth0 master bridge2022
# ip link set dev veth0 up
# ip link set dev veth1 netns container_53_001
# ip netns exec container_53_001 ip link set lo up
# ip netns exec container_53_001 ip link set veth1 name eth0
# ip netns exec container_53_001 ip addr add 172.200.1.99/24 dev eth0
# ip netns exec container_53_001 ip link set eth0 up
# ip netns exec container_53_001 ip add  	   

```

## 测试

在模拟容器container_53_001中执行`ping 172.200.1.2/3`操作

```
# ip netns exec container_53_001 ping  172.200.1.2
PING 172.200.1.2 (172.200.1.2) 56(84) bytes of data.
64 bytes from 172.200.1.2: icmp_seq=1 ttl=64 time=1.08 ms
64 bytes from 172.200.1.2: icmp_seq=2 ttl=64 time=0.517 ms
^C
--- 172.200.1.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.517/0.802/1.087/0.285 ms
# ip netns exec container_53_001 ping  172.200.1.3
PING 172.200.1.3 (172.200.1.3) 56(84) bytes of data.
64 bytes from 172.200.1.3: icmp_seq=1 ttl=64 time=0.823 ms
^C
--- 172.200.1.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.823/0.823/0.823/0.000 ms
```


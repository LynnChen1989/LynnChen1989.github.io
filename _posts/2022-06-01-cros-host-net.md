---
layout: post
title: 跨主机组网Flannel
subtitle: 
categories: K8S基础原理
tags: [Kubernetes]
---

## 跨主机组网

### 规划

| IP          | 节点作用                             | Namespace                     |
|-------------|----------------------------------|-------------------------------|
| 10.168.0.94 | ETCD01(3.5.2) / Flannel(v0.17.0) | namespace94-01 namespace94-02 |
| 10.168.0.95 | ETCD02(3.5.2) / Flannel(v0.17.0) | namespace95-01 namespace95-02 |
| 10.168.0.96 | ETCD03(3.5.2) / Flannel(v0.17.0) | namespace96-01 namespace96-02 |

### Flannel原理和部署

#### （一）flannel原理

<img src="/assets/images/k8s/flannel-net.png" width="100%" height="100%" alt="" />

#### （二）前置部署ssl相关

（1）ssl工具安装
```
wget -O /usr/bin/cfssl https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64
wget -O /usr/bin/cfssljson https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64
chmod a+x /usr/bin/cfssl*
```
（2）下载安装包

```
# mkdir -p /root/install_flannel{pkg,ssl}
# etcd-v3.5.2-linux-amd64.tar.gz，flannel-v0.17.0-linux-amd64.tar.gz放到/root/install_flannel/pkg,然后解压（省略）
# cp -rp flanneld /usr/local/bin/
# cp -rp etcd-v3.5.2-linux-amd64/etcd* /usr/local/bin/
```
（3）证书
```
# cd /root/install_flannel
# cat > ssl/ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
EOF

# cat > ssl/ca-csr.json  << EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "k8s",
      "OU": "System"
    }
  ],
  "ca": {
    "expiry": "876000h"
 }
}
EOF

# cd ssl && cfssl gencert -initca ca-csr.json | cfssljson -bare ca


# cat > ssl/etcd-csr.json << EOF
{
  "CN": "etcd",
  "hosts": [　　
    "127.0.0.1",
    "10.168.0.99",
    "10.168.0.98",
    "10.168.0.97"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ChengDu",
      "L": "ChengDu",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF


# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```

(4)ECTD相关证书
```
[root@cd.ops.k8s-node-01:/opt/etcd]#ll ssl/
total 20
-rw-r--r-- 1 root root 1318 Apr 11 11:17 ca.pem
-rw-r--r-- 1 root root 1062 Apr 11 11:18 etcd.csr
-rw-r--r-- 1 root root  293 Apr 11 11:17 etcd-csr.json
-rw------- 1 root root 1679 Apr 11 11:18 etcd-key.pem
-rw-r--r-- 1 root root 1440 Apr 11 11:18 etcd.pem
```

#### （三）ETCD集群部署

**尤其注意v2和v3的区别，和操作区别**

（1）创建ETCD运行目录和配置文件  `mkdir /opt/etcd && mkdir -p /opt/etcd/ssl && mkdir -p /var/lib/etcd/default.etcd` 


（2）配置文件 `vim /opt/etcd/etcd.conf`

```
#[Member]
ETCD_NAME="etcd01"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://10.168.0.94:2380"   #集群通信端口
ETCD_LISTEN_CLIENT_URLS="https://10.168.0.94:2379" #监听的数据端口

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.168.0.94:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.168.0.94:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://10.168.0.94:2380,etcd02=https://10.168.0.95:2380,etcd03=https://10.168.0.96:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"  #认证token
ETCD_INITIAL_CLUSTER_STATE="new"                                                   
```
(3) ETCD证书拷贝到运行目录 

``` 
# cp -rp /root/install_flannel/ssl/etcd* /opt/etcd/ssl/
# cp -rp /root/install_flannel/ssl/ca.pem /opt/etcd/ssl/
```

(4) 创建ETCD服务 `vim /usr/lib/systemd/system/etcd.service` --enable-v2 参数开启

```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=-/opt/etcd/etcd.conf
ExecStart= /usr/local/bin/etcd \
--enable-v2 \
--cert-file=/opt/etcd/ssl/etcd.pem \
--key-file=/opt/etcd/ssl/etcd-key.pem \
--peer-cert-file=/opt/etcd/ssl/etcd.pem \
--peer-key-file=/opt/etcd/ssl/etcd-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
(5)启动服务

```
systemctl daemon-reload
systemctl enable etcd
systemctl restart etcd
```
(6)查看状态

```
etcdctl --endpoints=https://10.168.0.94:2379,https://10.168.0.95:2379,https://10.168.0.96:2379  --cacert=/opt/etcd/ssl/ca.pem  --cert=/opt/etcd/ssl/etcd.pem  --key=/opt/etcd/ssl/etcd-key.pem  endpoint health
```

#### （四）Flannel部署

（1）下载解压（略）

（2）创建运行目录 `mkdir /opt/flannel`

（3）创建配置文件 `cd /opt/flannel && vim flanneld.conf `

```
FLANNEL_ETCD="-etcd-endpoints=https://10.168.0.94:2379,https://10.168.0.95:2379,https://10.168.0.96:2379"
FLANNEL_ETCD_KEY="-etcd-prefix=/xwfintech.com/network"
FLANNEL_ETCD_CAFILE="-etcd-cafile=/opt/etcd/ssl/ca.pem"
FLANNEL_ETCD_CERTFILE="-etcd-certfile=/opt/etcd/ssl/etcd.pem"
FLANNEL_ETCD_KEYFILE="-etcd-keyfile=/opt/etcd/ssl/etcd-key.pem"
```
（4）创建service `vim /usr/lib/systemd/system/flanneld.service `

```
[Unit]
Description=Flanneld overlay address etcd agent
After=network-online.target network.target
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/opt/flannel/flanneld.conf
ExecStart=/usr/local/bin/flanneld --ip-masq ${FLANNEL_ETCD} ${FLANNEL_ETCD_KEY} ${FLANNEL_ETCD_CAFILE} ${FLANNEL_ETCD_CERTFILE} ${FLANNEL_ETCD_KEYFILE}
Restart=on-failure

[Install]
WantedBy=multi-user.target
                               
```
（5）启动Flannel
```
systemctl daemon-reload
systemctl start flanneld  
```
**其他主机一样的安装完毕**
### 不同NS网络组网

上面的flannel安装启动完毕后，三个主机上的flanneld只是处于loaded状态，不会处于active状态，因为我们还没定义网络。
为了简化操作，我们添加一条alias，`alias etcdopt="ETCDCTL_API=2 etcdctl --endpoints=https://10.168.0.94:2379,https://10.168.0.95:2379,https://10.168.0.96:2379  --ca-file=/opt/etcd/ssl/ca.pem  --cert-file=/opt/etcd/ssl/etcd.pem  --key-file=/opt/etcd/ssl/etcd-key.pem"`
后续的操作都以`etcdopt`命令来执行。

```
etcdopt set /xwfintech.com/network/config '{ "Network": "172.22.0.0/16"}'
```
**注意：** flannel只能使用v2来操作，v3不兼容

使用上面的命令为ETCD写入网络的网段后，此时我们查看宿主机的三台主机多了一个flannel0的网卡分别如下

```
# 10.168.0.94 主机
3: flannel0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1472 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none 
    inet 172.22.82.0/32 scope global flannel0
       valid_lft forever preferred_lft forever
    inet6 fe80::2660:9d65:f860:9613/64 scope link flags 800 
       valid_lft forever preferred_lft forever

# 10.168.0.95 主机       
3: flannel0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1472 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none 
    inet 172.22.13.0/32 scope global flannel0
       valid_lft forever preferred_lft forever
    inet6 fe80::5bc9:a56:90f4:400c/64 scope link flags 800 
       valid_lft forever preferred_lft forever
       
# 10.168.0.96 主机       
3: flannel0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1472 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none 
    inet 172.22.66.0/32 scope global flannel0
       valid_lft forever preferred_lft forever
    inet6 fe80::d965:b340:7900:9f8f/64 scope link flags 800 
       valid_lft forever preferred_lft forever              
```
上面三个宿主机我们可以看到flannel0配置的IP分别是`172.22.82.0/32`、`172.22.13.0/32`、`172.22.66.0/32`, 我们看下ETCD中的数据

```
# etcdopt ls /xwfintech.com/network/subnets
/xwfintech.com/network/subnets/172.22.13.0-24
/xwfintech.com/network/subnets/172.22.82.0-24
/xwfintech.com/network/subnets/172.22.66.0-24
```
#### 查看Flannel配置了些什么

```
# etcdopt get /xwfintech.com/network/subnets/172.22.13.0-24 
{"PublicIP":"10.168.0.95","PublicIPv6":null}
```

flannel默认把网络定义信息写入到本地的 `/run/flannel/subnet.env`中，查看文件中的信息如下：

```
FLANNEL_NETWORK=172.22.0.0/16
FLANNEL_SUBNET=172.22.82.1/24
FLANNEL_MTU=1472
FLANNEL_IPMASQ=true
```

flannel创建的iptables转发规则如下：

```
[root@cd.ops.k8s-node-01:~]#iptables -S -t nat
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-A POSTROUTING -s 172.22.0.0/16 -d 172.22.0.0/16 -m comment --comment "flanneld masq" -j RETURN
-A POSTROUTING -s 172.22.0.0/16 ! -d 224.0.0.0/4 -m comment --comment "flanneld masq" -j MASQUERADE
-A POSTROUTING ! -s 172.22.0.0/16 -d 172.22.82.0/24 -m comment --comment "flanneld masq" -j RETURN
-A POSTROUTING ! -s 172.22.0.0/16 -d 172.22.0.0/16 -m comment --comment "flanneld masq" -j MASQUERADE


[root@cd.ops.k8s-node-01:~]#iptables -S
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
-A FORWARD -s 172.22.0.0/16 -m comment --comment "flanneld forward" -j ACCEPT
-A FORWARD -d 172.22.0.0/16 -m comment --comment "flanneld forward" -j ACCEPT
```

### Docker使用Flannel容器组网

#### 安装Docker(主机0.94,其他主机配置类似)

```
# yum install -y yum-utils
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# yum-config-manager --enable docker-ce-nightly
# yum-config-manager --enable docker-ce-test
# yum-config-manager --disable docker-ce-nightly
# yum install docker-ce docker-ce-cli containerd.io
```

#### 默认网卡查看`ip add`
```
.....省略.....
3: flannel0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1472 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none 
    inet 172.22.82.0/32 scope global flannel0
       valid_lft forever preferred_lft forever
    inet6 fe80::9d7e:302f:5b01:2506/64 scope link flags 800 
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:1b:3a:f9:2e brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```

#### 查看flannel网络配置`cat /run/flannel/subnet.env`
```
FLANNEL_NETWORK=172.22.0.0/16
FLANNEL_SUBNET=172.22.82.1/24
FLANNEL_MTU=1472
FLANNEL_IPMASQ=true
```
#### 修改docker service后重启

```
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --bip=172.22.82.1/24 --mtu=1472
```
增加`--bip=172.22.82.1/24 --mtu=1472` 和 `/run/flannel/subnet.env`里面的保持一致。
--bip参数，会把172.22.82.1作为网关，然后--bip=172.22.82.0/24作为当前宿主机上所有container分配的ip地址池。

#### 再次查看网卡`ip add`

```
...... 省略 ......
3: flannel0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1472 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none 
    inet 172.22.82.0/32 scope global flannel0
       valid_lft forever preferred_lft forever
    inet6 fe80::9d7e:302f:5b01:2506/64 scope link flags 800 
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:1b:3a:f9:2e brd ff:ff:ff:ff:ff:ff
    inet 172.22.82.1/24 brd 172.22.82.255 scope global docker0
       valid_lft forever preferred_lft forever
```
其他主机按照类似的配置完毕

#### 测试

（1）在0.94上运行web1
```
# docker run -itd --name web1 hub.xwfintech.com/busybox
# docker exec -it web1 ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
5: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1472 qdisc noqueue state UP group default 
    link/ether 02:42:ac:16:52:02 brd ff:ff:ff:ff:ff:ff
    inet 172.22.82.2/24 brd 172.22.82.255 scope global eth0
       valid_lft forever preferred_lft forever
```
（2）在0.95上运行web2
```
# docker run -itd --name web2 hub.xwfintech.com/busybox
# docker exec -it web2 ip add 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
5: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1472 qdisc noqueue state UP group default 
    link/ether 02:42:ac:16:0d:02 brd ff:ff:ff:ff:ff:ff
    inet 172.22.13.2/24 brd 172.22.13.255 scope global eth0
       valid_lft forever preferred_lft forever
```
(3) 在0.95上web2到0.94的web1进行连通性测试

```
#docker exec -it web2 ping -c 1 172.22.82.2    
PING 172.22.82.2 (172.22.82.2): 56 data bytes
64 bytes from 172.22.82.2: seq=0 ttl=60 time=1.337 ms

--- 172.22.82.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 1.337/1.337/1.337 ms
```
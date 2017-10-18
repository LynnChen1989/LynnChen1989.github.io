---
layout: post
title: shadowsocks
tags: shadowsocks
    - shadowsocks
categories: 网络
description:  shadowsocks 
---

[引言] shadowsocks搭建， [shadowsocks搭建](https://shadowsocks.org/en/download/servers.html)

## 安装

### Debian / Ubuntu

    apt-get install python-pip
    pip install shadowsocks

### CentOS

    yum install python-setuptools && easy_install pip
    pip install shadowsocks
    
## 服务端配置（墙外）

建议把 SS 的配置文件放置在当前用户主目录下的 ss 文件夹内，对于 root 用户而言，则是：/root/ss 目录。其余用户一般则是：/home/用户名 目录。

下面我们以 root 用户为例：

在 root 文件夹内，新建 ss/ssserver.json 配置文件：vim ~/ss/ssserver.json

编辑配置文件，依然是按 i 进入编辑，按 ESC 退出编辑，按 :wq 保存并退出：

```
{
    "server": "my_server_ip", // 这里输入本机的 IP 地址
    "server_port": 8388, // 为了安全，可修改为大于 1024 的数字
    "local_address": "127.0.0.1",
    "local_port": 1080, // 为了安全，可修改为大于 1024 的数字
    "password": "mypassword", // 设置一个密码
    "timeout": 300,
    "method": "aes-256-cfb",
    "fast_open": false
}
```

**启动**

    nohup ssserver -c /root/ss/ssserver.json -d start &
    
**停止**
    
    ssserver -c /root/ss/ssserver.json -d stop
    

## 客户端配置（墙内）

### Linux

安装方式和服务端一样， 配置文件示例, 这次依然把配置文件放置在当前用户主目录下的 ss 文件夹内，只不过配置文件命名为 sslocal.json：

```
{
    "server": "my_server_ip", // 这里输入墙外服务器的 IP 地址
    "server_port": 8388, // 与 ssserver.json 配置文件设置同样的端口
    "local_address": "127.0.0.1",
    "local_port": 1080, // 为了安全，可修改为大于 1024 的数字
    "password": "mypassword", // 填写 ssserver.json 配置文件中设置的密码
    "timeout": 300,
    "method": "aes-256-cfb",
    "fast_open": false
}
```

**启动**

    nohup sslocal -c /root/ss/sslocal.json -d start &
    
**停止**

    sslocal -c /root/ss/sslocal.json -d stop
    

由于 Shadowsocks 使用的是 SOCKS5 协议，必须把 SOCKS5 请求转化为 HTTP 协议请求，墙内服务器的其他软件才能使用该代理。这里使用 **Privoxy** 软件进行协议请求转换。

+ 安装

    yum install privoxy -y
    
+ 编辑 Privoxy 配置文件，将 SOCKS5 协议转化为 HTTP 协议： 
    
    vim /etc/privoxy/config

+ 在配置文件中增加这个配置： 

    forward-socks5 / 127.0.0.1:1080 . （这最后面确实有个英文句号，不要遗漏）

+ 设置 Privoxy 随系统自动启动： 

    systemctl enable privoxy

+ 启动 Privoxy： 

    systemctl start privoxy

+  查看 Privoxy 状态： 
    
    systemctl status privoxy， 如果有 running 和 active 字样，说明成功运行。


**为当前服务器用户设置 Bash 代理**

在 .bash_profile 的最后，增加一句命令： 

        export http_proxy=http://127.0.0.1:8118

8118为本机privoxy监听的端口

# 使用C版本

### ubuntu
    
    sudo apt update
    sudo apt install shadowsocks-libev

### centos
    
    1.添加仓库
    cd /etc/yum.repos.d/ && wget https://copr.fedorainfracloud.org/coprs/librehat/shadowsocks/repo/epel-7/librehat-shadowsocks-epel-7.repo
    
    2.安装
    yum install shadowsocks-libev
    
### 客户端

    配置文件所有的都类似
    
    启动
    
    ss-local -c /root/ss/sslocal.json > /tmp/sslocal.log 2>&1 &
    
 
### 自动路由配置 
    
    通过ss-redir的方式
    

##### 拉取大清的ip加入到ipset

这一步可以到git上，有很多脚本，直接拉取会直接翻不过去
    
    # wget -O- 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /root/chnroute-full.txt

    # ipset create chnroute hash:net
    
    # cat /root/chnroute-full.txt | sudo xargs -I ip ipset add chnroute ip
    
    
添加防火墙过滤规则

    ```
    iptables -t nat -N SHADOWSOCKS
    iptables -t nat -A SHADOWSOCKS -d 104.223.111.157 -j RETURN
    # 120.77.221.93 就是你墙外ss服务器的ip
    iptables -t nat -A SHADOWSOCKS -d 0.0.0.0/8 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 10.0.0.0/8 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 127.0.0.0/8 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 169.254.0.0/16 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 172.16.0.0/12 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 192.168.0.0/16 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 224.0.0.0/4 -j RETURN
    iptables -t nat -A SHADOWSOCKS -d 240.0.0.0/4 -j RETURN
    
    iptables -t nat -A SHADOWSOCKS -p tcp -m set --match-set chnroute dst -j RETURN
    iptables -t nat -A SHADOWSOCKS -p udp -m set --match-set chnroute dst -j RETURN
    iptables -t nat -A SHADOWSOCKS -p icmp -m set --match-set chnroute dst -j RETURN
    
    iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-port 1080
    iptables -t nat -A SHADOWSOCKS -p udp -j REDIRECT --to-port 1080
    iptables -t nat -A SHADOWSOCKS -p icmp -j REDIRECT --to-port 1080
    
    iptables -t nat -A OUTPUT -p tcp -j SHADOWSOCKS
    iptables -t nat -A OUTPUT -p udp -j SHADOWSOCKS
    iptables -t nat -A OUTPUT -p icmp -j SHADOWSOCKS
    ```
    
 可以把上诉的步骤写入一个脚本中。
 
 启动ss-redir
    
    ss-redir -c /root/ss/ss-redir.json
    
    
---
layout: post
title: ssh+proxychains科学上网
tags:
    - proxychains
categories: 网络
description:  ssh+proxychains
---

[引言]ssh+proxychains科学上网

### 安装proxychains
    
    # cd ~
    # git clone https://github.com/rofl0r/proxychains-ng.git 
    # cd proxychains-ng 
    # ./configure && make && make install  
    # make install-config  #　默认安装的配置文件路径为/usr/local/etc/proxychains.conf
    
### 配置
    
    # vim /usr/local/etc/proxychains.conf
    
    更改：
    socks4 127.0.0.1 9050
    socks5 127.0.0.1 9050
    
    注释：
    strict_chain
    
    取消注释：
    dynamic_chain
    
    
9050为本地的翻墙端口， 可以是shadowsocks的sslocal运行的端口，也可以是ssh翻墙监听的端口， 本文主要说明的是ssh


### 启动ssh
    
    ssh -D 9050 root@104.223.111.157 

### 使用

    proxychains4 w3m https://www.google.com.vn/

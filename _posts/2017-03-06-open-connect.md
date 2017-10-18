---
layout: post
title: openconnect 搭建VPN
tags:
    - VPN
    - openconnect
categories: 网络
description: openconnect 搭建VPN
---

[引言]通过openconnect 搭建VPN

### 安装依赖包

```
yum -y install wget gcc nettle* gnutls  gnutls-devel *readline* libev libev-devel autogen protobuf* 
```


### 安装ocserv

```
    # wget ftp://ftp.infradead.org/pub/ocserv/ocserv-0.11.7.tar.xz
    # xz -d ocserv-0.11.7.tar.xz
    # tar xf ocserv-0.11.7.tar
    # cd ocserv-0.11.7 && ./configure --prefix=/usr/local/ocserv && make && make install
```

### 配置ocserv

参考文档： http://www.infradead.org/ocserv/manual.html

#### 证书
    
##### 生成CA

1、生成ca-key

```
    certtool --generate-privkey --outfile ca-key.pem 
```

2、ca模板



```
# vi ca.tmpl

cn = "VPN CA" 
organization = "Big Corp" 
serial = 1 
expiration_days = -1 
ca 
signing_key 
cert_signing_key 
crl_signing_key
```


3、使用模板生成证书

```
certtool --generate-self-signed --load-privkey ca-key.pem --template ca.tmpl --outfile ca-cert.pem
```

##### 服务器证书

1、生成ca-key

```
certtool --generate-privkey --outfile server-key.pem 
```

2、ca模板

```
# vi server.tmpl

cn = "VPN server" 
dns_name = "www.example.com" 
dns_name = "vpn1.example.com" 
#ip_address = "1.2.3.4" 
organization = "MyCompany" 
expiration_days = -1 
signing_key 
encryption_key #only if the generated key is an RSA one 
tls_www_server 
```


3、使用模板生成证书

```
certtool --generate-certificate --load-privkey server-key.pem --load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem --template server.tmpl --outfile server-cert.pem
```

##### 客户端证书

客户端使用证书来进行认证的时候需要，如果使用anyconnect, 可以不用，也可以用。

```
$ certtool --generate-privkey --outfile user-key.pem 
$ cat << _EOF_ >user.tmpl 
cn = "user" 
unit = "admins" 
expiration_days = 365 
signing_key 
tls_www_client 
_EOF_ 
$ certtool --generate-certificate --load-privkey user-key.pem --load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem --template user.tmpl --outfile user-cert.pem

$ certtool --to-p12 --load-privkey user-key.pem --pkcs-cipher 3des-pkcs12 --load-certificate user-cert.pem --outfile user.p12 --outder
```


#### ocserv配置

```
# mkdir /usr/local/ocserv/etc
# mkdir /usr/local/ocserv/etc/certificates

复制源码里面的sample.config， sample.passwd到/usr/local/ocserv/etc

# cd /usr/local/ocserv/etc && cp /home/centos/ocserv-0.11.7/doc/sample.passwd .
# cd /usr/local/ocserv/etc && cp /home/centos/ocserv-0.11.7/doc/sample.config .

增加一个用户
/usr/local/ocserv/bin/ocpasswd -c sample.passwd snake


最终/usr/local/ocserv目录结构为：
.
├── bin
│   ├── occtl
│   ├── ocpasswd
│   └── ocserv-fw
├── etc
│   ├── certificates
│   │   ├── ca-cert.pem
│   │   ├── ca-key.pem
│   │   ├── ca.tmpl
│   │   ├── server-cert.pem
│   │   ├── server-key.pem
│   │   └── server.tmpl
│   ├── sample.config
│   └── sample.passwd
├── sbin
│   └── ocserv
└── share
    └── man
        └── man8
            ├── occtl.8
            ├── ocpasswd.8
            └── ocserv.8

7 directories, 15 files

```



配置：sample.config

```
auth = "plain[passwd=./sample.passwd]"
tcp-port = 443
udp-port = 443
run-as-user = root
run-as-group = root
socket-file = /var/run/ocserv-socket
server-cert = /usr/local/ocserv/etc/certificates/server-cert.pem
server-key = /usr/local/ocserv/etc/certificates/server-key.pem
ca-cert = /usr/local/ocserv/etc/certificates/ca-cert.pem
isolate-workers = false
max-clients = 16
max-same-clients = 2
keepalive = 32400
dpd = 90
mobile-dpd = 1800
switch-to-tcp-timeout = 25
try-mtu-discovery = false
cert-user-oid = 0.9.2342.19200300.100.1.1
tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-VERS-SSL3.0"
auth-timeout = 240
min-reauth-time = 300
max-ban-score = 50
ban-reset-time = 300
cookie-timeout = 300
deny-roaming = false
rekey-time = 172800
rekey-method = ssl
use-occtl = true
pid-file = /var/run/ocserv.pid
device = vpns
predictable-ips = true
ipv4-network = 172.16.16.0
ipv4-netmask = 255.255.255.0
ping-leases = false

# 路由表示要改变路由的主机或者网段
route = 10.200.0.0/255.255.0.0
route = 10.100.0.0/255.255.0.0
route = 10.90.0.0/255.255.0.0
no-route = 192.168.0.0/255.255.0.0
cisco-client-compat = true
dtls-legacy = true
```

##### 启动

```
/usr/local/ocserv/sbin/ocserv -f -c /usr/local/ocserv/etc/sample.config -d 1
```

### iptables NAT 配置

```
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

### 路由转发功能

```
永久添加： vi  /etc/sysctl.conf
增加 net.ipv4.ip_forward = 1

临时添加：
echo 1 > /proc/sys/net/ipv4/ip_forward

```

### 客户端配置

1、windows可使用思科anyconnect

2、linux

```
yum -y install vpnc-script openconnect


使用：

openconnect https://ops-vpn.camera360.com/

```

### 结合FREEIPA

##### 配置文件修改

```text
auth = "pam"
tcp-port = 1443
udp-port = 1443
run-as-user = root
run-as-group = root
socket-file = /var/run/ocserv-socket
server-cert = /usr/local/ocserv/etc/certs/ivpn.ilongyuan.cn.cert
server-key = /usr/local/ocserv/etc/certs/ivpn.ilongyuan.cn.key
isolate-workers = false
max-clients = 32
max-same-clients = 2
keepalive = 32400
dpd = 90
mobile-dpd = 1800
switch-to-tcp-timeout = 25
try-mtu-discovery = false
cert-user-oid = 2.5.4.3
tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-VERS-SSL3.0"
auth-timeout = 240
min-reauth-time = 300
max-ban-score = 50
ban-reset-time = 300
cookie-timeout = 300
deny-roaming = false
rekey-time = 172800
rekey-method = ssl
use-occtl = true
pid-file = /var/run/ocserv.pid
device = vpns
predictable-ips = true
ipv4-network = 172.16.16.0
ipv4-netmask = 255.255.255.0
ping-leases = false
route = 172.16.249.32/255.255.255.255
route = 172.16.254.250/255.255.255.255
route = 172.16.254.253/255.255.255.255
route = 192.168.10.206/255.255.255.255
cisco-client-compat = true
dtls-legacy = true
```
修改认证方式为pam


增加 vim /etc/pam.d/ocserv

```text
#%PAM-1.0
auth       include      password-auth
account    required     pam_nologin.so
account    include      password-auth
session    include      password-auth

```

##### 注册freeipa

将安装ocserv的服务器注册到freeipa

    # yum -y install ipa-client
    # ipa-client-install
 
根据自己实际情况填写如下信息，然后等待安装完毕。



```text
DNS discovery failed to determine your DNS domain
	Provide the domain name of your IPA server (ex: example.com): ipa.ops.ilongyuan.cn
	Provide your IPA server name (ex: ipa.example.com): ipa.ops.ilongyuan.cn
	The failure to use DNS to find your IPA server indicates that your resolv.conf file is not properly configured.
	Autodiscovery of servers for failover cannot work with this configuration.
	If you proceed with the installation, services will be configured to always access the discovered server for all operations and will not fail over to other servers in case of failure.
	Proceed with fixed values and no DNS discovery? [no]: yes
	Client hostname: ivpn.ilongyuan.cn
	Realm: OPS.ILONGYUAN.CN
	DNS Domain: ipa.ops.ilongyuan.cn
	IPA Server: ipa.ops.ilongyuan.cn
	BaseDN: dc=ops,dc=ilongyuan,dc=cn

	Continue to configure the system with these values? [no]: yes
	Synchronizing time with KDC...
	Attempting to sync time using ntpd.  Will timeout after 15 seconds
	Unable to sync time with NTP server, assuming the time is in sync. Please check that 123 UDP port is opened.
	User authorized to enroll computers: admin
	Password for admin@OPS.ILONGYUAN.CN: 
	Successfully retrieved CA cert
```



##### freeipa配置

在freeipa的web中操作，"允许登录ocserv服务器"
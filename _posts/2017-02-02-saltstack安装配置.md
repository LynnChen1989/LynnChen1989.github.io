---
layout: post
title: saltstack安装配置
tags:
    - saltstack
categories: saltstack
description:  saltstack安装配置 
---

[引言] saltstack官网配置 https://docs.saltstack.com/en/latest/contents.html

# 简介

Salt一种全新的基础设施管理方式，部署轻松，在几分钟内可运行起来，扩展性好，很容易管理上万台服务器，速度够快，服务器之间秒级通讯。

salt底层采用动态的连接总线, 使其可以用于编配, 远程执行, 配置管理等等。

# 安装

各种平台 `Arch Linux` `Debian GNU/Linux / Raspbian` `Fedora` `Ubuntu` `Windows` `SUSE` 具体的安装方式参考官方文档即可

这里介绍python pip的安装方式， 我们知道saltstack是基于python开发的， 所以肯定可以基于pip来进行安装。

```text
1、 创建python虚拟环境
/usr/local/python27/bin/virtualenv /usr/local/develop/python_ve/python27
2、 切换虚拟环境
. /usr/local/develop/python_ve/python27/bin/activate
3、 安装salt
pip install salt
```

安装完成后在当前的python虚拟环境中可以执行salt的各种cli， 但是退出了当前python的虚拟环境后，如果要使用salt的cli需要使用绝对路径。

下面的所有操作都是基于绝对路径。

# 配置

在使用pip的方式安装之后，并不会有配置文件的产生，如果需要产生配置文件，以centos为例， 需要执行 `yum -y install salt`。

执行完成后，会在/etc目录中产生一个配置目录名为salt， 配置目录结构如下：

```text

root@192.168.238.128:/usr/local/salt [22:29:58] # tree -L 10 /etc/salt/
/etc/salt/
├── cloud
├── cloud.conf.d
├── cloud.deploy.d
├── cloud.maps.d
├── cloud.profiles.d
├── cloud.providers.d
├── master
├── master.d
├── minion
├── minion.d
│   └── _schedule.conf
├── pki
│   ├── master
│   └── minion
├── proxy
├── proxy.d
└── roster

11 directories, 6 files

```
我们主要用到master和minion两个文件， master为主控端的配置文件， minion为受控端的配置文件，其余的配置文件和配置目录可以参考文档研究。

**需要说明一下：**

由于几乎所有的cli操作都有一个-c参数来制定配置目录，默认是/etc/salt。

所以不建议当需要个性化的统一环境的时候把/etc/salt这个目录移动到别的路径，避免带来的麻烦就是，每一次需要执行一个cli操作，如slat-key，都需要手动指定-c这个参数，很麻烦，也很坑爹。

### master配置

快速的配置master是比较简单的，但是要配置的非常详细，需要去熟读官方文档，master的配置文件，包括其中的注释有1000+行。
 
一个简单的master配置示例：

```text

default_include: master.d/*.conf
interface: 0.0.0.0
ipv6: False
publish_port: 4505
user: root
ret_port: 4506
pidfile: /var/run/salt-master.pid
root_dir: /usr/local/salt
conf_file: /etc/salt/master
pki_dir: /etc/salt/pki/master
cachedir: /var/cache/salt/master
module_dirs:
   - /var/cache/salt/minion/extmods
verify_env: True
keep_jobs: 24
gather_job_timeout: 10
timeout: 5
loop_interval: 60
output: nested
show_timeout: True
color: True
strip_colors: False
sock_dir: /var/run/salt/master
job_cache: True
minion_data_cache: True
cache: localfs
max_open_files: 100000
worker_threads: 6
open_mode: False
auto_accept: True
autosign_timeout: 120
autosign_file: /etc/salt/autosign.conf
autoreject_file: /etc/salt/autoreject.conf
state_top: top.sls
file_roots:
  base:
    - /srv/salt/
  dev:
    - /srv/salt/dev/services
    - /srv/salt/dev/states
  prod:
    - /srv/salt/prod/services
    - /srv/salt/prod/states
log_file: /var/log/salt/master.log
log_level: info
log_level_logfile: info
log_datefmt_logfile: '%Y-%m-%d %H:%M:%S'
log_fmt_console: '%(colorlevel)s %(colormsg)s'

```

##### 特别注意

conf_file， 指定配置文件入口的绝对路径

root_dir， 这个参数，是用来指明 `pki_dir`, `cachedir`, `sock_dir`, `log_file`, `autosign_file`, `autoreject_file`, `extension_modules`, `key_logfile`, `pidfile`

这几个参数的前缀，所以，**后面这几个参数的所定义的路径都是root_dir的相对路径，** 这个参数在配置文件的注释中有做出说明:

```text

37 # The root directory prepended to these options: pki_dir, cachedir,
38 # sock_dir, log_file, autosign_file, autoreject_file, extension_modules,
39 # key_logfile, pidfile

```

这就可以用来个性化的定义，这些参数最终产生数据的最终位置，我个人习惯使用`/usr/local/salt`路径，所以当我最终配置后运行，大概会有如下内容产生：

```text

root@192.168.238.128:/usr/local/salt [22:42:48] # tree -L 3 /usr/local/salt/
/usr/local/salt/
├── etc
│   └── salt
│       ├── minion_id
│       └── pki
├── pki
│   ├── master
│   │   ├── master.pem
│   │   ├── master.pub
│   │   ├── minions
│   │   ├── minions_autosign
│   │   ├── minions_denied
│   │   ├── minions_pre
│   │   └── minions_rejected
│   └── minion
│       ├── minion_master.pub
│       ├── minion.pem
│       └── minion.pub
├── run
└── var
    ├── cache
    │   └── salt
    ├── log
    │   └── salt
    └── run
        ├── salt
        ├── salt-master.pid
        └── salt-minion.pid

19 directories, 8 files

```

### minion配置

minion的配置相对来说简单一些，一个参考的配置示例如下

```text

default_include: minion.d/*.conf
master: 192.168.238.128
master_port: 4506
user: root
pidfile: /var/run/salt-minion.pid
root_dir: /usr/local/salt
conf_file: /etc/salt/minion
pki_dir: /etc/salt/pki/minion
minion_id_caching: True
cachedir: /var/cache/salt/minion
verify_env: True
sock_dir: /var/run/salt/minion
output: nested
color: True
strip_colors: False
file_roots:
  base:
    - /srv/salt/
  dev:
    - /srv/salt/dev/services
    - /srv/salt/dev/states
  prod:
    - /srv/salt/prod/services
    - /srv/salt/prod/states
log_file: /var/log/salt/minion.log
key_logfile: /var/log/salt/key
log_level: info
log_level_logfile: info
log_datefmt_logfile: '%Y-%m-%d %H:%M:%S'
log_fmt_console: '%(colorlevel)s %(colormsg)s'
```

主要的参数就是master，用来指定master所在的ip或者主机名，master_port用来指定进行注册交互的端口。

配置文件中存在id参数，用来指定minion端的一个标识，一般用ip或者主机名，如果没有指定，会根据主机名产生一个minion_id文件，

该文件的位置为： `<root_dir>/etc/salt/minion_id`, 可以删除，删除后重启minion又会产生一个。

# 运行

```text
# master
/usr/local/develop/python_ve/python27/bin/salt-master -c /etc/salt/ > /dev/null 2>&1 &

# minion
/usr/local/develop/python_ve/python27/bin/salt-minion -c /etc/salt/ > /dev/null 2>&1 &
```

# 测试

```text
root@192.168.238.128:/usr/local/salt [22:44:44] # /usr/local/develop/python_ve/python27/bin/salt '*' test.ping
192.168.238.128:
    True
```


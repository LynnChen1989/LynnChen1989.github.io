---
layout: post
title: saltstack可视化工具halite
tags:
    - saltstack
    - halite
categories: saltstack
description:  saltstack可视化工具halite 
---

[引言] halite是saltstack的图形化管理工具，虽然比较简陋，但是也提供一个可参考的使用方式

需要提前安装slat-api，通过python pip安装的salt都是自动安装了该工具，其他方式安装的，需要重新安装salt-api

#### 克隆代码

```text

Git clone https://github.com/saltstack/halite

```
#### 生成index.html文件

```text

cd halite/halite  
./genindex.py -C 

会产生一个index.html文件

```
 

#### 修改master配置

在行尾增加, 注意yaml文件是不接受tab的

```text
# add config for saltstack WebUI halite

rest_cherrypy:
  host: 0.0.0.0
  port: 8080
  debug: true
  static: /root/halite/halite
  app: /root/halite/halite/index.html
external_auth:
  pam:
    admin:
      - .*
#      - '@runner'
#      - '@wheel'

```

admin为系统用户， 如果不存在需要useradd admin， passwd admin，进行创建和增加密码， halite不接受root


#### 运行halite

    # cd halite/halite
    # python server_bottle.py -d -C -l debug -s gevent

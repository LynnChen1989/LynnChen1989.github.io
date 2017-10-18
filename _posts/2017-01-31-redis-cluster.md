---
layout: post
title: redis集群
tags:
    - redis
categories: Redis
description:  redis集群
---

[引言]redis从3.0开始支持cluster的模式

# redis cluster的现状

目前redis支持的cluster特性:

+ 节点自动发现
+ slave->master 选举,集群容错
+ Hot resharding:在线分片
+ 集群管理:cluster xxx
+ 基于配置(nodes-port.conf)的集群管理
+ ASK 转向/MOVED 转向机制

# redis cluster 架构

### redis-cluster架构图

![image](/images/document_images/redis_cluster1.jpg)

+ 所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽.
+ 节点的fail是通过集群中超过半数的master节点检测失效时才生效.
+ 客户端与redis节点直连,不需要中间proxy层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可
+ redis-cluster把所有的物理节点映射到[0-16383]slot上,cluster 负责维护node<->slot<->key

### redis-cluster选举:容错

![image](/images/document_images/redis_cluster2.jpg)

+ 选举过程是集群中所有master参与,如果半数以上master节点与故障节点通信超过(cluster-node-timeout),认为该节点故障，自动触发故障转移操作.
+ 什么时候整个集群不可用(cluster_state:fail)? 

    (1)如果集群任意master挂掉,且当前master没有slave.集群进入fail状态,也可以理解成集群的slot映射[0-16383]不完成时进入fail状态. 
        
    (2)如果集群超过半数以上master挂掉，无论是否有slave集群进入fail状态.
    
+ 当集群不可用时,所有对集群的操作做都不可用，收到((error) CLUSTERDOWN The cluster is down)错误
  
  
# redis cluster 实践

### 安装redis

redis的安装非常简单， 直接下载redis源码包, 执行

```text
make PREFIX=/usr/local/redis install
```

### 集群搭建

redis集群至少需要6个节点，搭建环境为： 主机：127.0.0.1， 端口: 7000~7005共6个redis实例

IP | 端口 | 角色 
---|---|---
127.0.0.1 | 7000| master
127.0.0.1 | 7001| master 
127.0.0.1 | 7002| master
127.0.0.1 | 7003| slave
127.0.0.1 | 7004| slave
127.0.0.1 | 7005| slave

##### 配置环境

redis的安装目录为/usr/local/redis

redis的cluster所在目录为/usr/local/redis/cluster

```text

root@192.168.238.128:/usr/local/redis/cluster [13:07:22] # tree .
.
├── 7000
│   ├── dump-7000.rdb
│   ├── nodes-7000.conf
│   └── redis.conf
├── 7001
│   ├── dump-7001.rdb
│   ├── nodes-7001.conf
│   └── redis.conf
├── 7002
│   ├── dump-7002.rdb
│   ├── nodes-7002.conf
│   └── redis.conf
├── 7003
│   ├── dump-7003.rdb
│   ├── nodes-7003.conf
│   └── redis.conf
├── 7004
│   ├── dump-7004.rdb
│   ├── nodes-7004.conf
│   └── redis.conf
└── 7005
    ├── dump-7005.rdb
    ├── nodes-7005.conf
    └── redis.conf

6 directories, 18 files

```

##### 配置文件

和一般的配置略有不同，就是取消注释掉如下配置所在的位置即可开启集群模式, 7000~7005每一个目录下面的redis.conf修改为对应的配置即可

```text
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 5000
```

**一个完整的参考配置如下(7000端口的配置)**

```text
bind 127.0.0.1
protected-mode yes
port 7000
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize yes
supervised no
pidfile /usr/local/redis/logs/redis-7000.pid
loglevel notice
logfile "/usr/local/redis/logs/redis-7000.log"
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump-7000.rdb
dir /usr/local/redis/cluster/7000
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes

```

**批量修改配置**

```text
# cd /usr/local/redis/cluster/7000 
# sed 's/7000/7005/' redis.conf >> ../7005/redis.conf
# sed 's/7000/7001/' redis.conf >> ../7001/redis.conf
# sed 's/7000/7002/' redis.conf >> ../7002/redis.conf
# sed 's/7000/7003/' redis.conf >> ../7003/redis.conf
# sed 's/7000/7004/' redis.conf >> ../7004/redis.conf

```

##### 启动redis

```text

/usr/local/redis/bin/redis-server /usr/local/redis/cluster/7000/redis.conf

```
7001-7005端口启动一样


##### 安装集群工具（redis-trib.rb）依赖

redis-trib.rb这个在源码包中附带了， make install的时候不会安装到对应的目录中去，最好是手动拷贝到安装目录/usr/local/redis/bin中去

前面已经准备好了搭建集群的redis节点，接下来我们要把这些节点都串连起来搭建集群。官方提供了一个工具：redis-trib.rb， 所以我们需要安装ruby工具
  
    yum -y install ruby ruby-devel rubygems rpm-build
    gem install redis    //等一会儿就好了
    
##### 创建集群
    
    /usr/local/redis/bin/redis-trib.rb --replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
    
--replicas  1  表示 自动为每一个master节点分配一个slave节点    上面有6个节点，程序会按照一定规则生成 3个master（主）3个slave(从)

创建过程中大致会打印如下的信息：

```text
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
127.0.0.1:7000
127.0.0.1:7001
127.0.0.1:7002
Adding replica 127.0.0.1:7003 to 127.0.0.1:7000
Adding replica 127.0.0.1:7004 to 127.0.0.1:7001
Adding replica 127.0.0.1:7005 to 127.0.0.1:7002
M: 42cdca32ab743d79d32a84937bd7c1c728317db5 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
M: eedd3513535e1d6a1f9d6d4992b00ab44faa936e 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
M: f0aafbcd12840cadefb378280ef3e7df22160116 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
S: 92ec476beabffccdeb3788093aa40f432460923d 127.0.0.1:7003
   replicates 42cdca32ab743d79d32a84937bd7c1c728317db5
S: 9d93fd49ceeb731deceb74043544847fe8f1129b 127.0.0.1:7004
   replicates eedd3513535e1d6a1f9d6d4992b00ab44faa936e
S: e3cf405c85dee6104c6ff58d78feda039328a289 127.0.0.1:7005
   replicates f0aafbcd12840cadefb378280ef3e7df22160116
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join...
>>> Performing Cluster Check (using node 127.0.0.1:7000)
M: 42cdca32ab743d79d32a84937bd7c1c728317db5 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
M: eedd3513535e1d6a1f9d6d4992b00ab44faa936e 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
S: e3cf405c85dee6104c6ff58d78feda039328a289 127.0.0.1:7005
   slots: (0 slots) slave
   replicates f0aafbcd12840cadefb378280ef3e7df22160116
S: 9d93fd49ceeb731deceb74043544847fe8f1129b 127.0.0.1:7004
   slots: (0 slots) slave
   replicates eedd3513535e1d6a1f9d6d4992b00ab44faa936e
S: 92ec476beabffccdeb3788093aa40f432460923d 127.0.0.1:7003
   slots: (0 slots) slave
   replicates 42cdca32ab743d79d32a84937bd7c1c728317db5
M: f0aafbcd12840cadefb378280ef3e7df22160116 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

至此，集群的创建就基本完毕。


### python API使用举例

```python

#!/usr/bin/env python
#coding:utf-8

from rediscluster import StrictRedisCluster
import sys

def redis_cluster():
    redis_nodes = [{"host": "127.0.0.1", "port": i} for i in xrange(7000, 7005)]
    try:
        redis_conn = StrictRedisCluster(startup_nodes=redis_nodes)
    except Exception, e:
        print "Connect Error!"
        sys.exit(1)

    redis_conn.set('name','admin')
    redis_conn.set('age',18)
    print "name is: ", redis_conn.get('name')
    print "age  is: ", redis_conn.get('age')

redis_cluster()

```

# redis cluster 管理

### 增加节点

##### 新启动节点7006
    
    /usr/local/redis/bin/redis-server /usr/local/redis/cluster/7006/redis.conf
    
##### 把新节点加入集群
    
    /usr/local/redis/bin/redis-trib.rb add-node 127.0.0.1:7006 127.0.0.1:7000
 
新节点没有包含任何数据， 因为它没有包含任何slot。新加入的加点是一个主节点， 当集群需要将某个从节点升级为新的主节点时， 这个新节点不会被选中
    
##### 为新节点分配slot 

    /usr/local/redis/bin/redis-trib.rb reshard 127.0.0.1:7006

##### 如果是新增加的节点为slave, 添加--slave参数即可

    /usr/local/redis/bin/redis-trib.rb add-node --slave 127.0.0.1:7006 127.0.0.1:7000
    
注意这种方式，如果添加了多个slave节点，可能导致master的slaves不均衡，比如一些有3个slave，其它只1个slave。

可以在slave节点上执行redis命令“CLUSTER REPLICATE”进行调整，让它成为其它master的slave。

“CLUSTER REPLICATE”带一个参数，即master ID，注意使用redis-cli -c登录到slave上执行。
 
在线添加slave 时，需要dump整个master进程，并传递到slave，再由 slave加载rdb文件到内存，rdb传输过程中Master可能无法提供服务,整个过程消耗大量io,小心操作。
    
##### 明确master

上面方法没有指定7006的master，而是随机指定。下面方法可以明确指定为哪个master的slave

    /usr/local/redis/bin/redis-trib.rb add-node --slave --master-id 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 127.0.0.1:7006 127.0.0.1:7000
    
    
### 删除slave节点

    /usr/local/redis/bin/redis-trib del-node 127.0.0.1:7000 ``
    
第一个参数为集群中任意一个节点，第二个参数为需要删除节点的ID。

### 删除master节点

删除master节点之前首先要使用reshard移除master的全部slot,然后再删除当前节点(目前只能把被删除 master的slot迁移到一个节点上)


### 检查节点状态

    /usr/local/redis/bin/redis-trib.rb check 127.0.0.1:7000
    
    
### 变更主从关系

使用命令cluster replicate，参数为master节点ID，注意不是IP和端口，在被迁移的slave上执行该命令。

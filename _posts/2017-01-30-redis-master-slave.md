---
layout: post
title: redis主从复制和故障转移
tags:
    - redis
categories: Redis
description:  redis主从复制和故障转移
---

[引言] 本文介绍redis主从复制基本原理和搭建以及通过sentinel进行故障转移

# redis主从复制

#### 复制原理简介

1. 如果设置了一个Slave，无论是第一次连接还是重连到Master，它都会发出一个SYNC命令；

2. 当Master收到SYNC命令之后，会做两件事：
   
   (1) Master执行BGSAVE，即在后台保存数据到磁盘（rdb快照文件）；
   
   (2) Master同时将新收到的写入和修改数据集的命令存入缓冲区（非查询类）；

3. 当Master在后台把数据保存到快照文件完成之后，Master会把这个快照文件传送给Slave，而Slave则把内存清空后，加载该文件到内存中；

4. 而Master也会把此前收集到缓冲区中的命令，通过Reids命令协议形式转发给Slave，Slave执行这些命令，实现和Master的同步；

5. Master/Slave此后会不断通过异步方式进行命令的同步，达到最终数据的同步一致；

6. 需要注意的是Master和Slave之间一旦发生重连都会引发全量同步操作。但在2.8之后版本，也可能是部分同步操作。

**部分复制**

2.8开始，当Master和Slave之间的连接断开之后，他们之间可以采用持续复制处理方式代替采用全量同步。

Master端为复制流维护一个内存缓冲区（in-memory backlog），记录最近发送的复制流命令；

同时，Master和Slave之间都维护一个复制偏移量(replication offset)和当前Master服务器ID（Master run id）。

当网络断开，Slave尝试重连时：

+ 如果MasterID相同（即仍是断网前的Master服务器），并且从断开时到当前时刻的历史命令依然在Master的内存缓冲区中存在，则Master会将缺失的这段时间的所有命令发送给Slave执行，然后复制工作就可以继续执行了；

+ 否则，依然需要全量复制操作；

Redis 2.8 的这个部分重同步特性会用到一个新增的 PSYNC 内部命令， 而 Redis 2.8 以前的旧版本只有 SYNC 命令， 不过， 只要从服务器是 Redis 2.8 或以上的版本， 它就会根据主服务器的版本来决定到底是使用 PSYNC 还是 SYNC ：

+ 如果主服务器是 Redis 2.8 或以上版本，那么从服务器使用 PSYNC 命令来进行同步。

+ 如果主服务器是 Redis 2.8 之前的版本，那么从服务器使用 SYNC 命令来进行同步


#### 开启复制

只需要添加一条配置文件即可开启redis的主从复制， 格式 slaveof \<master ip\> \<master port\>

```text
slaveof  127.0.0.1  6379
```

写入了本条配置文件的redis即为slave， 127.0.0.1:6379则为master.

#### redis简单配置

redis的配置文件包含在了redis的源码安装包中了， 只需要从源码包中简单的复制一个进行修改即可， 下面对几个语义不是很明确和常用的需要改动的参数做一个简要的解释。

```
bind 127.0.0.1  #绑定ip
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 0
daemonize yes  #后台运行
supervised no
pidfile "/usr/local/redis/logs/redis-6379.pid"
loglevel notice
logfile "/usr/local/redis/logs/redis-6379.logs"
databases 16  #可用数据库
save 900 1    #指出在多长时间内，有多少次更新操作，就将数据同步到数据文件rdb
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename "dump_6379.rdb"   #本地持久化数据库文件名
dir "/usr/local/redis/data"  #工作目录,dump_6379.rdb将存储在该目录下
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
slaveof 127.0.0.1 6381   #开启主从模式，拥有当前配置的redis为slave，127.0.0.1:6381为master

```

# 配置主从复制的故障转移（redis-sentinel）

### redis-sentinel原理

+ 启动 n 个 sentinel 实例，这些 sentinel 实例会去监控你指定的 redis master/slaves
+ 当 redis master 节点挂掉后， Sentinel 实例通过 ping 检测失败发现这种情况就认为该节点进入 SDOWN 状态，也就是检测的 sentinel 实例主观地（Subjectively）认为该 redis master 节点挂掉。
+ 当一定数目(Quorum 参数设定）的 Sentinel 实例都认为该 master 挂掉的情况下，该节点将转换进入 ODOWN 状态，也就是客观地（Objectively）挂掉的状态。
+ 接下来 sentinel 实例之间发起选举，选择其中一个 sentinel 实例发起 failover 过程：从 slave 中选择一台作为新的 master，让其他 slave 从新的 master 复制数据，并通过 Pub/Sub 发布事件。
+ 使用者客户端从任意 Sentinel 实例获取 redis 配置信息，并监听（可选） Sentinel 发出的事件： SDOWN, ODOWN 以及 failover 等，并做相应主从切换，Sentinel 还扮演了服务发现的角色。
+ Sentinel 的 Leader 选举采用的是 Raft 协议。

### 基础环境

由于是在测试环境中，所以我的redis主从和sentinel都在同一台主机，具体生产环境安装需求进行改变

+ bin目录，是redis的二进制工具
+ logs目录，存放日志文件和pid文件
+ data目录，存放redis持久化数据文件

具体的目录树结构如下：

```text

redis/
├── bin
│   ├── redis-benchmark
│   ├── redis-check-aof
│   ├── redis-check-rdb
│   ├── redis-cli
│   ├── redis-sentinel -> redis-server
│   └── redis-server
├── data
│   ├── dump_6379.rdb
│   ├── dump_6380.rdb
│   ├── dump_6381.rdb
│   └── dump_6382.rdb
├── logs
│   ├── redis-6379.logs
│   ├── redis-6379.pid
│   ├── redis-6380.logs
│   ├── redis-6381.logs
│   ├── redis-6381.pid
│   ├── redis-6382.logs
│   ├── redis-6382.pid
│   ├── redis-sentinel-26379.log
│   ├── redis-sentinel-26379.pid
│   ├── redis-sentinel-26380.log
│   ├── redis-sentinel-26380.pid
│   ├── redis-sentinel-26381.log
│   ├── redis-sentinel-26381.pid
│   └── redis-sentinel.log
├── redis-6379.conf
├── redis-6380.conf
├── redis-6381.conf
├── redis-6382.conf
├── sentinel-26379.conf
├── sentinel-26380.conf
└── sentinel-26381.conf
```

IP | 端口 | 角色 
---|---|---
127.0.0.1 | 6379| master
127.0.0.1 | 6380| slave1 
127.0.0.1 | 6381| slave2
127.0.0.1 | 6382| slave3
127.0.0.1 | 26379| sentinel1
127.0.0.1 | 26380| sentinel2
127.0.0.1 | 26381| sentinel3


### 主从搭建

主从为一主三从， 6379为master， 6380、6381、6382为slave, 搭建的过程非常简单，就不具体说明。

#### 验证

在master/slave上通过命令`info replication`命令查看主从信息

##### master(6379)

```text
# /usr/local/redis/bin/redis-cli -h 127.0.0.1 -p 6379
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:3
slave0:ip=127.0.0.1,port=6382,state=online,offset=437038,lag=1
slave1:ip=127.0.0.1,port=6381,state=online,offset=437038,lag=1
slave2:ip=127.0.0.1,port=6380,state=online,offset=437171,lag=1
master_repl_offset:437171
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:437170
127.0.0.1:6379> 

```

##### slave(6380),其他端口信息类似

```text
# /usr/local/redis/bin/redis-cli -h 127.0.0.1 -p 6380
127.0.0.1:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6381
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_repl_offset:434763
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
127.0.0.1:6380> 

```

##### 启动redis

```text
/usr/local/redis/bin/redis-server /usr/local/redis/redis-6380.conf 
```

### sentinel哨兵集群

##### 简介

Redis 的 Sentinel 系统用于管理多个 Redis 服务器（instance）， 该系统执行以下三个任务：

**监控（Monitoring）：** Sentinel 会不断地检查你的主服务器和从服务器是否运作正常。

**提醒（Notification）：** 当被监控的某个 Redis 服务器出现问题时， Sentinel 可以通过 API 向管理员或者其他应用程序发送通知。

**自动故障迁移（Automatic failover）：**  当一个主服务器不能正常工作时， Sentinel 会开始一次自动故障迁移操作， 它会将失效主服务器的其中一个从服务器升级为新的主服务器， 并让失效主服务器的其他从服务器改为复制新的主服务器； 

当客户端试图连接失效的主服务器时， 集群也会向客户端返回新主服务器的地址， 使得集群可以使用新主服务器代替失效服务器。

Redis Sentinel 是一个分布式系统， 你可以在一个架构中运行多个 Sentinel 进程（progress）， 这些进程使用流言协议（gossip protocols)来接收关于主服务器是否下线的信息， 并使用投票协议（agreement protocols）来决定是否执行自动故障迁移， 以及选择哪个从服务器作为新的主服务器。

虽然 Redis Sentinel 释出为一个单独的可执行文件 redis-sentinel ， 但实际上它只是一个运行在特殊模式下的 Redis 服务器， 你可以在启动一个普通 Redis 服务器时通过给定 --sentinel 选项来启动 Redis Sentinel 。

##### 了解主观和客观下线

前面说过， Redis 的 Sentinel 中关于下线（down）有两个不同的概念：

+ 主观下线（Subjectively Down， 简称 SDOWN）指的是单个 Sentinel 实例对服务器做出的下线判断。
+ 客观下线（Objectively Down， 简称 ODOWN）指的是多个 Sentinel 实例在对同一个服务器做出 SDOWN 判断， 并且通过 SENTINELis-master-down-by-addr 命令互相交流之后， 得出的服务器下线判断。 （一个 Sentinel 可以通过向另一个 Sentinel 发送SENTINEL is-master-down-by-addr 命令来询问对方是否认为给定的服务器已下线。）

如果一个服务器没有在 master-down-after-milliseconds 选项所指定的时间内， 对向它发送 PING 命令的 Sentinel 返回一个有效回复（valid reply）， 那么 Sentinel 就会将这个服务器标记为主观下线。

服务器对 PING 命令的有效回复可以是以下三种回复的其中一种：

+ 返回 +PONG 。
+ 返回 -LOADING 错误。
+ 返回 -MASTERDOWN 错误。

如果服务器返回除以上三种回复之外的其他回复， 又或者在指定时间内没有回复 PING 命令， 那么 Sentinel 认为服务器返回的回复无效（non-valid）。

注意， 一个服务器必须在 master-down-after-milliseconds 毫秒内， 一直返回无效回复才会被 Sentinel 标记为主观下线。

举个例子， 如果 master-down-after-milliseconds 选项的值为 30000 毫秒（30 秒）， 那么只要服务器能在每 29 秒之内返回至少一次有效回复， 这个服务器就仍然会被认为是处于正常状态的。

从主观下线状态切换到客观下线状态并没有使用严格的法定人数算法（strong quorum algorithm）， 而是使用了流言协议： 如果 Sentinel 在给定的时间范围内， 从其他 Sentinel 那里接收到了足够数量的主服务器下线报告， 那么 Sentinel 就会将主服务器的状态从主观下线改变为客观下线。 如果之后其他 Sentinel 不再报告主服务器已下线， 那么客观下线状态就会被移除。

客观下线条件只适用于主服务器： 对于任何其他类型的 Redis 实例， Sentinel 在将它们判断为下线前不需要进行协商， 所以从服务器或者其他 Sentinel 永远不会达到客观下线条件。

只要一个 Sentinel 发现某个主服务器进入了客观下线状态， 这个 Sentinel 就可能会被其他 Sentinel 推选出， 并对失效的主服务器执行自动故障迁移操作。


##### 配置文件修改

哨兵26379的配置文件示例， 26380,26381的配置类似, 在实际的生产环境中不建议只通过一个哨兵来决定是否需要failover，至少需要两到三个，测试为三个。‘

```
port 26379
dir "/tmp" #Sentinel服务运行时使用的临时文件夹
sentinel myid 9cde85c41b79b10c131b99373bad74595b611ea2  # 这个id每一个sentinel必须唯一，否则不报错，但是也不成功
sentinel monitor mymaster 127.0.0.1 6381 2  #Sentinel去监视一个名为mymaster的主redis实例，这个主实例的IP地址为本机地址127.0.0.1，端口号为6379，而将这个主实例判断为失效至少需要2个 Sentinel进程的同意，只要同意Sentinel的数量不达标，自动failover就不会执行
sentinel down-after-milliseconds mymaster 5000  #指定了Sentinel认为Redis实例已经失效所需的毫秒数
sentinel failover-timeout mymaster 5000 #如果在该时间（ms）内未能完成failover操作，则认为该failover失败
daemonize yes   #后台模式
logfile "/usr/local/redis/logs/redis-sentinel-26379.log"  # 日志文件
pidfile "/usr/local/redis/logs/redis-sentinel-26379.pid"  # pid文件


# 下面的这些是在实际的运行过程中自动产生的
sentinel config-epoch mymaster 3
sentinel leader-epoch mymaster 3
sentinel known-slave mymaster 127.0.0.1 6380
sentinel known-slave mymaster 127.0.0.1 6382
sentinel known-slave mymaster 127.0.0.1 6381
sentinel known-sentinel mymaster 127.0.0.1 26380 9cde85c51b79b10c131b99373bad74595b611ea2
sentinel known-sentinel mymaster 127.0.0.1 26381 9cde85c41b79b10c131b99373bad74595b614ea2
sentinel current-epoch 3

```

##### 启动哨兵

```text
/usr/local/redis/bin/redis-sentinel /usr/local/redis/sentinel-26380.conf --sentinel
```

##### 通过哨兵info检查

```text
# /usr/local/redis/bin/redis-cli -h 127.0.0.1 -p 26379
127.0.0.1:26379> info Sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=127.0.0.1:6382,slaves=3,sentinels=3
```
可以看到，当前master的名称为mymaster, 状态ok, 有3个slave，3个sentinel


##### 进程检查

所有的都搭建好了之后，简单检查下redis和哨兵的进程运行情况

```text
root@192.168.238.128:~ [13:46:57] # ps -ef|grep redis|grep -v grep
root      36960      1  0 12:34 ?        00:00:25 /usr/local/redis/bin/redis-server 127.0.0.1:6382
root      43194      1  0 13:24 ?        00:00:07 /usr/local/redis/bin/redis-server 127.0.0.1:6380
root      43525      1  0 13:26 ?        00:00:09 /usr/local/redis/bin/redis-sentinel *:26381 [sentinel]
root      43529      1  0 13:26 ?        00:00:09 /usr/local/redis/bin/redis-sentinel *:26380 [sentinel]
root      43533      1  0 13:26 ?        00:00:09 /usr/local/redis/bin/redis-sentinel *:26379 [sentinel]
root      75517      1  0 00:13 ?        00:04:17 /usr/local/redis/bin/redis-server 127.0.0.1:6379
root     121771      1  0 Jan30 ?        00:06:00 /usr/local/redis/bin/redis-server 127.0.0.1:6381

```


### 验证故障转移

当前的主从情况是， 6379为master， 6380、6381、6382三个端口为slave的一主三从的结构。

##### 手动停掉master

```text
/usr/local/redis/bin/redis-cli -h 127.0.0.1 -p 6379 shutdown
```

##### sentinel的日志输出

```text
43533:X 31 Jan 13:51:17.485 # +sdown master mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:17.575 # +odown master mymaster 127.0.0.1 6380 #quorum 2/2
43533:X 31 Jan 13:51:17.575 # +new-epoch 5
43533:X 31 Jan 13:51:17.575 # +try-failover master mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:17.582 # +vote-for-leader 9cde85c41b79b10c131b99373bad74595b611ea2 5
43533:X 31 Jan 13:51:17.587 # 9cde85c51b79b10c131b99373bad74595b611ea2 voted for 9cde85c41b79b10c131b99373bad74595b611ea2 5
43533:X 31 Jan 13:51:17.587 # 9cde85c41b79b10c131b99373bad74595b614ea2 voted for 9cde85c41b79b10c131b99373bad74595b611ea2 5
43533:X 31 Jan 13:51:17.649 # +elected-leader master mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:17.649 # +failover-state-select-slave master mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:17.716 # +selected-slave slave 127.0.0.1:6382 127.0.0.1 6382 @ mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:17.716 * +failover-state-send-slaveof-noone slave 127.0.0.1:6382 127.0.0.1 6382 @ mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:18.649 * +failover-state-wait-promotion slave 127.0.0.1:6382 127.0.0.1 6382 @ mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:18.657 # +promoted-slave slave 127.0.0.1:6382 127.0.0.1 6382 @ mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:18.657 # +failover-state-reconf-slaves master mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:18.702 * +slave-reconf-sent slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:19.698 * +slave-reconf-inprog slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:19.698 * +slave-reconf-done slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:19.782 # -odown master mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:19.782 # +failover-end master mymaster 127.0.0.1 6380
43533:X 31 Jan 13:51:19.782 # +switch-master mymaster 127.0.0.1 6380 127.0.0.1 6382
43533:X 31 Jan 13:51:19.783 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6382
43533:X 31 Jan 13:51:19.783 * +slave slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6382
43533:X 31 Jan 13:51:19.783 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6382
43533:X 31 Jan 13:51:24.837 # +sdown slave 127.0.0.1:6381 127.0.0.1 6381 @ mymaster 127.0.0.1 6382
43533:X 31 Jan 13:51:24.840 # +sdown slave 127.0.0.1:6380 127.0.0.1 6380 @ mymaster 127.0.0.1 6382
```

##### 验证切换

随便进入一个slave查看当前的主从信息，会发现master已经切换到了6381端口，6380,6382为slave, 也可以通过sentinel的info sentinel命令进行查看

### sentinel常用操作

除了上面提到的一些查看信息的命令之外， sentinel 还支持下列命令来管理和检测 sentinel 配置：

    SENTINEL reset <pattern> 强制重设所有监控的 master 状态，清除已知的 slave 和 sentinel 实例信息，重新获取并生成配置文件。
    SENTINEL failover <master name> 强制发起一次某个 master 的 failover，如果该 master 不可访问的话。
    SENTINEL ckquorum <master name> 检测 sentinel 配置是否合理， failover 的条件是否可能满足，主要用来检测你的 sentinel 配置是否正常。
    SENTINEL flushconfig 强制 sentinel 重写所有配置信息到配置文件。

增加和移除监控以及修改配置参数：

    SENTINEL MONITOR <name> <ip> <port> <quorum>
    SENTINEL REMOVE <name>
    SENTINEL SET <name> <option> <value>

### 增加和移除 Sentinel

增加新的 Sentinel 实例非常简单，修改好配置文件，启动即可，其他 Sentinel 会自动发现该实例并加入集群。如果要批量启动一批 Sentinel 节点，最好以 30 秒的间隔一个一个启动为好，这样能确保整个 Sentinel 集群的大多数能够及时感知到新节点，满足当时可能发生的选举条件。

移除一个 sentinel 实例会相对麻烦一些，因为 sentinel 不会忘记已经感知到的 sentinel 实例，所以最好按照下列步骤来处理：

    停止将要移除的 sentinel 进程。
    给其余的 sentinel 进程发送 SENTINEL RESET * 命令来重置状态，忘记将要移除的 sentinel，每个进程之间间隔 30 秒。
    确保所有 sentinel 对于当前存货的 sentinel 数量达成一致，可以通过 SENTINEL MASTER [mastername] 命令来观察，或者查看配置文件。

### 生产环境推荐

对于一个最小集群，Redis 应该是一个 master 带上两个 slave，并且开启下列选项：

    min-slaves-to-write 1
    min-slaves-max-lag 10

这样能保证写入 master 的同时至少写入一个 slave，如果出现网络分区阻隔并发生 failover 的时候，可以保证写入的数据最终一致而不是丢失，写入老的 master 会直接失败，参考 Consistency under partitions。

Slave 可以适当设置优先级，除了 0 之外（0 表示永远不提升为 master），越小的优先级，越有可能被提示为 master。

如果 slave 分布在多个机房，可以考虑将和 master 同一个机房的 slave 的优先级设置的更低以提升他被选为新的 master 的可能性。

考虑到可用性和选举的需要，Sentinel 进程至少为 3 个，推荐为 5 个，如果有网络分区，应当适当分布（比如 2 个在 A 机房， 2 个在 B 机房，一个在 C 机房）等。


# 配置集群(cluster)的故障转移（redis-sentinel)
---
layout: post
title: Percona-toolkit的使用
tags:
    - 数据库
categories: 数据库
description:  percona-tookit，是一款数据库相关的工具条件，提供众多的DBA常用的工具，如数据比对，在线schema修改等。
---

## 安装

#### 依赖

```
yum install perl perl-devel perl-Time-HiRes perl-DBI perl-DBD-MySQL perl-Digest-MD5

```

#### 直接下载通用二进制包

```
# wget https://www.percona.com/downloads/percona-toolkit/3.0.4/binary/tarball/percona-toolkit-3.0.4_x86_64.tar.gz

# tar zxf percona-toolkit-3.0.4_x86_64.tar.gz
# mv percona-toolkit-3.0.4 /usr/local/percona
# export PATH=$PATH:/usr/local/percona/bin
```

## 常用举例

### 查询db信息(pt-mysql-summary)

```
pt-mysql-summary -- --user=root --password=p\!Opb5otdcbo5Pmybazk --host=172.31.6.45
```

### 数据一致性对比(pt-table-checksum)

https://tyoung.me/2016/09/usage_of_pt-table-checksum_and_replication_consistency_check/

#### 参数解释

```
--nocheck-replication-filters ：不检查复制过滤器，建议启用。后面可以用--databases来指定需要检查的数据库。
--no-check-binlog-format      : 不检查复制的binlog模式，要是binlog模式是ROW，则会报错。
--replicate-check-only :只显示不同步的信息。
--replicate=   ：把checksum的信息写入到指定表中，建议直接写到被检查的数据库当中。 
--databases=   ：指定需要被检查的数据库，多个则用逗号隔开。
--tables=      ：指定需要被检查的表，多个用逗号隔开
h=127.0.0.1    ：Master的地址
u=root         ：用户名
p=123456       ：密码
P=3306         ：端口
```

#### 场景一，非主从
##### 在MySQL A上执行

```
pt-table-checksum  --nocheck-replication-filters --no-check-binlog-format  --databases=110005_doomsday_game h='172.31.13.9',u=root,P=3306,p=p\!Opb5otdcbo5Pmybazk
```

##### 在MySQL B上执行

```
pt-table-checksum  --nocheck-replication-filters --no-check-binlog-format --databases=110005_doomsday_game h='172.31.13.9',u=root,P=3306,p=p\!Opb5otdcbo5Pmybazk
```

以上是一些常规场景的使用，得出结果后请人肉对比，对应主从场景，会直接得出比较结果，不需要人肉check。

#### 场景二，主从

如果是有主从关系，只需要在master执行即可。

```
pt-table-checksum h=172.31.6.45,u=root,P=3306,p=p\!Opb5otdcbo5Pmybazk -d 110005_doomsday_game  --nocheck-replication-filters --nocheck-binlog-format --nocheck-plan --recursion-method=hosts
```

**说明**
--recursion-method 是master发现slave的方式，默认有两种。

+ hosts, 通过`"SHOW SLAVE HOSTS"`命令，这个需要slave在启动参数添加'report-host=xxxx', 不然无法找到slave.
+ processlist, 通过`SHOW PROCESSLIST`命令发现slave。

其他的方式可以查文档。


输出结果：

```
            TS ERRORS  DIFFS     ROWS  CHUNKS SKIPPED    TIME TABLE
10-18T07:37:01      0      0        1       1       0   0.012 110005_doomsday_game.activity_data
10-18T07:37:01      0      0        1       1       0   0.013 110005_doomsday_game.alliance_boss_data
10-18T07:37:01      0      0        9       1       0   0.016 110005_doomsday_game.alliance_data
10-18T07:37:01      0      0       16       1       0   0.012 110005_doomsday_game.checksums
10-18T07:37:02      0      0    13502       4       0   0.093 110005_doomsday_game.npc_data
10-18T07:37:02      0      0    22209       1       0   0.436 110005_doomsday_game.player_data
10-18T07:37:02      0      0        0       1       0   0.012 110005_doomsday_game.recharge_data
10-18T07:37:02      0      0        1       1       0   0.011 110005_doomsday_game.server_exchange_data
10-18T07:37:02      0      0        1       1       0   0.012 110005_doomsday_game.server_info
10-18T07:37:02      0      0       62       1       0   0.016 110005_doomsday_game.system_email
10-18T07:37:02      0      0        1       1       0   0.016 110005_doomsday_game.t2
10-18T07:37:02      0      0        1       1       0   0.012 110005_doomsday_game.world_boss_data
10-18T07:37:02      0      0        1       1       0   0.011 110005_doomsday_game.world_secretshop_data
10-18T07:37:02      0      0        1       1       0   0.011 110005_doomsday_game.world_state_data
```

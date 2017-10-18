---
layout: post
title: Ec2磁盘离线扩容
tags:
    - 扩容
    - Ec2
    - aws
categories: 亚马逊Aws
description: Ec2磁盘离线扩容
---


[引言] aws的磁盘容量达到了瓶颈，但是无法通过简单增加磁盘来解决，只有进行离线扩容，假如需要进行扩容的磁盘为/dev/xvdb，对应的分区为/dev/xvdb1

#### 扩容前现状
```
# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda2       99G   41G   53G  44% /
tmpfs           7.3G     0  7.3G   0% /dev/shm
/dev/xvda1      194M   28M  157M  15% /boot
/dev/xvdb1      1024G  748G 226G  77% /home/worker/data/mysql

# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda    202:0    0  100G  0 disk 
├─xvda1 202:1    0  200M  0 part /boot
└─xvda2 202:2    0 99.8G  0 part /
xvdb    202:16   0    1T  0 disk 
└─xvdb1 202:17   0    1T  0 part /home/worker/data/mysql
```

存在一块磁盘/dev/xvdb大小为1T,一个分区/dev/xvdb1大小为1T

现在需要将/dev/xvdb1分区空间直接扩大到2T，而不是选择增加磁盘(现状不允许)的方式.

#### 扩容步骤

##### 1、对需要扩容的EC2实例的对应数据卷做快照（snap-05fd663f8e9322cdb）
- 这一步可以提前做，因为首次做快照是全量快照，1T的数据盘需要很长时间，这一步是在线做快照


##### 2、停止EC2实例
- 这一步直接在控制台操作，注意是停止，不是终止
    
##### 3、分离当前/dev/xvdb1对应的数据卷（vol-026993fc9d3495f35）
- 在aws控制台操作

##### 4、对分离的数据卷进行二次做快照(snap-6d3b20e8)

- 在aws控制台操作，这一次是增量快照，应该会很快

##### 5、从最新的快照(snap-6d3b20e8)生成数据卷(vol-06f04817ffd8bb512)
- 这一步就是真正的扩容， 注意生成的数据卷大小为2T
- 注意这一步要注意，如果后续需要建立msdos分区表，那么最大不能超过2T, 超过2T的磁盘只能建立gpt分区表， 这一步要想好，避免从头再来，如果不知道当前分区表类型可以通过下面的方式查看.

```
# sudo parted /dev/xvdb
# (parted) print
```

##### 6、挂载数据卷（vol-06f04817ffd8bb512）到EC2
- 挂在新生成的数据卷到EC2, **注意一定要和刚才的盘符一致**，不然fstab中无对应记录的话，会导致无法通过自检而启动失败

##### 7、启动EC2


#### 启动后的操作

##### 1、查看当前磁盘和分区情况

可以看到的是df -h还是原来那样，因为磁盘虽然扩容了，但是文件系统并未扩容

```
# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda    202:0    0  100G  0 disk 
├─xvda1 202:1    0  200M  0 part /boot
└─xvda2 202:2    0 99.8G  0 part /
xvdb    202:16   0    2T  0 disk 
└─xvdb1 202:17   0    1T  0 part /home/worker/data/mysql
```

lsblk可以看到磁盘空间已经2T, 但是文件系统还是1T


##### 2、卸载被自动挂载上来的分区

```
# sudo umount /home/worker/data/mysql

如果出现device is busy的错误

# fuser -m -v -i -k /home/worker/data/mysql

```


##### 3、对设备（而不是设备上的分区）运行 parted 命令

```
# sudo parted /dev/xvdb
GNU Parted 2.1
Using /dev/xvdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
```

##### 4、将 parted 度量单位更改为扇区

```
(parted) unit s
```

##### 5、运行 print 命令，列出设备上的分区。对于某些分区表类型，系统可能会提示您针对较大卷大小修复分区表。对所有关于是否修复现有分区表的问题回答"Ignore"；之后开始创建新表。

```
(parted) print
```

以下是 gpt 分区表示例

```

Model: Xen Virtual Block Device (xvd)
Disk /dev/xvdb: 209715200s
Sector size (logical/physical): 512B/512B
Partition Table: gpt

Number  Start  End        Size       File system  Name                 Flags
128     2048s  4095s      2048s                   BIOS Boot Partition  bios_grub
 1      4096s  16777182s  16773087s  ext4         Linux
```

以下是 msdos 分区表示例

```
Model: Xen Virtual Block Device (xvd)
Disk /dev/xvdb: 104857600s
Sector size (logical/physical): 512B/512B
Partition Table: msdos

Number  Start  End        Size       Type     File system  Flags
 1      2048s  35649535s  35647488s  primary  ext3
```

##### 6、使用来自上一步的编号 (1) 删除分区的分区条目

```
(parted) rm 1
```

##### 7、创建扩展到卷末尾的新分区

（对于 gpt 分区表示例）记下上面的分区 1 的起点和名称。对于 gpt 示例，起点为 4096s，名称为 Linux。使用分区 1 的起点、名称和 100% 运行 mkpart 命令，以使用所有可用空间。

```
(parted) mkpart Linux 4096s 100%
```

（对于 msdos 分区表示例）记下上面的分区 1 的起点和分区类型。对于 msdos 示例，起点为 2048s，分区类型为 primary。使用主分区类型、分区 1 的起点和 100% 运行 mkpart 命令，以使用所有可用空间。

```
(parted) mkpart primary 2048s 100%
```

##### 8、再次运行 print 命令以验证分区

对于扩展的分区，检查以前存在的任何标志是否仍存在。有时 boot 标志可能会丢失。如果在扩展时分区丢弃了标志，请使用以下命令添加标志（替换为您的分区编号和标志名称）。例如，以下命令将 boot 标志添加到分区 1。

```
(parted) set 1 boot on
```

##### 9、可以再次运行 print 命令以验证更改

##### 10、运行 quit 命令，退出 parted

```
(parted) quit
```

###### Note
由于您删除了一个分区并添加了一个分区，因此parted 会警告您需要更新 /etc/fstab。仅当分区编号更改时，才需要此操作。

##### 11、检查文件系统以确保没有错误（需要先执行此操作，然后才能扩展文件系统）。记录上一个 print 命令中的文件系统类型。根据您的文件系统类型选择以下命令之一；如果使用其他文件系统，请参阅该文件系统的文档以确定正确的检查命令。

对于 ext3 或 ext4 文件系统

```
# sudo e2fsck -f /dev/xvdb1
e2fsck 1.42.3 (14-May-2012)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/: 31568/524288 files (0.4% non-contiguous), 266685/2096635 blocks
```

##### 12、重新挂载上去

```
# sudo mount /dev/xvdb1 /home/worker/data/mysql
```

##### 13、可使用特定于文件系统的命令根据新的卷大小调整文件系统大小

```
# sudo resize2fs /dev/xvdb1
resize2fs 1.42.3 (14-May-2012)
Filesystem at /dev/xvda1 is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 5
Performing an on-line resize of /dev/xvda1 to 18350080 (4k) blocks.
The filesystem on /dev/xvda1 is now 18350080 blocks long.
```


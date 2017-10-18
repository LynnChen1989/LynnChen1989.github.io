---
layout: post
title: sudo精准用户授权
tags:
    - Linux
    - shell
categories: Linux系统
description:  使用sudo进行精准用户授权
---

[引言]visudo是直接操作/etc/sudoers文件，我们也可以直接 vi /etc/sudoers，但是visudo命令的好处在于，退出/etc/sudoers文件时，系统会检查/etc/sudoers语法是否正确。

一旦/etc/sudoers的配置没被校验而强制更新了之后，带来的后果是灾难性的的，直接导致root相关的权限无法被执行，只有通过系统recovery模式来进行恢复。

### /etc/sudoers 强制更新错误后的拯救方法

-   重启ubuntu，启动时按Esc或Shift键，可以看到引导选项；
-   在引导选项中选择Recovery模式的那一项来引导；
-   进入Recovery Menu页面，选择root，也就是进入试用root用户进行系统恢复，在这里可以执行超级用户的权限的操作，回车后可以看到熟悉的 root@user ~# 命令提示符；
-   mount -o remount rw /， 要重新挂载根目录， 才能修改/etc/sudoers文件，修改完后保存重启；

## sudo详解

##### 语法解释

root | ALL=(ALL) | ALL
---| ---| ---
使用者账号 | 登入者的来源主机名=(可切换的身份) | 可下达的指令


##### 详细解释

使用者帐号 | 来源主机名称 |  可切换的身份 |  可下达的指令
---| ---| --- | ---
代表哪个用户使用sudo的权限 |指定信任用户，即根据w查看[使用者帐号]的来源主机 | 代表可切换的用户角色，和sudo -u结合使用，默认是切换为root.| 用于权限设置，也可使用!来表示不可执行的命令


##### 举例
```text
    snake ALL=(root) !/usr/bin/passwd,!/usr/bin/passwd root
    #允许yang2用户通过sudo来修改所有其它用户的密码，但不能修改root的密码
```


## 群组授权

```text
    %testgroup ALL=(ALL) ALL
    # 在最左边加上 % ，代表后面接的是一个群组,格式和单用户授权一样
```
    
##### 无密码sudo
```text    
    %wheel ALL=(ALL) NOPASSWD: ALL
```

## 别名授权

公司有很多部门，要方便管理，就可使用别名的方式，如：开发部，运维部，技术支持部，每个部门里多个人，部门之间拥有的命令权限也不一样，同一部门权限一样。如果一条一条写的话，写也麻烦，改就更麻烦了。

使用方法可通过 man sudoers后面的举例找到

root |ALL= | (ALL) |ALL
---| ---| ---| ---
使用者账号|登入者的来源主机名|可切换的身份|可下达的指令
User_Alias FULLTIMERS = millert, mikef, dowdy|Host_Alias CSNETS = 128.138.243.0, 128.138.204.0/24, 128.138.242.0|Runas_Alias OP = root, operator|Cmnd_Alias PRINTING = /usr/sbin/lpc, /usr/bin/lprm

##### 举例
    User_Alias ADMPW = pro1,pro2, pro3, myuser1, myuser2  #配置用户别名ADMPW
    Cmnd_Alias ADMPWCOM =!/usr/bin/passwd, /usr/bin/passwd [A-Za-z]*,!/usr/bin/passwd root  #配置命令别名ADMPWCOM
    
    ADMPW ALL=(root) ADMPWCOM  #指定用户别名里的成员，拥有命令别名里的权限


## sudo记录到syslog
新建一个log文件

    touch /var/log/sudo.log
    
修改rsyslog.conf,  vi rsyslog.conf

    #在最后一行增加  
    local2.debug    /var/log/sudo.log

修改vi /etc/sudoers
    
    #在最后一行增加
    Defaults    logfile=/var/log/sudo.log

重启rsyslog服务

    service rsyslog restart

就可以看到 /var/log/sudo.log记录了所有的sudo操作

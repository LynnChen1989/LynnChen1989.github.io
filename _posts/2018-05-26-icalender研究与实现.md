---
layout: post
title: icalender研究与实现
tags:
    - icalender
categories: icalender
description: icalender研究与实现
---


# 关于icalender


iCalendar是“日历数据交换”的标准（RFC 5545）。 此标准有时指的是“iCal”，即苹果公司的出品的一款同名日历软件（见iCal），这个软件也是此标准的一种实现方式。

关于该标准的介绍，请参考维基百科 https://zh.wikipedia.org/wiki/ICalendar


# 关键字说明

```
BEGIN:VCALENDAR     #日历开始
PRODID:-//Google Inc//Google Calendar 70.9054//EN   #软件信息，这个不是那么严格，可以随意
VERSION:2.0     #遵循的 iCalendar 版本号
CALSCALE:GREGORIAN  #历法：公历
METHOD:PUBLISH      #方法：公开 也可以是 REQUEST 等用于日历间的信息沟通方法
X-WR-CALNAME:yulanggong@gmail.com   #这是一个通用扩展属性 表示本日历的名称
X-WR-TIMEZONE:Asia/Shanghai     #通用扩展属性，表示时区

BEGIN:VEVENT    #事件开始
DTSTART:20090305T112200Z    #开始的日期时间
DTEND:20090305T122200Z      #结束的日期时间
DTSTAMP:20140613T033914Z    #有Method属性时表示 实例创建时间，没有时表示最后修订的日期时间
UID:9r5p7q78uohmk1bbt0iedof9s4@google.com #UID
CLASS:PRIVATE   #保密类型
CREATED:20090305T092105Z    #创建的日期时间
DESCRIPTION:test    #描述
LAST-MODIFIED:20090305T092130Z  #最后修改日期时间
LOCATION:test   #地址
SEQUENCE:1  #排列序号
STATUS:CONFIRMED    #状态 TENTATIVE 不确定 CONFIRMED 确认 CANCELLED 取消
SUMMARY: test   #简介 一般是标题
TRANSP:OPAQUE   #对于忙闲查询是否透明 OPAQUE 不透明 TRANSPARENT 透明
END:VEVENT  #事件结束

END:VCALENDAR   #日历结束
```

## 关键字注意点

### 第一点，通用

```
第一行必须是"BEGIN:VCALENDER"，最后一行必须是"END:VCALENDER"；两行之间数据称之为"icalbody"，icalbody由一系列日历属性和一个以上的日历组件组成。
```

### 第二点，METHOD

```
Description:  When used in a MIME message entity, the value of this
      property MUST be the same as the Content-Type "method" parameter
      value.  If either the "METHOD" property or the Content-Type
      "method" parameter is specified, then the other MUST also be
      specified.
```

所以，当METHOD为REQUEST是，那么python代码中必须是

```
part_cal = MIMEText(ical,'calendar;method=REQUEST')

```

## 在线语法校验

https://icalendar.org/validator.html#results


# 注意

+ 除了outlook和gmail，其余的邮件客户端或者web端都没有完全遵循icalendar协议。这一点反复亲测如此，当我在明确指出ORGANIZER之后，会议参加者
的邮件回执，在outlook和gmail是回执给ORGANIZER的，而不是这封会议邀请的发送者。 因为邮件发送者并不一定是会议组织者。但是可以使用邮件的Reply-To来解决。



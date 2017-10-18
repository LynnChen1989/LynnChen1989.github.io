---
layout: post
title: mysql表达式
tags:
    - mysql
    - 数据库
categories: MySQL
description: mysql表达式
---


[引言] mysql的各种表达式，可以像编写其他程序代码那样使用

#### IF表达式


```
IF(expr1,expr2,expr3)
```


如果 expr1 是TRUE (expr1 <> 0 and expr1 <> NULL)，则 IF()的返回值为expr2; 否则返回值则为 expr3。IF() 的返回值为数字值或字符串值，具体情况视其所在语境而定。


```
select *,if(sva=1,"男","女") as ssva from taname where sva != ""
```


作为表达式的if也可以用CASE when来实现：


```
select CASE sva WHEN 1 THEN '男' ELSE '女' END as ssva from taname where sva != ''


```
在第一个方案的返回结果中， value=compare-value。
而第二个方案的返回结果是第一种情况的真实结果。如果没有匹配的结果值，则返回结果为ELSE后的结果，如果没有ELSE 部分，则返回值为 NULL。


**例如：**

```
SELECT CASE 1 WHEN 1 THEN 'one'
  WHEN 2 THEN 'two' 
   ELSE 'more' END
as testCol
```

将输出 `one`




#### IFNULL(expr1,expr2)


假如expr1 不为 NULL，则 IFNULL() 的返回值为 expr1; 否则其返回值为 expr2。IFNULL()的返回值是数字或是字符串，具体情况取决于其所使用的语境。


```
mysql> SELECT IFNULL(1,0);   => 1
mysql> SELECT IFNULL(NULL,10);  => 10
mysql> SELECT IFNULL(1/0,10);  => 10
mysql> SELECT IFNULL(1/0,'yes');  => 'yes'
```
IFNULL(expr1,expr2) 的默认结果值为两个表达式中更加“通用”的一个，顺序为`STRING`、 `REAL`或 `INTEGER`。


IF ELSE 做为流程控制语句使用


if实现条件判断，满足不同条件执行不同的操作，这个我们只要学编程的都知道if的作用了，下面我们来看看mysql 存储过程中的if是如何使用的吧。


```
IF search_condition THEN 
    statement_list  
[ELSEIF search_condition THEN]  
    statement_list ...  
[ELSE 
    statement_list]  
END IF 
```


与PHP中的IF语句类似，当IF中条件search_condition成立时，执行THEN后的statement_list语句，否则判断ELSEIF中的条件，成立则执行其后的statement_list语句，否则继续判断其他分支。当所有分支的条件均不成立时，执行ELSE分支。search_condition是一个条件表达式，可以由“=、<、<=、>、>=、!=”等条件运算符组成，并且可以使用AND、OR、NOT对多个表达式进行组合。


**例如**
建立一个存储过程，该存储过程通过学生学号（student_no）和课程编号（course_no）查询其成绩（grade），返回成绩和成绩的等级，成绩大于90分的为A级，小于90分大于等于80分的为B级，小于80分大于等于70分的为C级，依次到E级。那么，创建存储过程的代码如下：


```
create procedure dbname.proc_getGrade  
(stu_no varchar(20),cour_no varchar(10))  
BEGIN 
declare stu_grade float;  
select grade into stu_grade from grade where student_no=stu_no and course_no=cour_no;  
if stu_grade>=90 then 
    select stu_grade,'A';  
elseif stu_grade<90 and stu_grade>=80 then 
    select stu_grade,'B';  
elseif stu_grade<80 and stu_grade>=70 then 
    select stu_grade,'C';  
elseif stu_grade70 and stu_grade>=60 then 
    select stu_grade,'D';  
else 
    select stu_grade,'E';  
end if;  
END
```
**注意：**
IF作为一条语句，在END IF后需要加上分号“;”以表示语句结束，其他语句如CASE、LOOP等也是相同的。




#### 顺便说下SQL变量
mysql中变量不用事前申明，在用的时候直接用`“@变量名”`使用就可以了。
1. 第一种用法：set @num=1; 或set @num:=1;                //这里要使用变量来保存数据，直接使用@num变量


2. 第二种用法：select @num:=1; 或 select @num:=字段名 from 表名 where ……


**注意上面两种赋值符号，使用set时可以用“=”或“：=”，但是使用select时必须用“：=赋值”**
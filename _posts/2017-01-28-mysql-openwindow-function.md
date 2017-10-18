---
layout: post
title: mysql开窗函数
tags:
    - mysql
    - 数据库
categories: MySQL
description: mysql开窗函数
---

[引言]在数据库来进行数据分析的过程中，经常会使用到开窗函数

### 创建实例数据
```
drop table if exists heyf_t10;  
create table heyf_t10 (empid int ,deptid int ,salary decimal(10,2) );   
 
insert into heyf_t10 values   
(1,10,5500.00),  
(2,10,4500.00),  
(3,20,1900.00),  
(4,20,4800.00),  
(5,40,6500.00),  
(6,40,14500.00),  
(7,40,44500.00),  
(8,50,6500.00),  
(9,50,7500.00);  
```


### SQL实现
```
SELECT empid, deptid, salary, rank
FROM (
	SELECT  heyf_tmp.empid, heyf_tmp.deptid, heyf_tmp.salary, 
			@rownum:=@rownum+1, IF(@pdept=heyf_tmp.deptid, @rank:=@rank+1, @rank:=1) AS rank, @pdept:=heyf_tmp.deptid
	FROM ( SELECT empid, deptid, salary FROM heyf_t10 ORDER BY deptid ASC, salary DESC ) heyf_tmp,
		( SELECT @rownum:=0, @pdept:= NULL, @rank:=0 ) a
	) result;
```




```
SELECT empid, deptid, salary, rank
FROM (
	SELECT  heyf_tmp.empid, heyf_tmp.deptid, heyf_tmp.salary, 
			@rownum:=@rownum+1, 
			-- IF(@pdept=heyf_tmp.deptid, @rank:=@rank+1, @rank:=1) AS rank, 
			CASE @pdept=heyf_tmp.deptid WHEN true THEN @rank:=@rank+1 ELSE @rank:=1 END AS rank,
			
			@pdept:=heyf_tmp.deptid
	FROM ( SELECT empid, deptid, salary FROM heyf_t10 ORDER BY deptid ASC, salary DESC ) heyf_tmp,
		( SELECT @rownum:=0, @pdept:= NULL, @rank:=0 ) a
	) result;
```
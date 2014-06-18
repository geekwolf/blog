---
layout: post
title: "sql基础操作总结"
date: 2014-04-04 10:29:01 +0800
comments: true
categories: Mysql数据库
keywords: SQL操作,sql基本操作总结
description : Mysql sql基本操作总结
---
本文主要根据自己的学习过程总结的文章，可能并不全面，牛人请绕过~<br>
show create table user \G;      查看创建user表的sql<br>
**修改表结构：**
    alter  table tablename modify[culumn] column_definition [first|after col_name]
    例子：
    修改emp的字段数据类型
    alter table  emp modify ename varchar(20);
    表首增加字段
    alter table emp add column id int(10) first;
    删除字段
    alter table emp drop [column]  id;
    字段改名 
    alter table tablename change [column] old_name column_definition [first/after col_name]
    更改表名
    alter table emp rename [to] test;
**注释：** modify只能修改字段数据类型，不能修改字段名称，first/after 表示修改字段的顺序
<!--more-->
**插入操作:**
    insert into table emp(f1,f2,f3) values(v1,v2,v3);
    insert into table  emp values(v1,v2,v3);
    一次插入多条记录
    insert into table emp(f1,f2,f3)
    values
    (r1_v1,r2_v1,r3_v1),
    (r1_v2,r2_v2,r3_v3);
**查询：**<br>
1.查询不重复记录 distinct
    select distinct deptno from emp;
2.条件查询 where  
where字段比较 \>、<、>=、<=、!= 等，多条件用or、and等
3.排序和限制 order by 
排序关键字（默认是升序排列）：<br>
&emsp;&emsp;DESC 表示按照字段进行降序排列<br>
&emsp;&emsp;ASC   表所升序排列<br>
&emsp;&emsp;limit  n显示前N条记录
4.聚合操作
    语法： select [fidled1,field2,...,fieldn] fun_name
    from  tablename
    [where where_contition]
    [group by  field1,field2,...fieldn]
    [with rollup]
    [having where_contition]

    fun_name :表示做聚合操作的函数，比如：sum、count（*）、max、min
    group by  :表示要进行分类聚合的字段
    with follup ：可选，表示是否对分类聚合后的结果进行在汇总
    having  ：表示在对分类后的结果在进行条件的过滤
**having和where的区别**<br>
&emsp;&emsp;having是对聚合后的结果进行条件的过滤，而where是在聚合前就对记录进行过滤，如果逻辑允许，我们尽可能用where先过滤记录，这样因为结果集减小，将对聚合的效率大大提高，最后在根据逻辑看是否用having进行再过滤

    例如：
     统计各个部门的人数：
     select deptno,count(1) from emp group by deptno;
     统计各个部门的人数，又要统计总人数:
     select deptno,count(1) from emp group by deptno with rollup;
     统计人数大于1的部门：
     select deptno,count(1) from emp group by deptno having count(1)>1;
     统计公司所有员工的薪水总额、最高和最低薪水
     select sum(sal),max(sal),min(sal) from emp;
5.表连接<br>
&emsp;&emsp;表连接分为：内连接和外连接；内连接：仅选出两张表中互相匹配的记录；外连接：选出其他不匹配的记录

    例如：
     查询雇员的名字和所在部门名称（雇员和部门分属于两个表）
     select ename,deptname from emp,dept where emp.deptno=dept.deptno;
       外连接分为： 
       左连接：包含所有的左边表中的记录甚至是右边表中没有和它匹配的记录
       右连接：包含所有的右边表中的记录甚至是左边表中没有和它匹配的记录
    
    例如：查询emp中所有用户和所在部门名称
    select ename,deptname from dept right join emp on dept.deptno=emp.deptno;
    或可用左连接代替
    select ename,deptname from emp left join dept on emp.deptno=dept.deptno;

>表连接的好文章：
http://coolshell.cn/articles/3463.html<br>
>MySQL的联结（Join）语法：
http://www.blogjava.net/chenpengyi/archive/2005/10/17/15747.html<br>
6.子查询
用于子查询的关键字包括in、not in、=、!=、exits、not exits等

    例如：
    从emp表中查询出所有部门在dept表中的所有记录
    select * from emp where deptno in(select deptno from dept);
    可以使用表连接替换：
    select emp.*  from emp,dept where emp.deptno=dept.deptno;
    
    查询记录数唯一，可用=代替in
    select * from emp where deptno = (select deptno from dept limit 1);
    
6.记录联合（union、union all）
场景：将两个表的数据按照一定的查询条件查询出来后，在合并显示
union和union all的区别在于union all把结果集直接合并在一起，而union是将union all后的结果进行了去重DISTINCT后的结果
![](/images/mysql/unionall.png)<br>
**查看帮助：**
    
    显示可供查询的分类
    mysql>? contents
    
    针对某个分类查看帮助
    mysql>? data types
    
    查看int数据类型介绍
    mysql>? int
    
    查看关键字的语法
    mysql>? show
    mysql>? create table


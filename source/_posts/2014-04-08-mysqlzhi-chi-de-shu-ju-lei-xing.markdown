---
layout: post
title: "Mysql支持的数据类型"
date: 2014-04-08 19:36:14 +0800
comments: true
categories: [Mysql数据库]
keywords: Mysql支持的数据类型,数据类型
description: MySQL支持的数据类型
---
#####一、数值类型

![](/images/mysql/numbers.png)

**注意事项：**<br>
**1. int类型里面默认的数据宽度是11，即int(11)**<br>
&emsp;关于zerofill，在数字位数不够的空间用字符0填充<br>
&emsp;例如：

![](/images/mysql/n2.png)

**2.整数类型有AUTO\_INCREMENT的属性**<br>
&emsp;AUTO\_INCREMENT的值一般从1开始，每行增加1，插入NULL到一个AUTO\_INCREMENT列时，Mysql插入一个比该列中当前最大值大1的值<br>
&emsp;一个表中最多只能有一个AUTO\_INCREMENT列<br>
&emsp;使用AUTO\_INCREMENT的条件：<br>
&emsp;该列应该定义为NOT NULL，并且为PRIMARY KEY 或者UNIQUE键<br>
<!--more-->
&emsp;例如：

    CREATE TABLE AI(ID INT AUTO_INCREMENT NOT NULL PRIMARY KEY);
    CREATE TABLE AI(ID INT AUTO_INCREMENT NOT NULL ,PRIMARY KEY(ID));
    CREATE TABLE AI(ID INT AUTO_INCREMENT NOT NULL ,UNIQUE(ID));

**3.BIT位数据类型查询方式**<br>
&emsp;bit位数据类型列不能直接查询得出结果，使用bin()显示为二进制格式，或者hex()显示为十六进制格式函数进行读取

&emsp;例如：

![](/images/mysql/n3.png)

#####二、日期和时间类型

![](/images/mysql/time.png)

**注意事项:**<br>
**1.经常插入或者跟新日期为当前系统时间通常用TIMESTAMP来表示**<br>
&emsp;但是MySQL规定TIMESTAMP类型字段只能有一列的默认值为current_timestamp,第二个TIMESTAMP字段的默认值为0，即0000-00-00 00：00：00；时间与时区相关<br>

**2. DATETIME是DATE和TIME的结合**

    mysql> create table t(d date,t time,dt datetime);
    Query OK, 0 rows affected (0.11 sec)
    mysql> desc t;
    +-------+----------+------+-----+---------+-------+
    | Field | Type | Null | Key | Default | Extra |
    +-------+----------+------+-----+---------+-------+
    | d | date | YES | | NULL | |
    | t | time | YES | | NULL | |
    | dt | datetime | YES | | NULL | |
    +-------+----------+------+-----+---------+-------+
    3 rows in set (0.00 sec)
    mysql> select now();
    +---------------------+
    | now() |
    +---------------------+
    | 2014-04-08 11:38:24 |
    +---------------------+
    1 row in set (0.01 sec)
    mysql> insert into t values(now(),now(),now());
    Query OK, 1 row affected, 1 warning (0.00 sec)
    mysql> show warnings;
    +-------+------+----------------------------------------+
    | Level | Code | Message |
    +-------+------+----------------------------------------+
    | Note | 1265 | Data truncated for column 'd' at row 1 |
    +-------+------+----------------------------------------+
    1 row in set (0.00 sec)
    mysql> select * from t;
    +------------+----------+---------------------+
    | d | t | dt |
    +------------+----------+---------------------+
    | 2014-04-08 | 11:38:40 | 2014-04-08 11:38:40 |
    +------------+----------+---------------------+
    1 row in set (0.00 sec)
    

**3.TIMESTAMP 和DATETIME的区别**<br>
&emsp;A.TIMESTAMP支持的范围较小<br>
&emsp;B.表中第一个TIMESTAMP列自动设置为系统时间；如果列中插入NULL或者不明确的赋值都会自动设置该列的值为当前的日期和时间；超过TIMESTAMP的取值范围时，直接由0000-00-00 00：00：00填补<br>
&emsp;C.TIMESTAMP的插入和查询都受当地时区的影响，更能反应实际日期；DATETIME则只能反映出插入时当地的失去，其他时区的人查看数据必然会有误差<br>
  
**4.YEAR**<br>
&emsp;有4位格式（允许的值是1901~2155、0000，默认值）的年，有2位格式（允许的值70-69，即1970~2069,Mysql5.5.27以后版本不再支持）的年


#####三、字符串类型

![](/images/mysql/char.png)

**注意事项：**<br>
**1.varchar和char的不同之处在于存储方式不同**<br>
&emsp;A. char列的长度固定为创建表时声明的长度，长度可以为0~255；varchar列中的值为可变长字符串，长度可以为0~65535

![](/images/mysql/charst.png)

&emsp;B.查询时char列删除了尾部的空格，而varchar则保留了这些空格<br>
&emsp;例如：

    mysql> CREATE TABLE vc (v VARCHAR(4), c CHAR(4));
    Query OK, 0 rows affected (0.10 sec)
    mysql> INSERT INTO vc VALUES ('ab ', 'ab ');
    Query OK, 1 row affected (0.00 sec)
    mysql> SELECT CONCAT('(', v, ')'), CONCAT('(', c, ')') FROM vc;
    +---------------------+---------------------+
    | CONCAT('(', v, ')') | CONCAT('(', c, ')') |
    +---------------------+---------------------+
    | (ab ) | (ab) |
    +---------------------+---------------------+
    1 row in set (0.00 sec)
    mysql> select length(v),length(c) from vc;
    +-----------+-----------+
    | length(v) | length(c) |
    +-----------+-----------+
    | 4 | 2 |
    +-----------+-----------+
    1 row in set (0.00 sec)
    mysql> select concat(v,'+'),concat(c,'+') from vc;
    +---------------+---------------+
    | concat(v,'+') | concat(c,'+') |
    +---------------+---------------+
    | ab + | ab+ |
    +---------------+---------------+
    1 row in set (0.00 sec)

**2.BINARY和VARBINARY类型：**类似CHAR和VARCHAR类型，不同的是它们包含二进制字符串而不包含非二进制字符串；CHAR(M)、VARCHAR(M)中的M表示字符长度，BINARY(M)和VARBINARY(M)中M表示字节的长度

**3.ENUM类型(枚举类型)**<br>
&emsp;对于1~255个成员的枚举需要1个字节存储，对于255~65535个成员，需要2个字节存储，最多允许65535个成员<br>
&emsp;例如:

![](/images/mysql/enum.png)

&emsp;ENUM类型是忽略大小写的，在存储"M" "f"时将它们都转换成大写，插入不存在的ENUM指定的范围内的值时，并没有警告，而是插入了enum('M','F')的第一个值“M”，只允许从集合中选取单个值，不能一次取多个

**4.SET类型**<br>
&emsp;SET是一个字符串对象，包含0~64个成员，与ENUM的区别在于可以一次选取多个成员<br>
&emsp;1~8成员的集合，占1个字节<br>
&emsp;9~16成员的集合，占2个字节<br>
&emsp;17~24成员的集合，占用3个字节<br>
&emsp;25~32成员的集合，占用4个字节<br>
&emsp;33~64成员的集合，占用8个字节<br>
&emsp;例如：

    mysql> create table t (col set('a','b','c','d'));
    Query OK, 0 rows affected (0.22 sec)
    mysql> desc t;
    +-------+----------------------+------+-----+---------+-------+
    | Field | Type | Null | Key | Default | Extra |
    +-------+----------------------+------+-----+---------+-------+
    | col | set('a','b','c','d') | YES | | NULL | |
    +-------+----------------------+------+-----+---------+-------+
    1 row in set (0.00 sec)
    mysql> insert into t values('a,b'),('a,d,a'),('a,b'),('a,c'),('a');
    Query OK, 5 rows affected (0.00 sec)
    Records: 5 Duplicates: 0 Warnings: 0
    mysql> select * from t;
    +------+
    | col |
    +------+
    | a,b |
    | a,d |
    | a,b |
    | a,c |
    | a |
    +------+
    5 rows in set (0.00 sec)
对于('a,d,a')这样包含重复成员的集合将取一次，写入后的结果为“a,d”
>http://dev.mysql.com/doc/refman/5.5/en/data-types.html


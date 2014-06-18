---
layout: post
title: "理解MySQL运算符和常用内置函数"
date: 2014-04-09 14:56:41 +0800
comments: true
categories: [Mysql数据库]
keywords: Mysql运算符、内置函数
description: 理解MySQL运算符和常用函数
---
#####一、MySQL中的运算符

![](/images/mysql/suanshu.png)

![](/images/mysql/lj.png)
<!--more-->

**注意事项：<br>
1.在除法运算和模数运算中，如果除数是0，将是非法除数，结果返回NULL**<br>
&emsp;取模运算中，也可以用MOD(a,b)函数或者a%b

    mysql> select 1/0, 100%0;
    +------+-------+
    | 1/0 | 100%0 |
    +------+-------+
    | NULL | NULL |
    +------+-------+
    1 row in set (0.01 sec)
    mysql> select 3%2,mod(3,2);
    +------+----------+
    | 3%2 | mod(3,2) |
    +------+----------+
    | 1 | 1 |
    +------+----------+
    1 row in set (0.00 sec)

**2.NULL只能用<=>进行比较，其他的比较运算符时返回NULL**

    mysql> select 'a'<'b','a'<'a',1<2,null<=>null;
    +---------+---------+-----+-------------+
    | 'a'<'b' | 'a'<'a' | 1<2 | null<=>null |
    +---------+---------+-----+-------------+
    | 1 | 0 | 1 | 1 |
    +---------+---------+-----+-------------+
    1 row in set (0.02 sec)
    mysql> select 'a'<'b','a'<'a',1<2,null<null;
    +---------+---------+-----+-----------+
    | 'a'<'b' | 'a'<'a' | 1<2 | null<null |
    +---------+---------+-----+-----------+
    | 1 | 0 | 1 | NULL |
    +---------+---------+-----+-----------+
    1 row in set (0.00 sec)
    
**3.BETWEEN IN**<br>
&emsp;between运算符使用“a BETWEEN min AND max”当a大于等于min并且小于等于max返回1，否则返回0
&emsp;IN运算符使用"a IN（values1,values2,...)"，当a的值存在于列表中时，则郑鄂表达式返回值1，否则0

    mysql> select 10 between 10 and 20,9 between 10 and 20;
    +----------------------+---------------------+
    | 10 between 10 and 20 | 9 between 10 and 20 |
    +----------------------+---------------------+
    | 1 | 0 |
    +----------------------+---------------------+
    1 row in set (0.00 sec)
    mysql> select 1 in(1,2,3),'t' in ('t','a','b','f'),0 in(1,2);
    +-------------+--------------------------+-----------+
    | 1 in(1,2,3) | 't' in ('t','a','b','f') | 0 in(1,2) |
    +-------------+--------------------------+-----------+
    | 1 | 1 | 0 |
    +-------------+--------------------------+-----------+
    1 row in set (0.00 sec)

**4.REGEXP运算符格式"str REGEXP str_pat"**<br>
&emsp;当str字符串中含有str_pat相匹配的字符串时返回1，否则0

    mysql> select 'abcdef' regexp 'ac','abcdef' regexp 'ab','abcdefg' regexp 'k';
    +----------------------+----------------------+----------------------+
    | 'abcdef' regexp 'ac' | 'abcdef' regexp 'ab' | 'abcdefg' regexp 'k' |
    +----------------------+----------------------+----------------------+
    | 0 | 1 | 0 |
    +----------------------+----------------------+----------------------+
    1 row in set (0.00 sec)

**5. 逻辑与AND和逻辑或OR**<br>
&emsp;AND：当所有操作数都为非零，并且不为NULL时，返回1；当一个或多个为0时，返回0；操作数任何一个为NULL，则返回NULL<br>
&emsp;OR   : 当两个操作数均为非NULL值时，如有任意一个为非零值，则返回1，否则0；<br>
&emsp;&emsp;&emsp; 当有一个操作数为NULL时，如另外一个为非0，则结果1，否则NULL;<br>
&emsp;&emsp;&emsp; 如果两个操作数均为NULL，则所得结果为NULL

    mysql> select (1 and 1),(0 and 1),(3 and 1),(1 and null);
    +-----------+-----------+-----------+--------------+
    | (1 and 1) | (0 and 1) | (3 and 1) | (1 and null) |
    +-----------+-----------+-----------+--------------+
    | 1 | 0 | 1 | NULL |
    +-----------+-----------+-----------+--------------+
    1 row in set (0.00 sec)
    mysql> select (1 or 0),(0 or 0),(1 or null),(1 or 1),(null or null);
    +----------+----------+-------------+----------+----------------+
    | (1 or 0) | (0 or 0) | (1 or null) | (1 or 1) | (null or null) |
    +----------+----------+-------------+----------+----------------+
    | 1 | 0 | 1 | 1 | NULL |
    +----------+----------+-------------+----------+----------------+
    1 row in set (0.00 sec)

**6.位运算**<br>
&emsp;位与对多个操作数的二进制位做逻辑与操作<br>

    mysql> select bin(2);
    +--------+
    | bin(2) |
    +--------+
    | 10 |
    +--------+
    1 row in set (0.00 sec)
    mysql> select bin(3);
    +--------+
    | bin(3) |
    +--------+
    | 11 |
    +--------+
    1 row in set (0.00 sec)
    mysql> select bin(100);
    +----------+
    | bin(100) |
    +----------+
    | 1100100 |
    +----------+
    1 row in set (0.00 sec)
    mysql> select 2&3&100;
    +---------+
    | 2&3&100 |
    +---------+
    | 0 |
    +---------+
    1 row in set (0.00 sec)

**7.位取反**<br>
&emsp;在MySQL中，常量数字默认会以8个字节来表示，8字节就是64位，常量1的二进制表示为63个0加1个1，位取反后就是63个1加1个0，转换成十进制后就是18446744073709551614

![](/images/mysql/qf.png)

**8.位右移**<br>

![](/images/mysql/wyy.png)

#####二、运算符的优先级

![](/images/mysql/yxj.png)

#####三、常用内置函数

![](/images/mysql/strfun.png)

![](/images/mysql/numfun.png)

![](/images/mysql/timefun.png)

![](/images/mysql/otherfun.png)

**注意事项：**<br>
date_format(date,fmt)fmt格式:<br>
[http://dev.mysql.com/doc/refman/5.5/en/date-and-time-functions.html#function_date-format](http://dev.mysql.com/doc/refman/5.5/en/date-and-time-functions.html#function_date-format)<br>
date_add(date,INTERVAL expr type) type类型：<br>
[http://dev.mysql.com/doc/refman/5.5/en/date-and-time-functions.html#function_date-add](http://dev.mysql.com/doc/refman/5.5/en/date-and-time-functions.html#function_date-add)

>http://dev.mysql.com/doc/refman/5.5/en/functions.html
>

---
layout: post
title: "Mysql事件调度器(Event Scheduler)"
date: 2014-03-20 18:15:54 +0800
comments: true
categories: [Mysql数据库]
keywords: Mysql事件调度器,Event Scheduler,Events
description: Mysql事件调度器
---
#####目录:
[一、查看当前是否开启了event scheduler三种方法:](#t1)<br>
[二、启动关闭event scheduler方法](#t2)<br>
[三、创建Event](#t3)<br>
[四、修改Event](#t4)<br>
[五、查询Event信息](#t5)<br>


&emsp;&emsp;Mysql中的事件调度器Event Scheduler类似于linux下的crontab计划任务的功能,它是由一个特殊的时间调度线程执行的
#####[一、查看当前是否开启了event scheduler三种方法](id:t1):
1)     SHOW VARIABLES LIKE 'event_scheduler';<br>
2)     SELECT @@event_scheduler;<br>
3)     SHOW PROCESSLIST;(是否有State为：Waiting for next activation的进程，User为event_scheduler)<br>
#####[二、启动关闭event scheduler方法](id:t2):
 时间调度器是否开启由全局变量event_scheduler决定，它有三个可以设定的值：
-  OFF  : 事件调度器是关闭的，调度线程并没有运行，并且在SHOW PROCESSLIST中不显示，默认值是OFF
-  ON ：事件调度器是开启的，调度线程并没有运行，并且执行所有的调度事件，通过SHOW PROCESSLIST可以查看Waiting for next activation的进程
-  DISABLED : 设定这个值表示Event Scheduler是被禁止的，无法在Mysql运行状态下改变其值<br>
<!--more--> 
 **注：**在Mysql启动时如果在my.cnf设置了event_scheduler=ON（OFF or 1 or 0）时，就不能在运行时修改撑DISABLED，如果设置event_scheduler=DISABLED时，就不能在运行时修改其值为ON （ OFF or 1 or 0）<br>

    mysql> SELECT @@event_scheduler;
    +-------------------+
    | @@event_scheduler |
    +-------------------+
    | DISABLED |
    +-------------------+
    1 row in set (0.00 sec)
    mysql> SET @@global.event_scheduler = 1; 
    ERROR 1290 (HY000): The MySQL server is running with the --event-scheduler=DISABLED or --skip-grant-tables option so it cannot execute this statement

    
    在mysql运行时开启Event（4种方法均可）：
    SET GLOBAL event_scheduler = ON;
    SET @@global.event_scheduler = ON;
    SET GLOBAL event_scheduler = 1;
    SET @@global.event_scheduler = 1;
    
    在mysql运行时关闭Event（4种方法均可）：
    SET GLOBAL event_scheduler = OFF;
    SET @@global.event_scheduler = OFF;
    SET GLOBAL event_scheduler = 0;
    SET @@global.event_scheduler = 0;



#####[三、创建Event](id:t3):
    语法：
    CREATE
        [DEFINER = { user | CURRENT_USER }]
        EVENT
        [IF NOT EXISTS]
        event_name
        ON SCHEDULE schedule
        [ON COMPLETION [NOT] PRESERVE]
        [ENABLE | DISABLE | DISABLE ON SLAVE]
        [COMMENT 'comment']
        DO event_body;
        
    schedule:
        AT timestamp [+ INTERVAL interval] ...
      | EVERY interval
        [STARTS timestamp [+ INTERVAL interval] ...]
        [ENDS timestamp [+ INTERVAL interval] ...]
    interval:
        quantity {YEAR | QUARTER | MONTH | DAY | HOUR | MINUTE |
                 WEEK | SECOND | YEAR_MONTH | DAY_HOUR | DAY_MINUTE |
                 DAY_SECOND | HOUR_MINUTE | HOUR_SECOND | MINUTE_SECOND}
**说明：**

&emsp;&emsp;DEFINER默认是CREATE EVENT的用户,可以理解为DEFINER=CURRENT_USER,指该event的用户，服务器在执行该事件时，使用该用户来检查权限；如果设置语法:'user_name'@'host_name'，如果当前CREATE EVENT用户没有supser权限，则无法将该event指派给其他用户；如果有super权限，则可以指定任意存在的用户，若不存在，时间执行时报错<br>
&emsp;&emsp;IF NOT EXISTS ： 如果在同一个schema创建一个已经存在的event_name时不会做任何操作，也不会出错，但会出现warings：该event已经存在；如果不增加此关键词已经存在的话提示ERROR： 1537 (HY000): Event 'countsum' already exists<br>
&emsp;&emsp;ON SCHEDULE ：用于设置什么时间执行，执行的频率及执行多久的问题<br>
&emsp;&emsp;AT timestamp ：表示在给定的datetime或者timestamp的时间执行一次<br>
&emsp;&emsp;+ INTERVAL interval：表示从AT timestamp多久之后执行<br>
&emsp;&emsp;EVERY interval ：有规律的重复执行<br>
&emsp;&emsp;[ENABLE | DISABLE]可是设置该事件创建后状态是否开启或关闭，默认为ENABLE<br>
&emsp;&emsp;[COMMENT 'comment']可以给该事件加上注释。

    event创建时间的3周2天后：
    AT  CURRENT_TIMESTAMP + INTERVAL 3 WEEK + INTERVAL 2 DAY
    
    2分钟10秒： 
    + INTERVAL '2:10' MINUTE_SECOND
    
    每6周：
    EVERY 6 WEEK
    
    从现在开始30分钟后每12小时执行一次到从现在到4周后结束执行：
    EVERY 12 HOUR STARTS CURRENT_TIMESTAMP + INTERVAL 30 MINUTE ENDS CURRENT_TIMESTAMP + INTERVAL 4 WEEK 

**实例：**<br>
前提：创建EVENT的用户需要只少对应schema的EVENT权限<br>
最基本的create event只需要三个部分：<br>
1. create event关键字以及一个event名称<br>
2. on schedule子句<br>
3. do子句<br>

    1. 在创建事件myevent1小时后执行，执行一条更新
    
    CREATE EVENT myevent
        ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 1 HOUR
    DO
      UPDATE myschema.mytable SET mycol = mycol + 1;
      
    2.2014年3月20日12点整清空test表：
    
    CREATE EVENT e_test
        ON SCHEDULE AT TIMESTAMP '2014-03-20 12:00:00'
        DO TRUNCATE TABLE test.aaa;
        
    3.5天后开启每天定时清空test表：
    
    CREATE EVENT e_test
        ON SCHEDULE EVERY 1 DAY
        STARTS CURRENT_TIMESTAMP + INTERVAL 5 DAY
        DO TRUNCATE TABLE test.aaa;
    
    4.每天定时清空test表，5天后停止执行
    
    CREATE EVENT e_test
        ON SCHEDULE EVERY 1 DAY
        ENDS CURRENT_TIMESTAMP + INTERVAL 5 DAY
        DO TRUNCATE TABLE test.aaa;
        
    5.5天后开启每天定时清空test表，一个月后停止执行：
    
    CREATE EVENT e_test
        ON SCHEDULE EVERY 1 DAY
        STARTS CURRENT_TIMESTAMP + INTERVAL 5 DAY
        ENDS CURRENT_TIMESTAMP + INTERVAL 1 MONTH
        DO TRUNCATE TABLE test.aaa;
        
    6.每天定时清空test表(只执行一次，任务完成后就终止该事件)：
    
    CREATE EVENT e_test
        ON SCHEDULE EVERY 1 DAY
        ON COMPLETION NOT PRESERVE
        DO TRUNCATE TABLE test.aaa;
    
    [ON COMPLETION [NOT] PRESERVE]可以设置这个事件是执行一次还是持久执行，默认为NOT PRESERVE。

#####[四、修改Event](id:t4):
    ALTER
       [DEFINER = { user | CURRENT_USER }]
       EVENT event_name
       [ON SCHEDULE schedule]
       [ON COMPLETION [NOT] PRESERVE]
       [RENAME TO new_event_name]
       [ENABLE | DISABLE | DISABLE ON SLAVE]
       [COMMENT 'comment']
       [DO event_body]

**说明：**<br>
&emsp;&emsp;对于任何一个拥有定义在database里面事件的event权限的用户都可以修改event，并且成功需改后，那个用户就会成为此event的definer<br>

    实例：
    CREATE EVENT myevent
        ON SCHEDULE
          EVERY 6 HOUR
        COMMENT 'A sample comment.'
        DO
          UPDATE myschema.mytable SET mycol = mycol + 1;
    
    将上面的event从开始之后每6个小时执行一次改为从开始4个小时后每12小时执行一次
    
    只修改schedule
    ALTER EVENT myevent
        ON SCHEDULE
          EVERY 12 HOUR
        STARTS CURRENT_TIMESTAMP + INTERVAL 4 HOUR;
    同时修改schedule和body
    ALTER EVENT myevent
        ON SCHEDULE
          AT CURRENT_TIMESTAMP + INTERVAL 1 DAY
        DO
          TRUNCATE TABLE myschema.mytable;

关闭、启动、别名、移动、删除event：

    临时关闭某个event
    ALTER EVENT myevent DISABLE;

    开启某个event
    ALTER EVENT myevent ENABLE;

    别名某个event
    ALTER EVENT olddb.myevent
    RENAME TO newdb.myevent;

    将myevent从olddb库移动到newdb库
    ALTER EVENT olddb.myevent
    RENAME TO newdb.myevent;

    删除event
    DROP EVENT [IF EXISTS] event_name

#####[五、查询Event信息](id:t5):

Event信息相关表：<br>
information_schema.events<br>
mysql.event 

查看事件的创建信息<br>
show create event countsum \G  

查看sem库的events信息<br>
USE sem；<br>
SHOW EVENTS \G

SHOW EVENTS FROM sem;

#####参考资料:<br>
> https://dev.mysql.com/doc/refman/5.5/en/events.html



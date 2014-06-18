---
layout: post
title: "MySQL5.6基于GTID复制配置"
date: 2014-06-12 16:06:59 +0800
comments: true
categories: [Mysql数据库]
keywords: GTID,GTID原理
description: MySQL5.6基于GTID复制配置,新增slave及故障的处理办法
---
一、[什么是GTID？](#gg1)<br>
二、[GTID的表示方式](#gg2)<br>
三、[基于GTID的复制配置](#gg3)<br>
四、[基于GTID复制增加新的slave](#gg4)<br>
五、[基于GTID复制出错的解决办法](#gg5)<br>
&emsp;&emsp;[注意事项](#gg6)<br>
&emsp;&emsp;[参考文档](#gg7)<br>


**[一、什么是GTID？](id:gg1)<br>**
&emsp;&emsp;GTID(Global Transaction Identifiers)是全局事务标识<br>
&emsp;&emsp;当使用GTIDS时，在主上提交的每一个事务都会被识别和跟踪，并且运用到所有从MySQL，而且配置主从或者主从切换时不再需要指定 master\_log\_files和master\_log\_pos；由于GTID-base复制是完全基于事务的，所以能很简单的决定主从复制的一致性；官方建议Binlog采用Row格式

**[二、GTID的表示方式](id:gg2)<br>**
source\_id：transaction\_id<br>
source\_id：表示执行事务的主库的UUID(server\_uuid:Mysql5.6的data目录下启动时会生成auto.cnf文件记录了uuid，重启后uuid不变，删除文件后会重新生成新的uuid)；<br>
transaction\_id：是一个从1开始自增的计数，表示在这个主库上执行的第n个事务；<br>
由于每台Mysql的uuid是全球唯一的，transaction\_id自身唯一，就保证了GTID全局唯一性
<!--more-->      
    mysql> show variables like 'server_uuid'; 
    +---------------+--------------------------------------+
    | Variable_name | Value |
    +---------------+--------------------------------------+
    | server_uuid | 4468c0e8-ef6f-11e3-9c2c-0200c0a80ad8 |
    +---------------+--------------------------------------+
    1 row in set (0.00 sec)

**[三、基于GTID的复制配置](id:gg3)<br>**

**master：**192.168.10.216<br>
**slave ：**192.168.10.217<br>
**步骤：**<br>
&emsp;&emsp;修改主从my.cnf增加GTID支持-->主只读-->拷贝数据到从数据目录-->重启主从-->在从上进行配置<br>
1.修改主从my.cnf增加GTID支持<br>

    主Mysql配置：
    server-id=216   
    binlog-format=ROW
    gtid-mode=on
    enforce-gtid-consistency=true 
    log-bin=mysql-bin
    log-slave-updates=true   slave更新是否记入日志

    从Mysql配置：
    server-id=217   同一个复制拓扑中的所有服务器的id号必须惟一
    binlog-format=ROW
    gtid-mode=on  启用gtid类型，否则就是普通的复制架构
    enforce-gtid-consistency=true 强制GTID的一致性
    log-bin=mysql-bin
    log-slave-updates=true   slave更新是否记入日志
    只从库配置：
    slave-paralles-workers 设定从服务器的SQL线程数；0表示关闭多线程复制功能；

2.主只读<br>

    mysql> SET @@global.read_only = ON;
拷贝主数据到从目录<br>

3.重启主从Mysql<br>

4.在从上配置基于GTID的复制<br>

    mysql> CHANGE MASTER TO 
     	 > MASTER_HOST = ‘192.168.10.216’,
         > MASTER_PORT = 3306,
         > MASTER_USER = 'rep',
         > MASTER_PASSWORD = 'geekwolf',
         > MASTER_AUTO_POSITION = 1;

5.启动从库<br>

    mysql> start slave; 
    Query OK, 0 rows affected, 1 warning (0.00 sec)
    
    mysql> show slave status \G
    *************************** 1. row ***************************
       Slave_IO_State: Waiting for master to send event
      Master_Host: 192.168.10.216
      Master_User: rep
      Master_Port: 3306
    Connect_Retry: 60
      Master_Log_File: mysql-bin.000002
      Read_Master_Log_Pos: 41921904
       Relay_Log_File: relay-bin.000002
    Relay_Log_Pos: 64520
    Relay_Master_Log_File: mysql-bin.000002
     Slave_IO_Running: Yes
    Slave_SQL_Running: Yes
      Replicate_Do_DB:
      Replicate_Ignore_DB:
       Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
      Replicate_Wild_Ignore_Table: mysql.%
       Last_Errno: 0
       Last_Error:
     Skip_Counter: 0
      Exec_Master_Log_Pos: 41921904
      Relay_Log_Space: 64718
      Until_Condition: None
       Until_Log_File:
    Until_Log_Pos: 0
       Master_SSL_Allowed: No
       Master_SSL_CA_File:
       Master_SSL_CA_Path:
      Master_SSL_Cert:
    Master_SSL_Cipher:
       Master_SSL_Key:
    Seconds_Behind_Master: 0
    Master_SSL_Verify_Server_Cert: No
    Last_IO_Errno: 0
    Last_IO_Error:
       Last_SQL_Errno: 0
       Last_SQL_Error:
      Replicate_Ignore_Server_Ids:
     Master_Server_Id: 216
      Master_UUID: 21ad8db5-f038-11e3-a14a-0200c0a80ad8
     Master_Info_File: /usr/local/mysql/data/master.info
    SQL_Delay: 0
      SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Reading event from the relay log
       Master_Retry_Count: 86400
      Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
       Master_SSL_Crl:
       Master_SSL_Crlpath:
       Retrieved_Gtid_Set: 21ad8db5-f038-11e3-a14a-0200c0a80ad8:76793-77026
    Executed_Gtid_Set: 21ad8db5-f038-11e3-a14a-0200c0a80ad8:1-77025
    Auto_Position: 1
    1 row in set (0.00 sec)

**注：**<br>

两个Yes代表复制正常<br>
Slave\_IO_Running: Yes  <br>
Slave\_SQL\_Running: Yes  <br>

基于GTID复制的新特性：<br>
 Retrieved\_Gtid\_Set: 21ad8db5-f038-11e3-a14a-0200c0a80ad8:76793-77026<br>
 Executed\_Gtid\_Set: 21ad8db5-f038-11e3-a14a-0200c0a80ad8:1-77025<br>
 
Retrieved\_Gtid\_Set项：记录了relay日志从Master获取了binlog日志的位置<br>
Executed\_Gtid\_Set项：记录本机执行的binlog日志位置（如果是从机，包括Master的binlog日志位置和slave本身的binlog日志位置）<br>

**[四、基于GTID复制增加新的slave](id:gg4)<br>**
&emsp;&emsp;备份主MySQL数据，记录主gtid\_executed-->将备份数据恢复到从数据目录-->设置从gtid\_purged的值为主的gtid\_executed值-->启动复制即可

1.使用mysqldump备份主数据<br>
&emsp;&emsp;mysqldump --all-databases --single-transaction --triggers --routines --host=127.0.0.1 --port=3306 --user=root  --password=geekwolf > backup.sql<br>
&emsp;&emsp;亦可以使用xtrabackup也支持GTID：<br>
&emsp;&emsp;请参考:[http://www.mysqlperformanceblog.com/2013/05/09/how-to-create-a-new-or-repair-a-broken-gtid-based-slave-with-percona-xtrabackup/](http://www.mysqlperformanceblog.com/2013/05/09/how-to-create-a-new-or-repair-a-broken-gtid-based-slave-with-percona-xtrabackup/)

2.传到从MySQL，恢复数据<br>
&emsp;&emsp;由于新版本msqldump会记录并设置GTID\_PURGED的值等于主的GTID\_EXECUTED，所以只需要将sql导入到从库即可

3.启动主从复制<br>

    从库执行
    mysql > CHANGE MASTER TO MASTER_HOST='127.0.0.1', MASTER_USER='root', MASTER_PASSWORD=geekwolf', MASTER_PORT=3306, MASTER_AUTO_POSITION = 1;
    mysql > START SLAVE;
    
**[五、基于GTID复制出错的解决办法](id:gg5)<br>**

**问题:**<br>

    Slave_IO_Running: No
    Slave_SQL_Running: Yes
    Last_IO_Error: Got fatal error 1236 from master when reading data from binary log: 'The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires.'
**解决思路:**<br>

从复制跳过已经丢失的binlog，继续复制或者重新做主从（可以参考上面的操作）

    主MySQL：
    mysql> show global variables like '%gtid_executed%';
    +---------------+-----------------------------------------------+
    | Variable_name | Value |
    +---------------+-----------------------------------------------+
    | gtid_executed | 21ad8db5-f038-11e3-a14a-0200c0a80ad8:1-223937 |
    +---------------+-----------------------------------------------+
    1 row in set (0.00 sec)
    
    从MySQL：
    mysql> set global GTID_PURGED="21ad8db5-f038-11e3-a14a-0200c0a80ad8:1-223937";
    ERROR 1840 (HY000): @@GLOBAL.GTID_PURGED can only be set when @@GLOBAL.GTID_EXECUTED is empty.
    
    mysql> reset master;
    Query OK, 0 rows affected (0.19 sec)
    mysql> show global variables like 'GTID_EXECUTED';
    +---------------+-----------------------------------------------+
    | Variable_name | Value |
    +---------------+-----------------------------------------------+
    | gtid_executed |  |
    +---------------+-----------------------------------------------+
    1 row in set (0.00 sec)
    
    mysql> stop slave;
    Query OK, 0 rows affected, 1 warning (0.00 sec)
    
    mysql> set global GTID_PURGED="21ad8db5-f038-11e3-a14a-0200c0a80ad8:1-223937";
    Query OK, 0 rows affected (0.13 sec)
    
    mysql> start slave;
    Query OK, 0 rows affected (0.04 sec)
    
    mysql> show slave status\G
    [...]
    Slave_IO_Running: Yes
    Slave_SQL_Running: Yes
    [...]
    

**[注意事项：](id:gg6)<br>**
&emsp;&emsp;使用基于GTID复制时，不需要再关心master\_log\_file和master\_log\_pos，替代的是只需要知道master上的GTID，并且配置在从上即可；<br>
&emsp;&emsp;记录GTID的有两个全局变量：gtid\_executed和gtid\_purged<br>

**与GTID复制相关的参数：**

![](http://geekwolf.github.io/images/mysql/gtid.png)

GTID\_EXECUTED  ：表示已经在该实例上执行过的事务；执行RESET MASTER可以置空该参数；也可以设置GTID\_NEXT执行一个空事务来影响GTID_EXECUTED<br>
GTID\_NEXT           ：是SESSION级别参数，表示下一个事务被执行使用的GTID（show variables like 'gtid_%';）<br>
GTID\_PURGED      ：表示被删除的binlog事务GTID，它是GTID\_EXCUTED的子集，MySQL5.6.9，该参数无法被设置<br>
GTID\_OWENED    ：表示正在执行的事务的GTID以及对应的线程ID<br>

如果设置MASTER\_AUTO\_POSITION = 1表示主从复制连接使用基于GTID的方式复制

    CHANGE MASTER TO MASTER_HOST='192.168.10.216',MASTER_USER='rep',MASTER_PASSWORD='geekwolf',MASTER_AUTO_POSITION=1;

如果在GTID复制模式下想要使用基于文件的复制协议需要MASTER\_AUTO\_POSITION=0（至少指定其中MASTER\_LOG\_FILE、MASTER\_LOG\_POSITION一个）

    CHANGE MASTER TO MASTER_HOST='192.168.10.216',MASTER_USER='rep',MASTER_PASSWORD='geekwolf',MASTER_LOG_FILE='mysql-bin.000002',MASTER_LOG_POS=120,MASTER_AUTO_POSITION=0;
    
**[参考文档：](id:gg7)**

>[MYSQL 5.6 GTID-based Replication](http://dev.mysql.com/doc/refman/5.6/en/replication-gtids-restrictions.html)<br>
>[MYSQL 5.6 GTID模式下手工删除日志导致备库数据丢失](http://www.woqutech.com/?p=1108)<br>
>[How to create a new (or repair a broken) GTID based slave with Percona XtraBackup](http://www.mysqlperformanceblog.com/2013/05/09/how-to-create-a-new-or-repair-a-broken-gtid-based-slave-with-percona-xtrabackup/)<br>

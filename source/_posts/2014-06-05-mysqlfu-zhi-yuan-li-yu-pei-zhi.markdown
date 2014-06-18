---
layout: post
title: "MySQL复制原理与配置"
date: 2014-06-05 17:11:56 +0800
comments: true
categories: [Mysql数据库]
keywords: 异步复制,半同步复制,Binlog格式,MySQL Replication
description: MySQL复制原理、方式、Binlog格式优缺点以及复制遇到的问题
---

一、[Mysql复制基本原理](#r1)<br>
二、[Mysql复制中Binlog的三种格式](#r2)<br>
&emsp;&emsp;&emsp;&emsp;2.1 [三种格式的介绍](#r3)<br>
&emsp;&emsp;&emsp;&emsp;2.2 [Binlog格式的优缺点](#r4)<br>
&emsp;&emsp;&emsp;&emsp;2.3 [Binlog基本配置](#r5)<br>
三、[Mysql常见两种复制方式](#r6)<br>
&emsp;&emsp;&emsp;&emsp;3.1 [异步复制(Asynchronous Replication)](#r7)<br>
&emsp;&emsp;&emsp;&emsp;3.2 [半同步复制(Semi-synchroous Replicaion)](#r8)<br>
四、[提升主从复制性能的方法](#r9)<br>
五、[Mysql复制遇到的一些问题](#r10)<br>



**[一、Mysql复制基本原理](id:r1)**
 
![](http://geekwolf.github.io/images/mysql/replication/fzyl.png)

<!--more-->    

1. Mysql主库在事务提交时会将数据变更作为Events记录在Binlog中，Mysql主库的sync_binlog参数(默认值为0<br>
可参考[http://dev.mysql.com/doc/refman/5.6/en/replication-options-binary-log.html#sysvar_sync_binlog](http://dev.mysql.com/doc/refman/5.6/en/replication-options-binary-log.html#sysvar_sync_binlog))控制Binlog日志刷新到磁盘
2. 主库推送Binlog中的事件到从库的Relay Log，之后从库根据Relay Log重做DML操作
3. Mysql通过3个线程完成主从复制：Binlog Dump线程跑在主库，I/O线程和SQL线程跑在从库；<br>
&emsp;&emsp;当从库启动复制，首先创建I/O线程连接到主库，主库随后创建Binlog Dump线程读取数据库事件并发给I/O线程，I/O线程获取到事件数据后更新到从库的Relay Log中去，之后从库上的SQL线程读取Relay Log中更新的数据库事件并应用<br>

**注释：**<br>
 &emsp;&emsp;从库上两个重要文件：`master.info`：记录I/O线程连接主库的一些参数；`relay-log.info`：记录SQL线程应用Relay Log的一些参数<br>
**[二、Mysql复制中Binlog的三种格式](id:r2)**
**[2.1 三种格式的介绍](id:r3)**<br>
&emsp;&emsp;**Statement (statement-based replication:SBR)：**基于SQL语句级别的Binlog，每条修改数据的SQL都会保存在Binlog里面；<br>
&emsp;&emsp;**Row(RBR)：**基于行级别，记录每一行数据的变化，也就是将每行数据的变化都记录到Binlog里面，记录得非常详细，单并不记录原始SQL；在复制过程，并不会因为存储过程或者触发器造成主从数据不一致问题，但记录的binlog大小会比Statement格式大很多,CREATE、DROP、ALTER操作只记录原始SQL，而不会记录每行数据的变化到Binlog；<br>
&emsp;&emsp;**Mixed(MBR):**混合Statement和Row模式，默认是Statement模式记录，某些情况下会切换到Row模式，例如SQL中包含与时间、用户相关的函数等statement无法完成主从复制的操作；<br>
**[2.2 Binlog格式的优缺点](id:r4)**<br>
**基于Statement复制(Mysql5.5默认格式):**<br>
**优点：**<br>
&emsp;&emsp;Binlog日志量少，节约IO，和减少了主从网络binlog传输量<br>
&emsp;&emsp;只记录在master上所执行的语句的细节，以及执行语句的上下文信息<br>
&emsp;&emsp;同时，审计数据库变的更容易<br>

**缺点：**<br>
&emsp;&emsp;由于此格式是记录原始执行的SQL，保证能在slave上正确执行必须记录每条语句的上下文信息<br>
&emsp;&emsp;部分修改数据库时使用的函数可能出现无法复制：sleep()、last\_insert\_id()、 load\_file()、uuid()、user()、found\_rows()、sysdate()(除非启动时--sysdate-is-now=true)<br>
&emsp;&emsp;可能会导致触发器或者存储过程复制导致数据不一致，如调用NOW()函数<br>
&emsp;&emsp;INSERT...SELECT 可能会产生比RBR更多的行级锁，例如没有order by的insert...select<br>
&emsp;&emsp;复制需要执行全表扫描(WHERE中没有使用索引)的UPDATE时，需比row请求更多的行级锁<br>
&emsp;&emsp;对于AUTO\_INCREMENT字段的InnoDB引擎表，INSERT会阻塞其他INSERT语句 

**注：**如果statement不能保证主从正常复制,error日志会有提示：Statement may not be safe to log in statement format<br>

**基于Row复制:**<br>
**优点：**<br>
&emsp;&emsp;只记录每一行数据变化的细节，不需要记录上下文信息<br>
&emsp;&emsp;不会出现某些情况下auto_increment columns,timestamps,.triger、function、procedure无法正常复制的问题<br>
&emsp;&emsp;新的row格式已经有了优化， CREATE、DROP、ALTER操作只记录原始SQL，而不会记录每行数据的变化到Binlog<br>
&emsp;&emsp;适用于主从复制要求强一致性的环境<br>

**缺点：**<br>
&emsp;&emsp;update、delete、load data local infile等频繁更新或者删除大量行时会产生大量的binlog日志，会有一定的I/O压力，主从同步产生不必要的流量<br>
&emsp;&emsp;如：UPDATE products set status='sold' where product\_id BETWEEN 30000 and 50000;<br>
&emsp;&emsp;无法很好的进行数据库审计

**[2.3 Binlog基本配置](id:r5)**

修改配置文件my.cnf

    binlog_format=row                               binlog日志格式 
    max_binlog_size = 512M                          每个日志文件大小
    binlog_cache_size=1M                            二进制日志缓冲大小,uncommitted事务产生的日志写在cache，committed的持久化到磁盘binlog里面，此参数不是全局的，是针对session的
    expire_logs_days = 3                            binlog有效期
    log-bin=/datas/mysql/logs/mysql-bin             binlog日志目录
    relay-log=/datas/mysql/logs/relay-bin           从库中继日志目录
    #slave_skip_errors = all

**[三、Mysql常见两种复制方式](id:r6)**<br>
**[3.1 异步复制（Asynchronous Replication）](id:r7)**

 
![](http://geekwolf.github.io/images/mysql/replication/ybfz.png)

&emsp;&emsp;主库执行完Commit后，在主库写入Binlog日志后即可成功返回客户端，无需等等Binlog日志传送给从库

**异步复制主从配置：**<br>

主 : 192.168.10.216<br>
从 : 192.168.10.217<br>

**步骤：**主从版本一致-->主库授权复制帐号-->确保开启binlog及主从server_id唯一-->主库只读，记录主binlog名称及偏移量-->拷贝主数据文件到从相应位置-->从库change master to -->slave start-->检查两个yes

**1.主MySQL配置**<br>

    mysql>GRANT REPLICATION SLAVE ON *.* TO 'rep'@'192.168.10.217'  IDENTIFIED BY  'geekwolf';
    mysql>FLUSH TABLES WITH READ LOCK;
    mysql> SHOW MASTER STATUS;
    +------------------+----------+--------------+------------------+-------------------+
    | File | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------------+----------+--------------+------------------+-------------------+
    | mysql-bin.000003 | 120 | | | |
    +------------------+----------+--------------+------------------+-------------------+
    1 row in set (0.00 sec)
    将主库数据文件拷贝到从库对应目录
    mysql>UNLOCK TABLES;

**2.从MySQL配置**

    mysql>CHANGE MASTER TO MASTER_HOST='192.168.10.216',MASTER_USER='rep',MASTER_PASSWORD='geekwolf',MASTER_LOG_FILE='mysql-bin.000003',MASTER_LOG_POS=120;
    mysql>START  SLAVE;
    mysql> SHOW SLAVE STATUS \G;
    *************************** 1. row ***************************
       Slave_IO_State: Waiting for master to send event
      Master_Host: 192.168.10.216
      Master_User: rep
      Master_Port: 3306
    Connect_Retry: 1
      Master_Log_File: mysql-bin.000003
      Read_Master_Log_Pos: 120
       Relay_Log_File: relay-bin.000002
    Relay_Log_Pos: 283
    Relay_Master_Log_File: mysql-bin.000003
     Slave_IO_Running: Yes
    Slave_SQL_Running: Yes
      Replicate_Do_DB:
      Replicate_Ignore_DB:
       Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
      Replicate_Wild_Ignore_Table:
       Last_Errno: 0
       Last_Error:
     Skip_Counter: 0
      Exec_Master_Log_Pos: 120
      Relay_Log_Space: 450
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
      Master_UUID: bd2a4c6b-d954-11e3-8c0a-0200c0a80ad8
     Master_Info_File: /usr/local/mysql/data/master.info
    SQL_Delay: 0
      SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
       Master_Retry_Count: 86400
      Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
       Master_SSL_Crl:
       Master_SSL_Crlpath:
       Retrieved_Gtid_Set:
    Executed_Gtid_Set:
    Auto_Position: 0
    1 row in set (0.00 sec)

**注:**
异步复制中只要binlog不丢失即可保证数据的完整性；当主宕机，从库未收到binlog时，就会丢失数据（主磁盘正常时可以提取差异binlog在从执行），此时就需要用到半同步复制方式


**[3.2 半同步复制(Semi-synchroous Replicaion)](id:r8)**

![](http://geekwolf.github.io/images/mysql/replication/btb.png)

&emsp;&emsp;主库每次事务成功提交时并不及时反馈给前端，而是等待其中一个从库也接收到Binlog事务并成功写入Relay Log之后，才返回Commit操作成功给客户端；如此半同步就保证了事务成功提交后，至少有两份日志记录，一份在主库Binlog上，另一份在从库的Relay Log上，从而进一步保证数据完整性；半同步复制很大程度取决于主从网络RTT（往返时延）；以插件形式存在，

**半同步复制主从配置：**

主 : 192.168.10.216<br>
从 : 192.168.10.217

**1.判断是否支持动态增加插件**<br>

    mysql> select @@have_dynamic_loading;
    +------------------------+
    | @@have_dynamic_loading |
    +------------------------+
    | YES |
    +------------------------+
    

**2.检查是否存在半同步插件,分别在主从安装**<br>
/usr/local/mysql/lib/mysql/plugin/semisync\_master.so<br>
/usr/local/mysql/lib/mysql/plugin/semisync\_slave.so<br>

主MySQL上安装semisync_master.so:<br>
mysql>install plugin rpl\_semi\_sync\_master SONAME 'semisync_master.so'

从MySQL上安装semisync_slave.so:<br>
mysql>install plugin rpl\_semi\_sync\_slave SONAME 'semisync_slave.so'

安装后通过show plugins;查看安装的插件

**3.分别在主从打开semi-sync(默认关闭)**<br>
主：<br>
修改my.cnf<br>

    rpl_semi_sync_master_enabled=1
    rpl_semi_sync_master_timeout=30000(毫秒)   从库宕机或网络故障导致binlog没有及时传送到从库，此时主库上的事务需要等待的时间；此时间内没恢复，MySQL自动调整复制为异步复制模式
    mysql> set global rpl_semi_sync_master_enabled=1; 
    Query OK, 0 rows affected (0.00 sec)
    mysql> set global rpl_semi_sync_master_timeout=30000;
    Query OK, 0 rows affected (0.00 sec)
    
**从:**<br>
修改my.cnf<br>

    rpl_semi_sync_master_enabled=1
    mysql> set global rpl_semi_sync_master_enabled=1; 
    Query OK, 0 rows affected (0.00 sec)
    由于之前配置的复制是异步的，所以需要重启下从库I/O线程(或者直接重启主从stop slave;start slave;)：
    
    mysql> STOP SLAVE  IO_THREAD;
    Query OK, 0 rows affected (0.04 sec)
    mysql> START SLAVE  IO_THREAD;
    Query OK, 0 rows affected (0.00 sec)

**4.验证**<br>
**主：**

    mysql> show status like '%semi_sync%';

![](http://geekwolf.github.io/images/mysql/replication/semi.png)

    Rpl_semi_sync_master_clients: 值为2，表示有2个semi-sync的备库
    Rpl_semi_sync_master_net_avg_wait_time: 表示事务提交后，等待备库响应的平均时间
    Rpl_semi_sync_master_no_times: 表示有几次从半同步切换到异步复制
    Rpl_semi_sync_master_status : 值为ON，表示半同步复制处于打开状态
    Rpl_semi_sync_master_tx_avg_wait_time ：开启Semi-sync，事务返回需要等待的平均时间
    Rpl_semi_sync_master_wait_sessions：当前有几个线程在等备库响应
    Rpl_semi_sync_master_yes_tx : 值为1054，表示主库有1054个事务是通过半同步复制到从库
    Rpl_semi_sync_master_no_tx  : 值为0，表示当前有0个事务不是通过半同步模式同步到从库的

**从：**

    检查半同步是否开启
    show status like '%semi_sync%';
![](http://geekwolf.github.io/images/mysql/replication/semion.png)

    检查复制是否正常
    show slave status \G;

**[四、提升主从复制性能的方法](id:r9)**

**方案1：**多级主从架构，将不同库分开复制到不同从上

![](http://geekwolf.github.io/images/mysql/replication/fzjg.png)

**注意事项：**<br>
&emsp;&emsp;M2上打开log-slave-updates配置，保证M1传送的binlog能够被记录在M2的RelayLog和Binlog；M2可以选择BLACKHOLE引擎降低M2的I/O；并且Binlog日志的过滤可以在M2去做<br>
&emsp;&emsp;BLACKHOLE引擎的使用测试参考:
&emsp;&emsp;[http://jroller.com/dschneller/entry/mysql_replication_using_blackhole_engine](http://jroller.com/dschneller/entry/mysql_replication_using_blackhole_engine)<br>
&emsp;&emsp;[http://blog.csdn.net/kylinbl/article/details/8903336](http://blog.csdn.net/kylinbl/article/details/8903336)

**方案2：多线程复制（MySQL5.6+）**<br>
&emsp;&emsp;多线程复制是基于库的，允许从库并行更新，若单库压力大，此处的多线程复制没有意义；从库设置slave_parallel_workers=4表示MySQL从库在复制时启动4个SQL线程<br>
 &emsp;&emsp;MySQL5.6一下版本可以尝试Transfer补丁[http://dinglin.iteye.com/blog/1888640](http://dinglin.iteye.com/blog/1888640)<br>
**[五、Mysql复制遇到的一些问题](id:r10)**<br>
**1.指定特定的数据库或者表**

    replicate-do-db  告诉从服务器限制默认数据库(由USE所选择)为db_name的语句的复制，指定多个库时多次使用此参数，一次指定一个库，不能跨数据库更新；需要跨数据库进行更新，使用--replicate-wild-do-table=db_name.%
    比如：
    如果用--replicate-do-db=sales启动从服务器，并且在主服务器上执行下面的语句，UPDATE语句不会复制：
    USE prices; UPDATE sales.january SET amount=amount+1000;
    
    replicate-do-table  只复制某个表 ，支持跨库更新，指定多个表时多次使用此参数，一次指定一个表
    replicate-ignore-db告诉从服务器不要复制默认数据库(由USE所选择)为db_name的语句。要想忽略多个数据库，应多次使用该选项，每个数据库使用一次。如果正进行跨数据库更新并且不想复制这些更新，不应使用该选项。应使用--replicate-wild-ignore-table=db_name.%
    replicate-ignore-table 告诉从服务器线程不要复制更新指定表的任何语句(即使该语句可能更新其它的表)。要想忽略多个表，应多次使用该选项，每个表使用一次。同--replicate-ignore-db对比，该选项可以跨数据库进行更新
    
    replicate-wild-do-table  告诉从服务器线程限制复制更新的表匹配指定的数据库和表名模式的语句。模式可以包含‘%’和‘_’通配符，与LIKE模式匹配操作符具有相同的含义。要指定多个表，应多次使用该选项，每个表使用一次。该选项可以跨数据库进行更新。请读取该选项后面的注意事项。
    例如：--replicate-wild-do-table=foo%.bar%只复制数据库名以foo开始和表名以bar开始的表的更新。
    
    replicate-wild-ignore-table告诉从服务器线程不要复制表匹配给出的通配符模式的语句 
    
    从库增加(同步test库的bench1表，忽略同步mysql库所有表)：
    replicate-wild-do-table=test.bench1
    replicate-wild-ignore-table=mysql.%
    
**2.从库复制出错跳过**

    mysql>SET GLOBAL SQL_SLAVE_SKIP_COUNTER=1;

**3.log event entry exceeded max_allowed_packet的处理**

&emsp;&emsp;适当增加max\_allowed\_packet大小

**4.因主库大量滞后binlog，启动slave时，可能会跑满网卡带宽**

&emsp;&emsp;前段时间微博上@zolker遇到这类问题，网友也给了很多解决办法，趁此blog总结下<br>
&emsp;&emsp;A.级联备库方式，避免主MySQL网卡跑满影响<br>
&emsp;&emsp;B.脚本的方式每隔几秒(sleep)把io\_thread停一会，进行缓解 （这种方法简单、粗暴、有效,但有抖动）<br>
&emsp;&emsp;C.使用facebook的patch [https://github.com/facebook/mysql-5.6/commit/d3b0c7814090bded6563fee7d46d2ae41ed32a60](https://github.com/facebook/mysql-5.6/commit/d3b0c7814090bded6563fee7d46d2ae41ed32a60)<br>

&emsp;&emsp;以上是本人在学习过程中的笔记，一码一字敲出来的，有错误地方请留言，谢谢~转载请写明出处~

**参考文档：**
> http://www.ovaistariq.net/528/statement-based-vs-row-based-replication/<br>
> http://www.orczhou.com/index.php/2011/06/mysql-5-5-semi-sync-replication-setup-config/<br>
> http://www.linuxde.net/2013/09/15194.html<br>
> http://dev.mysql.com/doc/refman/5.6/en/replication.html<br>
>《深入浅出MySQL》

---
layout: post
title: "MHA高可用部署及测试"
date: 2014-05-17 10:41:14 +0800
comments: true
categories: [Mysql数据库]
keywords: MHA高可用部署及测试,MySQL高可用解决方案
description: MHA高可用部署及测试,MySQL高可用解决方案

---
[一、MHA特性](#t1)<br>
[二、MHA工作机制及failover过程解析](#t2)<br>
[三、MHA适用的主从架构](#t3)<br>
[四、MHA高可用环境的构建](#t4)<br>
&emsp;&emsp;&emsp;&emsp;4.1 [实验环境](#t5)<br>
&emsp;&emsp;&emsp;&emsp;4.2 [实验大概步骤](#t6)<br>
&emsp;&emsp;&emsp;&emsp;4.3 [相关脚本说明](#t7)<br>
&emsp;&emsp;&emsp;&emsp;4.4 [MHA部署过程](#t8)<br>
&emsp;&emsp;&emsp;&emsp;4.5 [配置VIP的方式](#t9)<br>
[五、MHA常用命令](#t10)<br>
[六、注意事项](#t11)<br>
[七、部署过程遇到的问题](#t12)<br>

#####[一.MHA特性](id:t1)

1.主服务器的自动监控和故障转移 <br>
&emsp;&emsp;MHA监控复制架构的主服务器，一旦检测到主服务器故障，就会自动进行故障转移。即使有些从服务器没有收到最新的relay log，MHA自动从最新的从服务器上识别差异的relay log并把这些日志应用到其他从服务器上，因此所有的从服务器保持一致性了。MHA通常在几秒内完成故障转移，9-12秒可以检测出主服务器故障，7-10秒内关闭故障的主服务器以避免脑裂，几秒中内应用差异的relay log到新的主服务器上，整个过程可以在10-30s内完成。还可以设置优先级指定其中的一台slave作为master的候选人。由于MHA在slaves之间修复一致性，因此可以将任何slave变成新的master，而不会发生一致性的问题，从而导致复制失败。<br>
2.交互式主服务器故障转移 <br>
<!--more-->
&emsp;&emsp;可以只使用MHA的故障转移，而不用于监控主服务器，当主服务器故障时，人工调用MHA来进行故障故障。<br>
3.非交互式的主故障转移 <br>
&emsp;&emsp;不监控主服务器，但自动实现故障转移。这种特征适用于已经使用其他软件来监控主服务器状态，比如heartbeat来检测主服务器故障和虚拟IP地址接管，可以使用MHA来实现故障转移和slave服务器晋级为master服务器。<br>
4.在线切换主服务器 <br>
&emsp;&emsp;在许多情况下，需要将现有的主服务器迁移到另外一台服务器上。比如主服务器硬件故障，RAID控制卡需要重建，将主服务器移到性能更好的服务器上等等。维护主服务器引起性能下降，导致停机时间至少无法写入数据。另外，阻塞或杀掉当前运行的会话会导致主主之间数据不一致的问题发生。MHA提供快速切换和优雅的阻塞写入，这个切换过程只需要0.5-2s的时间，这段时间内数据是无法写入的。在很多情况下，0.5-2s的阻塞写入是可以接受的。因此切换主服务器不需要计划分配维护时间窗口(呵呵，不需要你在夜黑风高时通宵达旦完成切换主服务器的任务)。<br>
<!--more-->
#####[二.MHA工作机制](id:t2)
MHA自动Failover过程解析<br>
http://www.mysqlsystems.com/2012/03/figure-out-process-of-autofailover-on-mha.html<br>
https://code.google.com/p/mysql-master-ha/wiki/Sequences_of_MHA

#####[三.MHA适用的主从架构](id:t3) <br>
https://code.google.com/p/mysql-master-ha/wiki/UseCases 

#####[四.MHA高可用环境的构建](id:t4)<br> 
#####[4.1 实验环境](id:t5)
![](http://geekwolf.github.io/images/mysql/ar.png)

 - Node1:192.168.10.216 (主) 
 - Node2:192.168.10.217 (从,主故障切换的备主) 
 - Node3:192.168.10.218 (从,兼MHA管理节点) 
 - VIP : 192.168.10.219 
 - Mysql:Percona-Server-5.6.16-rel64.2-569 
 - 以上节点系统均为CentOS6.5 x64

#####[4.2 实验大概步骤](id:t6) 
A. 三节点配置epel的yum源，安装相关依赖包<br>
B. 建立主从复制关系<br> 
C. ssh-keygen实现三台机器之间相互免密钥登录 <br>
D. 三节点安装mha4mysql-node-0.56,node3上安装mha4mysql-manager-0.56 <br>
E. 在node3上管理MHA配置文件<br> 
F. masterha_check_ssh验证ssh信任登录是否成功,masterha_check_repl验证mysql复制是否成功<br> 
G. 启动MHA manager，并监控日志文件<br> 
H. 测试master(Node1)的mysql宕掉后，是否会自动切换正常<br> 
I . 配置VIP，切换后从自动接管主服务，并对客户端透明<br>

#####[4.3 脚本相关说明](id:t7) 
MHA node有三个脚本，依赖perl模块<br> 
save_binary_logs：保存和拷贝宕掉的主服务器二进制日志 <br>
apply_diff_relay_logs:识别差异的relay log事件，并应用到所有从服务器节点 <br>purge_relay_logs:清除relay log日志文件<br>

#####[4.4 MHA部署过程](id:t8)

**A.**三节点配置epel的yum源，安装相关依赖包

    rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
    rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-6
    yum  -y install perl-DBD-MySQL  ncftp

**B.** 建立主从复制关系

在node1上：

    mysql>grant replication slave  on *.* to 'rep'@'192.168.10.%' identified by 'geekwolf';
    mysql>grant all on *.* to 'root'@'192.168.10.%' identified by 'geekwolf';
    mysql>show master status;

拷贝node1的data目录同步到node2，node3
在node2 node3上：

    mysql>change master  to  master_host='192.168.10.216', master_user='rep', master_password='geekwolf',master_port=3306, master_log_file='mysql-in.000006',master_log_pos=120,master_connect_retry=1;
    mysql>start slave;

每个节点都做好mysql命令的软链

` ln -s /usr/local/mysql/bin/* /usr/local/bin/ `

**C.** ssh-keygen实现三台机器之间相互免密钥登录 
在node1(在其他两个节点一同)执行 

    ssh-keygen -t rsa 
    ssh-copy-id -i /root/.ssh/id_rsa.pub root@node1 
    ssh-copy-id -i /root/.ssh/id_rsa.pub root@node2 
    ssh-copy-id -i /root/.ssh/id_rsa.pub root@node3

**D.** 三节点安装mha4mysql-node-0.56,node3上安装mha4mysql-manager-0.56 <br>
在node1 node2 node3安装mha4mysql-node <br>
wget https://googledrive.com/host/0B1lu97m8-haWeHdGWXp0YVVUSlk/mha4mysql-node-0.56.tar.gz<br> 
tar xf mha4mysql-node-0.56.tar.gz <br>
cd mha4mysql-node <br>
perl Makefile.PL <br>
make && make install<br>

在node3上安装mha4mysql-manager<br> 
wget https://googledrive.com/host/0B1lu97m8-haWeHdGWXp0YVVUSlk/mha4mysql-manager-0.56.tar.gz<br>
tar xf mha4mysql-manager-0.56.tar.gz <br>
cd mha4mysql-manager-0.56 <br>
yum install perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager perl-Config-IniFiles perl-Time-HiRes

**E.** 在node3上管理MHA配置文件 <br>
mkdir -p /etc/mha/{app1,scripts} <br>
cp mha4mysql-manager-0.56/samples/conf/* /etc/mha/ <br>
cp mha4mysql-manager-0.56/samples/scripts/* /etc/mha/scripts/ <br>
mv /etc/mha/app1.cnf /etc/mha/app1/ <br>
mv /etc/mha/masterha_default.cnf /etc/masterha_default.cnf<br>

设置全局配置： <br>
vim /etc/mha/masterha_default.cnf

    [server default]
    user=root
    password=geekwolf
    ssh_user=root
    repl_user=rep
    repl_password=geekwolf
    ping_interval=1
    #shutdown_script=""
    secondary_check_script = masterha_secondary_check -s node1 -s node2 -s node3 --user=root --master_host=node1 --master_ip=192.168.10.216 --master_port=3306
    #master_ip_failover_script="/etc/mha/scripts/master_ip_failover"
    #master_ip_online_change_script="/etc/mha/scripts/master_ip_online_change"
    # shutdown_script= /script/masterha/power_manager
    #report_script=""
vim /etc/mha/app1/app1.cnf

    [server default] 
    manager_workdir=/var/log/mha/app1
    manager_log=/var/log/mha/app1/manager.log
    [server1] 
    hostname=node1
    master_binlog_dir="/usr/local/mysql/logs"
    candidate_master=1
    [server2]
    hostname=node2
    master_binlog_dir="/usr/local/mysql/logs"
    candidate_master=1
    [server3]
    hostname=node3
    master_binlog_dir="/usr/local/mysql/logs"
    no_master=1

**注释：** <br>
&emsp;&emsp;candidate\_master=1 表示该主机优先可被选为new master，当多个[serverX]等设置此参数时，优先级由[serverX]配置的顺序决定 <br>
&emsp;&emsp;secondary\_check_script mha强烈建议有两个或多个网络线路检查MySQL主服务器的可用性。默认情况下,只有单一的路线 MHA Manager检查:从Manager to Master,但这是不可取的。MHA实际上可以有两个或两个以上的检查路线通过调用外部脚本定义二次检查脚本参数<br> 
&emsp;&emsp;master\_ip\_failover\_script 在MySQL从服务器提升为新的主服务器时，调用此脚本，因此可以将vip信息写到此配置文件 <br>
&emsp;&emsp;master\_ip\_online\_change\_script 使用masterha\_master\_switch命令手动切换MySQL主服务器时后会调用此脚本，参数和master_ip_failover\_script 类似，脚本可以互用 
&emsp;&emsp;shutdown\_script 此脚本(默认samples内的脚本)利用服务器的远程控制IDRAC等，使用ipmitool强制去关机，以避免fence设备重启主服务器，造成脑列现象 <br>
&emsp;&emsp;report\_script 当新主服务器切换完成以后通过此脚本发送邮件报告，可参考使用 [http://caspian.dotconf.net/menu/Software/SendEmail/sendEmail-v1.56.tar.gz](http://caspian.dotconf.net/menu/Software/SendEmail/sendEmail-v1.56.tar.gz)<br> 
&emsp;&emsp;以上涉及到的脚本可以从mha4mysql-manager-0.56/samples/scripts/*拷贝进行修改使用 <br>
&emsp;&emsp;其他manager详细配置参数[https://code.google.com/p/mysql-master-ha/wiki/Parameters](https://code.google.com/p/mysql-master-ha/wiki/Parameters)<br>

**F.** masterha\_check\_ssh验证ssh信任登录是否成功,masterha\_check\_repl验证mysql复制是否成功 <br>
验证ssh信任：masterha\_check\_ssh --conf=/etc/mha/app1/app1.cnf

    [root@localhost ~]# masterha_check_ssh --conf=/etc/mha/app1/app1.cnf
    Tue May 13 07:53:15 2014 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
    Tue May 13 07:53:15 2014 - [info] Reading application default configuration from /etc/mha/app1/app1.cnf..
    Tue May 13 07:53:15 2014 - [info] Reading server configuration from /etc/mha/app1/app1.cnf..
    Tue May 13 07:53:15 2014 - [info] Starting SSH connection tests..
    Tue May 13 07:53:16 2014 - [debug]
    Tue May 13 07:53:15 2014 - [debug] Connecting via SSH from root@node1(192.168.10.216:22) to root@node2(192.168.10.217:22)..
    Tue May 13 07:53:15 2014 - [debug] ok.
    Tue May 13 07:53:15 2014 - [debug] Connecting via SSH from root@node1(192.168.10.216:22) to root@node3(192.168.10.218:22)..
    Tue May 13 07:53:16 2014 - [debug] ok.
    Tue May 13 07:53:16 2014 - [debug]
    Tue May 13 07:53:16 2014 - [debug] Connecting via SSH from root@node2(192.168.10.217:22) to root@node1(192.168.10.216:22)..
    Tue May 13 07:53:16 2014 - [debug] ok.
    Tue May 13 07:53:16 2014 - [debug] Connecting via SSH from root@node2(192.168.10.217:22) to root@node3(192.168.10.218:22)..
    Tue May 13 07:53:16 2014 - [debug] ok.
    Tue May 13 07:53:17 2014 - [debug]
    Tue May 13 07:53:16 2014 - [debug] Connecting via SSH from root@node3(192.168.10.218:22) to root@node1(192.168.10.216:22)..
    Tue May 13 07:53:16 2014 - [debug] ok.
    Tue May 13 07:53:16 2014 - [debug] Connecting via SSH from root@node3(192.168.10.218:22) to root@node2(192.168.10.217:22)..
    Tue May 13 07:53:17 2014 - [debug] ok.
    Tue May 13 07:53:17 2014 - [info] All SSH connection tests passed successfully.
验证主从复制：masterha\_check\_repl --conf=/etc/mha/app1/app1.cnf

    [root@localhost mha]# masterha_check_repl --conf=/etc/mha/app1/app1.cnf
    Tue May 13 08:10:54 2014 - [info] Reading default configuration from /etc/masterha_default.cnf..
    Tue May 13 08:10:54 2014 - [info] Reading application default configuration from /etc/mha/app1/app1.cnf..
    Tue May 13 08:10:54 2014 - [info] Reading server configuration from /etc/mha/app1/app1.cnf..
    Tue May 13 08:10:54 2014 - [info] MHA::MasterMonitor version 0.56.
    Tue May 13 08:10:54 2014 - [info] GTID failover mode = 0
    Tue May 13 08:10:54 2014 - [info] Dead Servers:
    Tue May 13 08:10:54 2014 - [info] Alive Servers:
    Tue May 13 08:10:54 2014 - [info] node1(192.168.10.216:3306)
    Tue May 13 08:10:54 2014 - [info] node2(192.168.10.217:3306)
    Tue May 13 08:10:54 2014 - [info] node3(192.168.10.218:3306)
    Tue May 13 08:10:54 2014 - [info] Alive Slaves:
    Tue May 13 08:10:54 2014 - [info] node2(192.168.10.217:3306) Version=5.6.16-64.2-rel64.2-log (oldest major version between slaves) log-bin:enabled
    Tue May 13 08:10:54 2014 - [info] Replicating from 192.168.10.216(192.168.10.216:3306)
    Tue May 13 08:10:54 2014 - [info] Primary candidate for the new Master (candidate_master is set)
    Tue May 13 08:10:54 2014 - [info] node3(192.168.10.218:3306) Version=5.6.16-64.2-rel64.2-log (oldest major version between slaves) log-bin:enabled
    Tue May 13 08:10:54 2014 - [info] Replicating from 192.168.10.216(192.168.10.216:3306)
    Tue May 13 08:10:54 2014 - [info] Not candidate for the new Master (no_master is set)
    Tue May 13 08:10:54 2014 - [info] Current Alive Master: node1(192.168.10.216:3306)
    Tue May 13 08:10:54 2014 - [info] Checking slave configurations..
    Tue May 13 08:10:54 2014 - [info] read_only=1 is not set on slave node2(192.168.10.217:3306).
    Tue May 13 08:10:54 2014 - [warning] relay_log_purge=0 is not set on slave node2(192.168.10.217:3306).
    Tue May 13 08:10:54 2014 - [info] read_only=1 is not set on slave node3(192.168.10.218:3306).
    Tue May 13 08:10:54 2014 - [warning] relay_log_purge=0 is not set on slave node3(192.168.10.218:3306).
    Tue May 13 08:10:54 2014 - [info] Checking replication filtering settings..
    Tue May 13 08:10:54 2014 - [info] binlog_do_db= , binlog_ignore_db=
    Tue May 13 08:10:54 2014 - [info] Replication filtering check ok.
    Tue May 13 08:10:54 2014 - [info] GTID (with auto-pos) is not supported
    Tue May 13 08:10:54 2014 - [info] Starting SSH connection tests..
    Tue May 13 08:10:55 2014 - [info] All SSH connection tests passed successfully.
    Tue May 13 08:10:55 2014 - [info] Checking MHA Node version..
    Tue May 13 08:10:55 2014 - [info] Version check ok.
    Tue May 13 08:10:55 2014 - [info] Checking SSH publickey authentication settings on the current master..
    Tue May 13 08:10:56 2014 - [info] HealthCheck: SSH to node1 is reachable.
    Tue May 13 08:10:56 2014 - [info] Master MHA Node version is 0.56.
    Tue May 13 08:10:56 2014 - [info] Checking recovery script configurations on node1(192.168.10.216:3306)..
    Tue May 13 08:10:56 2014 - [info] Executing command: save_binary_logs --command=test --start_pos=4 --binlog_dir=/usr/local/mysql/logs --output_file=/var/tmp/save_binary_logs_test --manager_version=0.56 --start_file=mysql-bin.000009
    Tue May 13 08:10:56 2014 - [info] Connecting to root@192.168.10.216(node1:22)..
      Creating /var/tmp if not exists.. ok.
      Checking output directory is accessible or not..
       ok.
      Binlog found at /usr/local/mysql/logs, up to mysql-bin.000009
    Tue May 13 08:10:56 2014 - [info] Binlog setting check done.
    Tue May 13 08:10:56 2014 - [info] Checking SSH publickey authentication and checking recovery script configurations on all alive slave servers..
    Tue May 13 08:10:56 2014 - [info] Executing command : apply_diff_relay_logs --command=test --slave_user='root' --slave_host=node2 --slave_ip=192.168.10.217 --slave_port=3306 --workdir=/var/tmp --target_version=5.6.16-64.2-rel64.2-log --manager_version=0.56 --relay_log_info=/usr/local/mysql/data/relay-log.info --relay_dir=/usr/local/mysql/data/ --slave_pass=xxx
    Tue May 13 08:10:56 2014 - [info] Connecting to root@192.168.10.217(node2:22)..
      Checking slave recovery environment settings..
        Opening /usr/local/mysql/data/relay-log.info ... ok.
        Relay log found at /usr/local/mysql/logs, up to relay-bin.000006
        Temporary relay log file is /usr/local/mysql/logs/relay-bin.000006
        Testing mysql connection and privileges..Warning: Using a password on the command line interface can be insecure.
     done.
        Testing mysqlbinlog output.. done.
        Cleaning up test file(s).. done.
    Tue May 13 08:10:57 2014 - [info] Executing command : apply_diff_relay_logs --command=test --slave_user='root' --slave_host=node3 --slave_ip=192.168.10.218 --slave_port=3306 --workdir=/var/tmp --target_version=5.6.16-64.2-rel64.2-log --manager_version=0.56 --relay_log_info=/usr/local/mysql/data/relay-log.info --relay_dir=/usr/local/mysql/data/ --slave_pass=xxx
    Tue May 13 08:10:57 2014 - [info] Connecting to root@192.168.10.218(node3:22)..
      Checking slave recovery environment settings..
        Opening /usr/local/mysql/data/relay-log.info ... ok.
        Relay log found at /usr/local/mysql/logs, up to relay-bin.000006
        Temporary relay log file is /usr/local/mysql/logs/relay-bin.000006
        Testing mysql connection and privileges..Warning: Using a password on the command line interface can be insecure.
     done.
        Testing mysqlbinlog output.. done.
        Cleaning up test file(s).. done.
    Tue May 13 08:10:57 2014 - [info] Slaves settings check done.
    Tue May 13 08:10:57 2014 - [info]
    node1(192.168.10.216:3306) (current master)
     +--node2(192.168.10.217:3306)
     +--node3(192.168.10.218:3306)
    Tue May 13 08:10:57 2014 - [info] Checking replication health on node2..
    Tue May 13 08:10:57 2014 - [info] ok.
    Tue May 13 08:10:57 2014 - [info] Checking replication health on node3..
    Tue May 13 08:10:57 2014 - [info] ok.
    Tue May 13 08:10:57 2014 - [warning] master_ip_failover_script is not defined.
    Tue May 13 08:10:57 2014 - [warning] shutdown_script is not defined.
    Tue May 13 08:10:57 2014 - [info] Got exit code 0 (Not master dead).
    MySQL Replication Health is OK.

**G.** 启动MHA manager，并监控日志文件<br> 
在node1上killall mysqld的同时在node3上启动manager服务

    [root@localhost mha]# masterha_manager --conf=/etc/mha/app1/app1.cnf
    Tue May 13 08:19:01 2014 - [info] Reading default configuration from /etc/masterha_default.cnf..
    Tue May 13 08:19:01 2014 - [info] Reading application default configuration from /etc/mha/app1/app1.cnf..
    Tue May 13 08:19:01 2014 - [info] Reading server configuration from /etc/mha/app1/app1.cnf..
      Creating /var/tmp if not exists.. ok.
      Checking output directory is accessible or not..
       ok.
      Binlog found at /usr/local/mysql/logs, up to mysql-bin.000009
    Tue May 13 08:19:18 2014 - [info] Reading default configuration from /etc/masterha_default.cnf..
    Tue May 13 08:19:18 2014 - [info] Reading application default configuration from /etc/mha/app1/app1.cnf..
    Tue May 13 08:19:18 2014 - [info] Reading server configuration from /etc/mha/app1/app1.cnf..

&emsp;&emsp;之后观察node3上/var/log/mha/app1/manager.log日志会发现node1 dead状态，主自动切换到node2上，而node3上的主从配置指向了node2，并且发生一次切换后会生成/var/log/mha/app1/app1.failover.complete文件；<br>
**手动恢复node1操作：** <br>
&emsp;&emsp;rm -rf /var/log/mha/app1/app1.failover.complete<br>
&emsp;&emsp;启动node1上的mysql，重新配置node2 node3 主从指向node1（change master to）<br>
**MHA Manager后台执行：** <br>
nohup masterha\_manager --conf=/etc/mha/app1/app1.cnf < /dev/null > /var/log/mha/app1/app1.log 2>&1 & <br>
守护进程方式参考： 
https://code.google.com/p/mysql-master-ha/wiki/Runnning_Background<br>
ftp://ftp.pbone.net/mirror/ftp5.gwdg.de/pub/opensuse/repositories/home:/weberho:/qmailtoaster/openSUSE_Tumbleweed/x86_64/daemontools-0.76-5.3.x86_64.rpm

#####[4.5 配置VIP的方式](id:t9) 
**A.**通过全局配置文件实现 <br>
vim /etc/mha/masterha_default.cnf
  

    [server default]
      user=root
      password=geekwolf
      ssh_user=root
      repl_user=rep
      repl_password=geekwolf
      ping_interval=1
      secondary_check_script = masterha_secondary_check -s node1 -s node2 -s node3 --user=root --master_host=node1 --master_ip=192.168.10.216 --master_port=3306
      master_ip_failover_script="/etc/mha/scripts/master_ip_failover"
      master_ip_online_change_script="/etc/mha/scripts/master_ip_online_change"
      #shutdown_script= /script/masterha/power_manager
      #report_script=""

修改后的master\_ip\_failover、master\_ip\_online\_change脚本

    #!/usr/bin/env perl
    use strict;
    use warnings FATAL => 'all';
    use Getopt::Long;
    my (
        $command, $ssh_user, $orig_master_host, $orig_master_ip,
        $orig_master_port, $new_master_host, $new_master_ip, $new_master_port
    );
    my $vip = '192.168.10.218'; # Virtual IP
    my $gateway = '192.168.10.1';#Gateway IP
    my $interface = 'eth0'
    my $key = "1";
    my $ssh_start_vip = "/sbin/ifconfig $interface:$key $vip;/sbin/arping -I $interface -c 3 -s $vip $gateway >/dev/null 2>&1";
    my $ssh_stop_vip = "/sbin/ifconfig $interface:$key down";
    GetOptions(
        'command=s' => \$command,
        'ssh_user=s' => \$ssh_user,
        'orig_master_host=s' => \$orig_master_host,
        'orig_master_ip=s' => \$orig_master_ip,
        'orig_master_port=i' => \$orig_master_port,
        'new_master_host=s' => \$new_master_host,
        'new_master_ip=s' => \$new_master_ip,
        'new_master_port=i' => \$new_master_port,
    );
    exit &main();
    sub main {
        print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";
        if ( $command eq "stop" || $command eq "stopssh" ) {
            # $orig_master_host, $orig_master_ip, $orig_master_port are passed.
            # If you manage master ip address at global catalog database,
            # invalidate orig_master_ip here.
            my $exit_code = 1;
            eval {
                print "Disabling the VIP on old master: $orig_master_host \n";
                &stop_vip();
                $exit_code = 0;
            };
            if ($@) {
                warn "Got Error: $@\n";
                exit $exit_code;
            }
            exit $exit_code;
        }
        elsif ( $command eq "start" ) {
            # all arguments are passed.
            # If you manage master ip address at global catalog database,
            # activate new_master_ip here.
            # You can also grant write access (create user, set read_only=0, etc) here.
            my $exit_code = 10;
            eval {
                print "Enabling the VIP - $vip on the new master - $new_master_host \n";
                &start_vip();
                $exit_code = 0;
            };
            if ($@) {
                warn $@;
                exit $exit_code;
            }
            exit $exit_code;
        }
        elsif ( $command eq "status" ) {
            print "Checking the Status of the script.. OK \n";
            `ssh $ssh_user\@cluster1 \" $ssh_start_vip \"`;
            exit 0;
        }
        else {
            &usage();
            exit 1;
        }
    }
    # A simple system call that enable the VIP on the new master
    sub start_vip() {
        `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
    }
    # A simple system call that disable the VIP on the old_master
    sub stop_vip() {
        `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
    }
    sub usage {
        print
        "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
    }

**B.**通过第三方HA（keepalived、heartbeat）实现VIP，以keepalived为例 <br>
以node1 node2互为主备进行配置keepalived <br>
在node1 node2上分别下载安装keepalived <br>
wget [http://www.keepalived.org/software/keepalived-1.2.13.tar.gz](http://www.keepalived.org/software/keepalived-1.2.13.tar.gz) <br>
yum -y install popt-* <br>
./configure --prefix=/usr/local/keepalived --enable-snmp <br>
make && make install <br>
cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/rc.d/init.d/ <br>
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/<br> 
chmod +x /etc/rc.d/init.d/keepalived <br>
chkconfig keepalived on <br>
mkdir /etc/keepalived <br>
ln -s /usr/local/keepalived/sbin/keepalived /usr/sbin<br>

修改node1(192.168.10.216)配置文件 <br>
vim /etc/keepalived/keepalived.conf

    ! Configuration File for keepalived
    global_defs {
     router_id MHA 
     notification_email {
     root@localhost   #接收邮件，可以有多个，一行一个
    }
     #当主、备份设备发生改变时，通过邮件通知
     notification_email_from  m@localhost
     #发送邮箱服务器
     smtp_server 127.0.0.1
     #发送邮箱超时时间
     smtp_connect_timeout 30
     }
    
    varrp_script check_mysql {
         script "/etc/keepalived/check_mysql.sh"
    }
    vrrp_sync_group VG1 {
        group {
              VI_1
        }
    notify_master "/etc/keepalived/master.sh"
    }
    
    vrrp_instance VI_1 {
         state master     
         interface eth0   
         virtual_router_id 110
         priority 100            
         advert_int 1
         nopreempt #不抢占资源，意思就是它活了之后也不会再把主抢回来
    
         authentication {
         # 认证方式，可以是PASS或AH两种认证方式
         auth_type PASS
         # 认证密码
         auth_pass geekwolf
         }
    track_script {
         check_mysql
    }
    virtual_ipaddress {
         192.168.10.219
         }
    
    }
修改node2(192.168.10.217)配置文件 <br>
vim /etc/keepalived/keepalived.conf

    ! Configuration File for keepalived
    global_defs {
     router_id MHA 
     notification_email {
     root@localhost   #接收邮件，可以有多个，一行一个
    }
     #当主、备份设备发生改变时，通过邮件通知
     notification_email_from  m@localhost
     #发送邮箱服务器
     smtp_server 127.0.0.1
     #发送邮箱超时时间
     smtp_connect_timeout 30
     }
    
    varrp_script check_mysql {
         script "/etc/keepalived/check_mysql.sh"
    }
    vrrp_sync_group VG1 {
        group {
              VI_1
        }
    notify_master "/etc/keepalived/master.sh"
    }
    vrrp_instance VI_1 {
         state backup    
         interface eth0    
         virtual_router_id 110
         priority 99            
         advert_int 1
    
         authentication {
         # 认证方式，可以是PASS或AH两种认证方式
         auth_type PASS
         # 认证密码
         auth_pass geekwolf
         }
    track_script {
         check_mysql
    }
    virtual_ipaddress {
         192.168.10.219
         }
    
    }

check_mysql.sh

    #!/bin/bash
    MYSQL=/usr/local/mysql/bin/mysql
    MYSQL_HOST=127.0.0.1
    MYSQL_USER=root
    MYSQL_PASSWORD=geekwolf
    CHECK_TIME=3
    #mysql  is working MYSQL_OK is 1 , mysql down MYSQL_OK is 0
    MYSQL_OK=1
    function check_mysql_helth (){
    $MYSQL -h $MYSQL_HOST -u $MYSQL_USER -e "show status;" >/dev/null 2>&1
    if [ $? = 0 ] ;then
         MYSQL_OK=1
    else
         MYSQL_OK=0
    fi
         return $MYSQL_OK
    }
    while [ $CHECK_TIME -ne 0 ]
    do
         let "CHECK_TIME -= 1"
         check_mysql_helth
    if [ $MYSQL_OK = 1 ] ; then
         CHECK_TIME=0
         exit 0
    fi
    if [ $MYSQL_OK -eq 0 ] &&  [ $CHECK_TIME -eq 0 ]
    then
         pkill keepalived
    exit 1
    fi
    sleep 1
    done
master.sh

    #!/bin/bash
    VIP=192.168.10.219
    GATEWAY=1.1
    /sbin/arping -I eth0 -c 5 -s $VIP $GATEWAY &>/dev/null

chmod +x /etc/keepalived/check_mysql.sh <br>
chmod +x /etc/keepalived/master.sh

#####[五.MHA常用命令](id:t10)

查看manager状态 <br>
masterha\_check\_status --conf=/etc/mha/app1/app1.cnf

查看免密钥是否正常 <br>
masterha\_check\_ssh --conf=/etc/mha/app1/app1.cnf

查看主从复制是否正常 <br>
masterha\_check\_repl --conf=/etc/mha/app1/app1.cnf

添加新节点server4到配置文件 <br>
masterha\_conf\_host --command=add --conf=/etc/mha/app1/app1.cnf --hostname=geekwolf --block=server4 --params="no\_master=1;ignore\_fail=1" 
删除server4节点 <br>
masterha\_conf\_host --command=delete --conf=/etc/mha/app1/app1.cnf --block=server4

**注：** <br>
block:为节点区名，默认值 为[server\_$hostname],如果设置成block=100，则为[server100] 
params:参数，分号隔开(参考https://code.google.com/p/mysql-master-ha/wiki/Parameters)

关闭manager服务 <br>
masterha\_stop --conf=/etc/mha/app1/app1.cnf

主手动切换(前提不要启动masterha\_manager服务) <br>
在主node1存活情况下进行切换 <br>
交互模式： <br>
masterha\_master\_switch --master\_state=alive --conf=/etc/mha/app1/app1.cnf --new_master\_host=node2 <br>
非交互模式： <br>
masterha\_master\_switch --master\_state=alive --conf=/etc/mha/app1/app1.cnf --new_master_host=node2 --interactive=0 <br>
在主node1宕掉情况下进行切换 <br>
masterha\_master\_switch --master\_state=dead --conf=/etc/mha/app1/app1.cnf --dead\_master\_host=node1 --dead\_master\_ip=192.168.10.216 --dead\_master\_port=3306 --new\_master\_host=192.168.10.217 
详细请参考:https://code.google.com/p/mysql-master-ha/wiki/TableOfContents?tm=6
*
#####[六.注意事项](id:t11) <br>
**A.** 以上两种vip切换方式，建议采用第一种方法 <br>
**B.** 发生主备切换后，manager服务会自动停掉，且在/var/log/mha/app1下面生成<br>app1.failover.complete，若再次发生切换需要删除app1.failover.complete文件<br> 
**C.** 测试过程发现一主两从的架构(两从都设置可以担任主角色candidate\_master=1)，当旧主故障迁移到备主后，删除app1.failover.complete，再次启动manager，停掉新主后，发现无法正常切换(解决方式：删除/etc/mha/app1/app1.cnf里面的旧主node1的信息后，重新切换正常) <br>
**D.** arp缓存导致切换VIP后，无法使用问题 <br>
**E.** 使用Semi-Sync能够最大程度保证数据安全<br>
**F.** Purge_relay_logs脚本删除中继日志不会阻塞SQL线程，在每台从节点上设置计划任务定期清除中继日志<br>
&emsp;&emsp;0 5 * * * root /usr/bin/purge_relay_logs --user=root --password=geekwolf --disable_relay_log_purge >> /var/log/mha/purge_relay_logs.log 2>&1

#####[七.部署过程遇到的问题](id:t12)

**问题1：** 
[root@node1 mha4mysql-node-0.56]# perl Makefile.PL <br>
Can't locate ExtUtils/MakeMaker.pm in @INC (@INC contains: inc /usr/local/lib64/perl5 /usr/local/share/perl5 /usr/lib64/perl5/vendor_perl /usr/share/perl5/vendor_perl /usr/lib64/perl5 /usr/share/perl5 .) at inc/Module/Install/Makefile.pm line 4.  <br>
BEGIN failed--compilation aborted at inc/Module/Install/Makefile.pm line 4. 
Compilation failed in require at inc/Module/Install.pm line 283.  <br>
Can't locate ExtUtils/MakeMaker.pm in @INC (@INC contains: inc /usr/local/lib64/perl5 /usr/local/share/perl5 /usr/lib64/perl5/vendor_perl /usr/share/perl5/ <br>vendor_perl /usr/lib64/perl5 /usr/share/perl5 .) at inc/Module/Install/Can.pm line 6.  <br>
BEGIN failed--compilation aborted at inc/Module/Install/Can.pm line 6.  <br>
Compilation failed in require at inc/Module/Install.pm line 283.  <br>
Can't locate ExtUtils/MM_Unix.pm in @INC (@INC contains: inc /usr/local/lib64/ <br>perl5 /usr/local/share/perl5 /usr/lib64/perl5/vendor_perl /usr/share/perl5/vendor_perl /usr/lib64/perl5 /usr/share/perl5 .) at inc/Module/Install/ <br>Metadata.pm line 349.  <br>
**解决办法：**  <br>
yum -y install perl-CPAN perl-devel perl-DBD-MySQL

**问题2：**  <br>
Can't locate Time/HiRes.pm in @INC (@INC contains: /usr/local/lib64/perl5 /usr/local/share/perl5 /usr/lib64/perl5/vendor_perl /usr/share/perl5/vendor_perl /usr/lib64/perl5 /usr/share/perl5 .) at /usr/local/share/perl5/MHA/SSHCheck.pm line 28.  <br>
BEGIN failed--compilation aborted at /usr/local/share/perl5/MHA/SSHCheck.pm line 28.  <br>
Compilation failed in require at /usr/local/bin/masterha_check_ssh line 25. 
BEGIN failed--compilation aborted at /usr/local/bin/masterha_check_ssh line 25. <br> 
**解决办法**：  <br>
yum -y install perl-Time-HiRes

**问题3：** 
![](http://geekwolf.github.io/images/mysql/mhaq.jpg)
**解决办法:** <br>
每个节点都做好mysql命令的软链 <br>
ln -s /usr/local/mysql/bin/* /usr/local/bin/<br>

**参考文档：**

> https://code.google.com/p/mysql-master-ha <br>
> http://blog.chinaunix.net/uid-28437434-id-3476641.html



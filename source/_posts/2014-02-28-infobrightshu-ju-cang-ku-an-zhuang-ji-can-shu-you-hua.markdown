---
layout: post
title: "Infobright数据仓库安装及参数优化"
date: 2014-02-28 20:30:10 +0800
comments: true
categories: Infobright数据仓库
keywords: infobright,数据仓库,warehouse
description : infobright安装部署及参数调整优化
---

#####一、认知Infobright数据仓库
1. Infobright是开源的DATA Warehouse，可以作为Mysql的存储引擎（BRIGHTHOUSE）使用，select查询与Mysql无区别
2. 适用于基于Mysql架构下的OLAP，区分为ICE（社区版免费）和IEE（企业版商业授权）
3. 千万级的查询性能比MyISAM、InnoDB等快5-60倍，且数据量很大时，查询性能基本在同一个数量级，适合聚合查询（SUM,COUNT，AVG，GROUP BY）
4. 列式存储，无需建索引和分区，高压缩比40:1（官方数字）
5. ICE和IEE版本的区别和限制<br>
   https://www.infobright.org/index.php/Learn-More/ICE_IEE_Comparison/

#####二、Infobright基本安装初始化
     wget http://www.infobright.org/downloads/ice/infobright-4.0.7-0-x86_64-ice.rpm
     rpm -ivh infobright-4.0.7-0-x86_64-ice.rpm --prefix=/usr/local/
      
     A.初始化数据库：
     /usr/local/infobright/scripts/mysql_install_db --datadir=/data/data/ --basedir=/usr/local/infobright-4.0.7-x86_64/ --force
       安装之后会生成
      /etc/my-ib.cnf.inactive （主配置文件，下面两个暂不讨论）
<!--more-->
      /etc/my-ib-master.cnf
      /etc/my-ib-slave.cnf
      /etc/rc.d/init.d/mysqld-ib（启动脚本）
      mv  /etc/my-ib.cnf.inactive /etc/my-ib.cnf
      修改/etc/rc.d/init.d/mysqld-ib /etc/my-ib.cnf  的datadir basedir参数到指定位置
      修改权限chown mysql.mysql /etc/my-ib*  /data/data /ibcache/cachedata

     B.修改warehouse引擎配置文件
      /data/data/brighthouse.ini

     C.参数优化
	   CacheFolder = /ibcache/cachedata/
	   ServerMainHeapSize = 28000
	   ServerCompressedHeapSize = 4000
       LoaderMainHeapSize = 800
       ControlMessages = 3
       KNFolder = BH_RSI_Repository
       AllowMySQLQueryPath = 0

**注释：**<br>
CacheFolder 临时数据目录，用于缓存处理查询的中间结果集，与Datadir相异为宜，可用空间大于20G<br>
ServerMainHeapSize IB主线程内存，一般设置为物理内存一半；若可能尽量增加<br>
ServerCompressedHeapSize  服务进程的压缩堆栈空间，存放压缩数据<br>
LoaderMainHeapSize Bhloader数据导入缓冲区，随目标表的列数增加而调整，loader进程的堆栈空间，一般最大不超过800M<br>
ControlMessages  控制盒查询日志的信息量级别（1-3之间）<br>
KNFolder  知识网络目录，默认在datadir目录下<br>
AllowMySQLQueryPath  是否支持Mysql原生的SQL查询，支持修改为1，否则0<br>
启动infobright：service mysqld-ib start

**ICE版本导入数据方式：<br>**
BRIGHTHOUSE<br>
load data infile '/data/data.txt' into table t fields terminated by ' ';

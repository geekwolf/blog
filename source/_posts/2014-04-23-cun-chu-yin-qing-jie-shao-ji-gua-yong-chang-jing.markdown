---
layout: post
title: "存储引擎介绍及适用场景"
date: 2014-04-23 14:05:56 +0800
comments: true
categories: [Mysql数据库]
keywords: Innodb、MyISAM、TokuDB、Memory、引擎
description: 储引擎介绍及适用场景
---

查看当前的默认存储引擎<br>
`   show variables  like 'table_type';`

查看当前数据库支持的引擎<br>
`    show engines \G;`<br>
`    show variables like 'have%';`
<!--more-->
创建表时指定存储引擎<br>
`    create table ai (i bigint(20) not null auto_increment,primary key(i));`<br>

修改表引擎<br>
`   alter table ai engine=innodb;`

**常见Mysql数据库引擎对比：**

![](/images/mysql/engines.png)
#####一、MyISAM引擎特点：<br>

在磁盘存储成3个文件，文件名和表名一样，但扩展名分别为：<br>
.frm        (存储表定义)<br>
.MYD     （MYData，存储数据）<br>
.MYI       （MYIndex，存储索引）<br>
其中数据文件和索引文件可以分开在不同目录，平均分布IO

创建表时指定数据和索引路径：<br>

`create table geekwolf (id int,c varchar(10)) data directory='/data/data/' index directory='/data/index' engine='MyISAM';`

MyISAM的表可能出现损坏的解决办法：<br>
check table geekwolf；检查表的健康情况<br>
repair table geekwolf；修改表<br>

MyISAM的表引擎支持3种不同的存储格式：<br>
**静态表**（默认格式，固定长度，存储时按照列宽度定义补足空格；在查询时会丢失尾部的空格）<br>
**动态表**（频繁更新删除记录会产生碎片，占用空间相对较少，需要定期执行optimize table 或myisamchk -r来改善性能）<br>
**压缩表** （由myisampack工具创建）<br>

**适用场景：**<br>
以读操作和插入操作为主，只有很少的更新和删除操作，并且对事务的完整性，并发性要求不是很高的场景

#####二、INNODB引擎特点：<br>
**1.自动增长列：**<br>
    InnoDB表，自动增长列必须是索引，如果是组合索引，必须是组合索引的第一列；<br>
    对于MyISAM引擎表，自动增长列可以是组合索引的其他列，这样插入记录后，自动增长列是按照组合索引的前面激烈进行排序后递增的<br>
    创建MyISAM表autoincre_demo<br>

`     create table autoincre_demo (d1 smallint not null auto_increment,d2 smallint not null,name varchar(10),index(d2.d1)) engine=myisam; `

如图所示：自动增长列是d1作为组合索引的第二列,插入记录后，发现增长列是按照组合索引的第一列d2进行排序后递增的

![](/images/mysql/autoincre.png)<br>

**2.外键约束：**
  Innodb引擎支持外键，在创建外键时，要求父表必须有对应的索引，子表在创建外键的时候也会自动创建对应的索引<br>
  外键信息可以通过show table status like 'test' \G;  show create table 'test';<br>

**3.存储方式：**<br>
**A.** 使用共享表空间，这种方式创建的表结果保存在.frm文件，数据和索引保存在innodb\_data\_home\_dir innodb\_data\_file\_path定义的表空间中<br>
**B.**多表空间存储，表结构保存在.frm文件中，但是每个表的数据和索引单独存放在.ibd中；每个分区对于单独的.ibd<br>
   （需要开启innodb_file_per_table=1）<br>
 对于使用多表空间的表可以方便进行单表备份恢复，简单复制ibd和frm文件的方法因没有共享表空间的字典信息，而无法使用；多表空间情况，因为Innodb把内部的数据字典和在线重做日志存放在共享表空间里面

**使用此语句删除.ibd文件：**<br>
`ALTER TABLE tbl_name DISCARD TABLESPACE;`<br>
要把备份的.ibd文件还原到表中，需把此文件复制到数据库目录中，然后书写此语句：<br>
`ALTER TABLE tbl_name IMPORT TABLESPACE;`

**适用场景：**<br>
需要事务处理，对事务的完整性要求高，并发条件下要求数据一致性的计费系统或者财务系统等对数据准心要求比较搞的系统（5.5+默认引擎）

#####三、MEMORY引擎：
 **A.**每个MEMORY表实际对应一个磁盘文件.frm，数据存放在内存，默认采用HASH索引（也可以设置撑Btree索引），服务关闭数据会丢失<br>
 **B.**是否memory表中的内存可以通过delete from 或者truncate 或者drop table<br>
**C.**memory表可以放置数据量的大小受到max_heap_table_size变量约束，默认16M，在定义表时可以用MAX_ROWS指定表的最大行数<br>
**D.**使用环境：用于内容变化不频繁或者作为统计操作的中间结果表

**适用场景：**<br>
一般用于更新不太频繁的小表，用以快速得到访问结果的环境，但对表大小有限制


#####四、TOKUDB引擎：<br>
   具有高压缩率高效的插入性能，支持大多数在线DDL<br>
   与InnoDB引擎对比测试：http://www.tokutek.com/resources/tokudb-vs-innodb/<br>
   **特性：**<br>
    使用Fractal树索引保证了高效的插入性能
    优秀的压缩特性，比InnoDB高近10倍
    Hot Schema Changes特性支持在线创建索引和添加/删除属性列等DDL操作
    使用Bulk Loader达到快速加载大数据量
    提供主从延迟消除技术
    支持ACID和MVCC

**适用场景：**<br>
日志数据，日志通常插入频繁切存储量大；<br>
历史数据，通常不会再有写操作，可以利用TokuDB的高压缩特性存储；<br>
在线DDL较频繁的场景，使用TokuDB可以大大增加系统可用性；

**注：**
具体使用哪种引擎要根据自己的业务的特点去决定

---
layout: post
title: "初认识Puppet"
date: 2014-06-07 16:25:41 +0800
comments: true
categories: [自动化运维]
keywords: puppet,自动化运维
description: puppet,自动化运维
---
[自动化运维都有哪些开源软件?](#pp1)<br>
[什么是Puppet?](#pp2)<br>
[Puppet都有哪些特性?](#pp3)<br>
[Puppet适用于哪些场合?](#pp4)<br>
[Puppet社区版和企业版本功能上有什么差别?](#pp5)<br>
[Puppet支持哪些系统?](#pp6)<br>
[Puppet架构是怎样的?](#pp7)<br>
[Puppet如何工作的?](#pp8)<br>
[Puppet组织结构是怎样的?](#pp9)<br>
[学习Puppet去哪里?](#pp10)<br>

**[自动化运维都有哪些开源软件？](id:pp1)**<br>

初始化：Kickstart、Cobbler、 Rpmbuild/Xen、Kvm、Lxc、Docker/Openstack、Cloudstack、Opennebula、Eucalyplus<br>
配置类工具：Chef、Puppet、Func、Cfengine<br>
命令和控制类工具: Fabric、Salstack、Ansible、Capistrano、Pssh、Dsh、Expect<br>
监控类工具：Cacti、Nagios、Zabbix、Ganglia<br>
推荐阅读：http://hotpu-meeting.b0.upaiyun.com/2014dtcc/post_pdf/liuyu.pdf<br>
<!--more-->   

**[什么是Puppet？](id:pp2)**<br>
&emsp;&emsp;puppet是一种Linux、Unix平台的集中配置管理系统，使用ruby语言，可管理配置文件、用户、cron任务、软件包、系统服务等。puppet把这些系统实体称之为资源，puppet的设计目标是简化对这些资源的管理以及妥善处理资源间的依赖关系

**[Puppet都有哪些特性？](id:pp3)**

- 可自动化重复任务、快速部署关键性应用及本地或者云端完成主动管理变更和快速扩展架构规模等
- 遵循GPL协议，基于ruby开发，2.7.0以后使用Apache 2.0 License
- 对于sa来讲是抽象的，只依赖于ruby与facter
- 基于C/S架构，配置master和agent
- 默认agent每30分钟连接到puppet master
- 能管理多达40多种资源，如：file、user 、group、host、packeage、service、cron、exec、yumrepo等，适合整个软件生命周期的管理

**[puppet的整个生命周期](id:pp4)**<br>
&emsp;&emsp;供应（provisioning:包安装）-->配置（configuration）-->联动(orchestration)-->报告(reporting)

**[Puppet适用于哪些场合？](id:pp5)**

- 初始化配置、修复、升级、审计
- 统一安装、配置管理软件
- 统一配置系统优化参数
- 定期检测服务是否运行
- 快速替换集群时设备的角色

**[Puppet社区版和企业版本功能上有什么差别？](id:pp6)**

![](http://geekwolf.github.io/images/puppet/cb.png)


请参考：[http://puppetlabs.com/puppet/enterprise-vs-open-source](http://puppetlabs.com/puppet/enterprise-vs-open-source)


**[Puppet支持哪些系统？](id:pp7)**

![](http://geekwolf.github.io/images/puppet/zcos.png)

**[Puppet架构是怎样的？](id:pp8)**

![](http://geekwolf.github.io/images/puppet/ppjg.png)

**[Puppet如何工作的？](id:pp9)**

![](http://geekwolf.github.io/images/puppet/yl.png)

&emsp;&emsp;所有配置信息为实现其通用性，在master端通常被定义为modules--class--resource（管理和被管理的对象）<br>
&emsp;&emsp;资源组成类，类封装成模块<br>
&emsp;&emsp;puppet只定义目标状态，不用关心实现过程；<br>
module(class(resource))-->node 'FQDN' {class1,class2}-->agent(facter)报告facter给master -->master根据facter信息生成相应的catalog结果-->agent 应用catalog<br>

**流程简述如下：**

1. 客户端puppetd向master发起认证请求。
2. Puppet Master告诉client是合法的。
3. 客户端puppetd开始调用facter，facter可以探测出主机的一些变量，例如主机名，内存大小，IP地址等。pupppetd 把这些信息通过ssl连接发送到服务器端。
4. 服务器端的puppet Master 检测客户端的主机名，然后找到manifest里面对应的node配置， 并对该部分内容进行解析，解析分为几个阶段，语法检查，如果语法错误就报错。如果语法没错，就继续解析，解析的结果会生成一个中间的“伪代码”(catalog)，然后把伪代码发给客户端。
5. 客户端接收到“伪代码”，并且执行。
6. 客户端在执行时判断有没有file文件，如果有就向Fileserver发起请求。
7. 客户端继续判断有没有配置Report。如果配置，就把执行结果发送给服务器。
8. 服务器端把客户端的执行结果写入日志。并可以发送给报告系统(DashBoard)

**[Puppet组织结构是怎样的？](id:pp10)**<br>

Puppet的目录结构描述如下：
|-- puppet.conf           # 主配置配置文件<br>
|-- fileserver.conf       #文件服务器配置文件<br>
|-- auth.conf             #认证配置文件  (只允许域内认证)<br>
|-- autosign.conf         #自动验证配置文件<br>
|-- tagmail.conf          # 邮件配置文件（将错误信息发送）<br>
|-- manifests             # 文件存储目录(puppet会先读取该目录的.pp文件<site.pp>)<br>
|-- nodes<br>
| |  | puppetclient.pp   #puppet解析主配置文件所有的模块和节点都在此文件里include<br>
| |-- site.pp            # 定义puppet相关的变量和默认配置<br>
| |-- modules.pp         # 加载class类模块文件（include nginx）<br>
|--  modules             # 定义模块<br>
| --nginx                # 以nginx为例<br>
|           |--  file<br>
|           |--  manifests<br>
|           |     |-- init.pp       #类的定义，类名必须与模块名相同<br>
|           |--- templates          # 模块配置目录，可以被模块的manifests引用<br>
|           |     |-- nginx.erb     #erb模板<br>

**学习Puppet去哪里？**

Puppet相关文档：[http://docs.puppetlabs.com/](http://docs.puppetlabs.com/)<br>
常用模块下载地址： [https://forge.puppetlabs.com/](https://forge.puppetlabs.com/)<br>
PuppetDashboard下载地址：[https://downloads.puppetlabs.com/dashboard/](https://downloads.puppetlabs.com/dashboard/)<br>
PuppetDashboard帮助文档：[http://docs.puppetlabs.com/dashboard/](http://docs.puppetlabs.com/dashboard/)<br>
Puppet中文wiki：[http://puppet.wikidot.com/](http://puppet.wikidot.com/)<br>
Puppet中文论坛：[http://www.puppetfans.com/](http://www.puppetfans.com/)<br>
Puppet运维自动化文档：[http://pan.baidu.com/s/1c0hBMgg](http://pan.baidu.com/s/1c0hBMgg)<br>
Puppet简单安装可以参考：[http://www.chenshake.com/puppet-study-notes/#i-3](http://www.chenshake.com/puppet-study-notes/#i-3)<br>

> 参考 南非蜘蛛
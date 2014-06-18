---
layout: post
title: "Centos6.5下安装Puppet及测试"
date: 2014-06-16 13:50:24 +0800
comments: true
categories: [自动化运维]
keywords: Puppet,证书签发
description: 在Centos6.5下安装Puppet及证书的管理和测试
---
一、[安装前注意事项](#ap1)<br>
二、[环境说明](#ap2)<br>
三、[安装步骤](#ap3)<br>
&emsp;&emsp;[配置puppet的yum源](#ap4)<br>
&emsp;&emsp;[分别安装puppet-master和puppet](#ap5)<br>
&emsp;&emsp;[修改配置](#ap6)<br>
&emsp;&emsp;[证书签发、注销、清楚](#ap7)<br>
&emsp;&emsp;[测试](#ap8)<br>

**[安装前注意事项：](id:ap1)**<br>

1. Puppet master尽量使用高配置server<br>
2. 任何官方未支持的系统也可以正常运行puppet，前提是要装合适的版本的ruby环境<br>
&emsp;请参考:[http://docs.puppetlabs.com/puppet/latest/reference/system_requirements.html#basic-requirements](http://docs.puppetlabs.com/puppet/latest/reference/system_requirements.html#basic-requirements)<br>
3. master防火墙放行8140端口给agent<br>
4. 每个节点都必须有一个唯一的主机名，正解析和反解析都被正确配置，如果没有DNS服务，必须在每个节点上配置/etc/hosts<br> 
&emsp;**注**：默认情况下puppet的master的主机名是puppet<br>
5. 由于Puppet master同时扮演着CA(认证授权机构)的角色,需要时间同步,启动ntpd服务；<br>
6. 两种工作模式：Master/Agent、Standatone<br>

**[环境说明：](id:ap2)**<br>
<!--more-->      

192.168.10.216&emsp;&emsp;&emsp;Puppet Agent&emsp;&emsp;&emsp;c1.geekwolf.github.io<br>
192.168.10.217&emsp;&emsp;&emsp;Puppet Agent&emsp;&emsp;&emsp;c2.geekwolf.github.io<br>
192.168.10.218&emsp;&emsp;&emsp;Puppet Master&emsp;&emsp;&emsp;m.geekwolf.github.io<br>

**[安装步骤](id:ap3)**<br>

**[1.安装puppet的yum源](id:ap4)**<br>
rpm -ivh http://yum.puppetlabs.com/puppetlabs-release-el-6.noarch.rpm<br>
说明:若要测试RC版及相关软件编辑/etc/yum.repos.d/puppetlabs.repo：<br>

    [puppetlabs-devel]
    name=Puppet Labs Devel El 6 - $basearch
    baseurl=http://yum.puppetlabs.com/el/6/devel/$basearch
    gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-puppetlabs
    enabled=1
    gpgcheck=1
    
    yum -y install  ntp
    service ntpd start
    chkconfig ntpd on

配置好hostname，并将解析写进hosts同步到所有节点<br>
**[2.在master:192.168.10.218安装puppet-server](id:ap5)**<br>

    yum -y install puppet-server（依赖puppet、facter一起安装）
    生成启动脚本:
    /etc/init.d/puppetmaster 
    Master配置文件目录：
    /etc/puppet
    
    chkconfig puppetmaster on
    service puppetmaster start
    
    升级puppet master：
    puppet resource package puppet ensure=latest
    service puppetmaster restart

**3.在c1 c2 上安装puppet**<br>

    yum -y install puppet
    chkconfig puppet on
    service puppet start
    Agent配置文件目录:
    /etc/sysconfig/puppet.conf

**[4.配置puppet agent c1 c2指定Puppet Master地址](id:ap6)**<br>

    vim  /etc/puppet/puppet.conf
    [main]
    # The Puppet log directory.
    # The default value is '$vardir/log'.
    logdir = /var/log/puppet
    # Where Puppet PID files are kept.
    # The default value is '$vardir/run'.
    rundir = /var/run/puppet
    # Where SSL certificates are kept.
    # The default value is '$confdir/ssl'.
    ssldir = $vardir/ssl
    [agent]
    # The file in which puppetd stores a list of the classes
    # associated with the retrieved configuratiion. Can be loaded in
    # the separate ``puppet`` executable using the ``--loadclasses``
    # option.
    # The default value is '$confdir/classes.txt'.
    classfile = $vardir/classes.txt
    # Where puppetd caches the local configuration. An
    # extension indicating the cache format is added automatically.
    # The default value is '$confdir/localconfig'.
    localconfig = $vardir/localconfig
    server = m.geekwolf.github.io

**[5.签发证书](id:ap7)**<br>
**A.手动签发证书**<br>
c1、c2申请证书，由于已经配置了server=m.geekwolf.github.io，故申请时不必在指定server<br>
    
    [root@c1 ~]# puppet agent -t
    Info: Creating a new SSL key for c1.geekwolf.github.io
    Info: Caching certificate for ca
    Info: Caching certificate_request for c1.geekwolf.github.io
    Info: Caching certificate for ca
    Exiting; no certificate found and waitforcert is disabled

在Master上管理证书：<br>
**签发证书：**<br>

    puppet cert list --all                   查看请求签发的证书（+表示已签发，-未签发）
    puppet cert --sign c1.geekwolf.github.io 签发主机c1.geekwolf.github.io的证书
    puppet cert --sign --all                 签发所有请求的主机的证书

**注销证书：**

    puppet cert revoke  c1.geekwolf.github.io 注销主机c1.geekwolf.github.io的证书
    puppet cert revoke --all                  注销所有主机的证书（若想在重新签名，需先重启puppetmaster,然后节点在请求申请证书，再签名即可）

**清除证书：**

    在master上清除某节点证书,重启puppetmaster后生效
    puppet cert --clean c1.geekwolf.github.io 
    
    在agent上删除相关目录,可以重新再申请签名   
    rm -rf /var/lib/puppet/ssl 或者rm -rf /var/lib/puppet/certs/c1.geekwolf.github.io.pem 
    
**B.自动签发证书**<br>
在Puppet Master创建/etc/puppet/autosign.conf文件<br>
*.geekwolf.github.io        (geekwolf.github.io域的申请会自动签发)<br>
service puppetmaster restart<br>
所有节点上执行<br>
rm -rf /var/lib/puppet/ssl<br>

然后所有的节点申请签名<br>
 puppet agent -t --server m.geekwolf.github.io

    在Puppet Master查看签名
    [root@m puppet]# puppet cert list --all
    + "c1.geekwolf.github.io" (SHA256) 2A:28:96:6C:B0:36:E8:CC:71:80:F4:C6:B5:D8:61:94:A8:59:46:9D:52:A3:58:2A:D9:78:45:A3:57:93:1C:38
    + "c2.geekwolf.github.io" (SHA256) 09:59:71:9A:CA:AE:92:82:1D:D4:0C:A6:D4:5F:51:C3:D6:E4:EE:80:20:19:CB:B1:71:EE:B3:24:F7:E3:80:71
    + "m.geekwolf.github.io" (SHA256) 10:F3:28:EA:36:25:38:C5:1C:8A:38:FD:94:EF:F9:77:6B:97:E9:FA:60:18:D5:53:DD:5D:DA:15:88:4F:96:A1 (alt names: "DNS:m.geekwolf.github.io", "DNS:puppet", "DNS:puppet.geekwolf.github.io")

**参考文档：** [多CA配置](http://docs.puppetlabs.com/puppet/3.6/reference/config_ssl_external_ca.html#option-3-two-intermediate-cas-issued-by-one-root-ca)

**[6.测试](id:ap8)**<br>

    默认agent（c1，c2）每30分钟连接到puppet master,为测试方便先修改连接时间
    echo "runinterval = 10" >>/etc/puppet/puppet.conf
    service puppet restart
    
    Puppet Master：
    vim /etc/puppet/manifests/site.pp
    file {"/tmp/test.txt" :
      content=>"test from geekwolf!~\n"; } 
    
    检查c1 c2是否有/tmp/test.txt文件

# CDH(6.3.1)

## 1.cdh简介

### cdh概念

​		CDH是Cloudera的100％开源平台发行版，包括Apache Hadoop，专为满足企业需求而构建。CDH提供开箱即用的企业使用所需的一切。通过将Hadoop与十几个其他关键的开源项目集成，Cloudera创建了一个功能先进的系统，可帮助您执行端到端的大数据工作流程。

​		简单来说：CDH 是一个拥有集群自动化安装、中心化管理、集群监控、报警功能的一个工具（软件），使得集群的安装可以从几天的时间缩短为几个小时，运维人数也会从数十人降低到几个人，极大的提高了集群管理的效率。

### Cloudera Manager的功能

- 管理：对集群进行管理，例如添加、删除节点等操作
- 监控：监控集群的健康情况，对设置的各种指标和系统的具体运行情况进行全面的监控
- 诊断：对集群出现的各种问题进行诊断，并且给出建议和解决方案
- 集成：多组件可以进行版本兼容间的整合

### cloudera manager 架构原理

cloudera manager的核心是管理服务器，该服务器承载管理控制台的Web服务器和应用程序逻辑，并负责安装软件，配置，启动和停止服务，以及管理上的服务运行群集。

![1_jg](C:\Users\xf109012\Desktop\新建文件夹\图片\CDH\1_jg.png)

### **Cloudera Manager Server由以下几个部分组成:**

Agent：安装在每台主机上。该代理负责启动和停止的过程，拆包配置，触发装置和监控主机。

Management Service：由一组执行各种监控，警报和报告功能角色的服务。

Database：存储配置和监视信息。通常情况下，多个逻辑数据库在一个或多个数据库服务器上运行。

Cloudera Repository：软件由Cloudera 管理分布存储库。

Clients：是用于与服务器进行交互的接口。

Admin Console ：基于Web的用户界面与管理员管理集群和Cloudera管理。

API ：与开发人员创建自定义的Cloudera Manager应用程序的API

![1_zj](C:\Users\xf109012\Desktop\新建文件夹\图片\CDH\1_zj.png)

## 2.cdh集群新增agent节点安装部署

#### 1.新增agent节点agent04

```shell
172.28.54.192  # hostnamectl set-hostname agent04
```

#### 2.将cm-server密钥分发至新agent04

```shell
# cm-server
[root@cm-server ~]# ssh-copy-id 172.28.54.192
```

#### 3.修改hosts解析新增节点&分发

```shell
# cm-server
[root@cm-server ~]# vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

# CDH
172.28.54.188   cm-server
172.28.54.189   agent01
172.28.54.190   agent02
172.28.54.191   agent03
172.28.54.192   agent04

# 分发新hosts
[root@cm-server ~]# for i in agent01 agent02 agent03 agent04;do scp /etc/hosts $i:/etc/hosts;done

```

#### 4.新节点关闭防火墙

```shell
[root@agent04 ~]# systemctl disable firewalld
[root@agent04 ~]# systemctl stop firewalld

```

#### 5.新节点关闭SELINUX

```shell
[root@agent04 ~]# setenforce 0
setenforce: SELinux is disabled

[root@cm-server ~]# vim /etc/selinux/config
# 将SELINUX置为disabled
SELINUX=disabled

```

#### 6.新节点配置时间同步服务

```shell
# cm-server
[root@cm-server ~]# scp /etc/chrony.conf agent04:/etc/chrony.conf
# agent04
[root@agent04 ~]# systemctl restart chronyd && systemctl enable chronyd
[root@agent04 ~]# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* cm-server                    11   6    17    21   +990ns[ -714ns] +/-   91ms
[root@agent04 ~]# 

```

#### 7.新节点调优系统swappiness参数

```shell
[root@agent04 ~]# vim /etc/sysctl.conf
vm.swappiness = 0

```

#### 8.新节点关闭透明大页面

```shell
[root@agent04 ~]# echo never > /sys/kernel/mm/transparent_hugepage/defrag
[root@agent04 ~]# echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

#### 9.确认节点上已安装python

```shell
[root@agent04 ~]# python -V
Python 2.7.5

```

#### 10.新节点java安装

```shell
# 创建java文件目录
[root@agent04 ~]# mkdir -p /usr/java
# cm-server 直接从server端拷贝
[root@cm-server ~]# scp -r /usr/java/default agent04:/usr/java/
# 环境变量profile
[root@agent04 ~]# vim /etc/profile
# JAVA_HOME
export JAVA_HOME=/usr/java/default
export CLASSPATH=./:$JAVA_HOME/lib
export PATH=$JAVA_HOME/bin:$PATH
# 引用生效
[root@agent04 ~]# source /etc/profile

```

#### 11.拷贝 JDBC 驱动包

```shell
# cm-server
[root@cm-server ~]# scp /opt/software/mysql-connector-java-8.0.20.jar agent03:/usr/share/java/mysql-connector-java.jar

```

#### 12.安装Clouder Manager Agent

```shell
# 将安装包从cm-server分发过去
[root@cm-server ~]# cd /opt/software/
[root@cm-server software]# scp cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm agent04:~/
[root@cm-server software]# scp cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm agent04:~/
# 在agent04上执行安装
[root@agent04 ~]# ll
total 1185864
-rw-r--r-- 1 root root   10483568 Sep 14 15:14 cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
-rw-r--r-- 1 root root 1203832464 Sep 14 15:14 cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
[root@agent04 ~]# yum install -y cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
[root@agent04 ~]# yum install -y cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
# 修改agent服务的配置文件指向server
[root@agent04 ~]# vim /etc/cloudera-scm-agent/config.ini
# 将server_host的配置改为server的主机名 [ 主机名是自己环境的server的host主机名 ]
server_host=cm-server
```

#### 13.启动新agent服务

```shell
[root@agent04 ~]# systemctl start cloudera-scm-agent
[root@agent04 ~]# systemctl enable cloudera-scm-agent
```

#### 14.页面进行后续添加

找到Add Hosts

![2_1](C:\Users\xf109012\Desktop\新建文件夹\图片\CDH\2_1.png)

选择添加到哪个集群并继续

![2_2](C:\Users\xf109012\Desktop\新建文件夹\图片\CDH\2_2.png)

选择当前管理的主机，会将还未加入到集群的新节点罗列出来

![2_3](C:\Users\xf109012\Desktop\新建文件夹\图片\CDH\2_3.png)

Install Parcels和检查主机正确性操作与之前相同，这里跳过

选择主机模板，如果长期使用的集群，一般都会有模板的，主要是节点的角色，创建好直接有哪些服务的node等等；没有则可以直接选择无，代表加入集群后，手动添加服务实例。

![2_4](C:\Users\xf109012\Desktop\新建文件夹\图片\CDH\2_4.png)

等待部署客户端基本配置和环境点击完成

![2_5](C:\Users\xf109012\Desktop\新建文件夹\图片\CDH\2_5.png)将服务实例部署至新agent

![2_6](C:\Users\xf109012\Desktop\新建文件夹\图片\CDH\2_6.png)

在添加时根据自己需求即可，例如DataNode

![2_7](C:\Users\xf109012\Desktop\新建文件夹\图片\CDH\2_7.png)

添加agent04的角色，所以建议长期使用创建要给模板最佳，相对方便

![2_8](C:\Users\xf109012\Desktop\新建文件夹\图片\CDH\2_8.png)

至此，添加一个新集群节点，并设置新服务实例全部完成

![2_9](C:\Users\xf109012\Desktop\新建文件夹\图片\CDH\2_9.png)


# Hadoop之YARN介绍

Apache Hadoop YARN （Yet Another Resource Negotiator，另一种资源协调者）是一种新的 Hadoop 资源管理器，它是一个通用资源管理系统和调度平台，可为上层应用提供统一的资源管理和调度。

![img](https://img-blog.csdnimg.cn/20200423082116738.png)

它的引入为集群在利用率、资源统一管理和数据共享等方面带来了巨大好处。
可以把[yarn](https://so.csdn.net/so/search?q=yarn&spm=1001.2101.3001.7020)理解为相当于一个分布式的操作系统平台，而mapreduce等运算程序则相当于运行于操作系统之上的应用程序，Yarn为这些程序提供运算所需的资源（内存、cpu）。



## 1. yarn介绍

1. yarn并不清楚用户提交的程序的运行机制

2. yarn只提供运算资源的调度（用户程序向yarn申请资源，yarn就负责分配资源）

3. yarn中的主管角色叫ResourceManager

4. yarn中具体提供运算资源的角色叫NodeManager

5. yarn与运行的用户程序完全解耦，意味着yarn上可以运行各种类型的分布式运算程序，比如mapreduce、storm，spark，tez

   .......

6. spark、storm等运算框架都可以整合在yarn上运行，只要他们各自的框架中有符合yarn规范的资源请求机制即可

7. yarn成为一个通用的资源调度平台.企业中以前存在的各种运算集群都可以整合在一个物理集群上，提高资源利用率，方便数据共享



## 2.Yarn基本框架

![img](https://img-blog.csdnimg.cn/2020042308232559.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjUxODU0MQ==,size_16,color_FFFFFF,t_70)

YARN是一个资源管理、任务调度的框架，主要包含三大模块：ResourceManager（RM）、NodeManager（NM）、ApplicationMaster（AM）

- ResourceManager负责所有资源的监控、分配和管理，一个集群只有一个；
- NodeManager负责每一个节点的维护，一个集群有多个。
- ApplicationMaster负责每一个具体应用程序的调度和协调，一个集群有多个；



## 3.yarn三大组建介绍

### 3.1 RescourceManager

- ResourceManager负责整个集群的资源管理和分配，是一个全局的资源管理系统
- NodeManager以心跳的方式向ResourceManager汇报资源使用情况（目前主要是CPU和内存的使用情况）。RM只接受NM的资源回报信息，对于具体的资源处理则交给NM自己处理
- YARN Scheduler根据application的请求为其分配资源，不负责application
  job的监控、追踪、运行状态反馈、启动等工作



### 3.2 NodeManager

- NodeManager是每个节点上的资源和任务管理器，它是管理这台机器的代理，负责该节点程序的运行，以及该节点资源的管理和监控。YARN集群每个节点都运行一个NodeManager
- NodeManager定时向ResourceManager汇报本节点资源（CPU、内存）的使用情况和Container的运行状态。当ResourceManager宕机时NodeManager自动连接RM备用节点
- NodeManager接收并处理来自ApplicationMaster的Container启动、停止等各种请求



### 3.3 ApplicationMaster

- 用户提交的每个应用程序均包含一个ApplicationMaster，它可以运行在ResourceManager以外的机器上
- 负责与RM调度器协商以获取资源（用Container表示）
- 将得到的任务进一步分配给内部的任务(资源的二次分配)
- 与NM通信以启动/停止任务
- 监控所有任务运行状态，并在任务运行失败时重新为任务申请资源以重启任务
- 当前YARN自带了两个ApplicationMaster实现，一个是用于演示AM编写方法的实例程序DistributedShell，它可以申请一定数目的Container以并行运行一个Shell命令或者Shell脚本；另一个是运行MapReduce应用程序的AM—MRAppMaster
  ==注：RM只负责监控AM，并在AM运行失败时候启动它。RM不负责AM内部任务的容错，任务的容错由AM完成==



## 4.Yarn运行流程

![img](https://img-blog.csdnimg.cn/20200423082917733.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjUxODU0MQ==,size_16,color_FFFFFF,t_70)

1.  client向RM提交应用程序，其中包括启动该应用的ApplicationMaster的必须信息，例如ApplicationMaster程序、启动ApplicationMaster的命令、用户程序等。
2.  ResourceManager启动一个container用于运行ApplicationMaster。
3.  启动中的ApplicationMaster向ResourceManager注册自己，启动成功后与RM保持心跳。
4.  ApplicationMaster向ResourceManager发送请求，申请相应数目的container。
5.  申请成功的container，由ApplicationMaster进行初始化。container的启动信息初始化后，AM与对应的NodeManager通信，要求NM启动container。
6.  NM启动启动container。
7.  container运行期间，ApplicationMaster对container进行监控。container通过RPC协议向对应的AM汇报自己的进度和状态等信息。
8. 应用运行结束后，ApplicationMaster向ResourceManager注销自己，并允许属于它的container被收回。
   

### 详细流程

![img](https://img-blog.csdnimg.cn/20200423082954568.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjUxODU0MQ==,size_16,color_FFFFFF,t_70)



## 5. Yarn 调度器Scheduler

理想情况下，我们应用对Yarn资源的请求应该立刻得到满足，但现实情况资源往往是有限的，特别是在一个很繁忙的集群，一个应用资源的请求经常需要等待一段时间才能的到相应的资源。在Yarn中，负责给应用分配资源的就是Scheduler。其实调度本身就是一个难题，很难找到一个完美的策略可以解决所有的应用场景。为此，Yarn提供了多种调度器和可配置的策略供我们选择。
在Yarn中有三种调度器可以选择：FIFO Scheduler ，Capacity Scheduler，Fair Scheduler。

### 5.1  FIFO Scheduler

FIFO Scheduler把应用按提交的顺序排成一个队列，这是一个先进先出队列，在进行资源分配的时候，先给队列中最头上的应用进行分配资源，待最头上的应用需求满足后再给下一个分配，以此类推。

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20200423083043866.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjUxODU0MQ==,size_16,color_FFFFFF,t_70)

FIFO Scheduler是最简单也是最容易理解的调度器，也不需要任何配置，但它并不适用于共享集群。大的应用可能会占用所有集群资源，这就导致其它应用被阻塞。在共享集群中，更适合采用Capacity Scheduler或Fair Scheduler，这两个调度器都允许大任务和小任务在提交的同时获得一定的系统资源。



### 5.2  Capacity Scheduler

Capacity 调度器允许多个组织共享整个集群，每个组织可以获得集群的一部分计算能力。通过为每个组织分配专门的队列，然后再为每个队列分配一定的集群资源，这样整个集群就可以通过设置多个队列的方式给多个组织提供服务了。除此之外，队列内部又可以垂直划分，这样一个组织内部的多个成员就可以共享这个队列资源了，在一个队列内部，资源的调度是采用的是先进先出(FIFO)策略。


![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20200423083126617.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjUxODU0MQ==,size_16,color_FFFFFF,t_70)

容量调度器 Capacity Scheduler 最初是由 Yahoo 最初开发设计使得 Hadoop 应用能够被多用户使用，且最大化整个集群资源的吞吐量，现被 IBM BigInsights 和 Hortonworks HDP 所采用。
Capacity Scheduler 被设计为允许应用程序在一个可预见的和简单的方式共享集群资源，即"作业队列"。Capacity Scheduler 是根据租户的需要和要求把现有的资源分配给运行的应用程序。Capacity Scheduler 同时允许应用程序访问还没有被使用的资源，以确保队列之间共享其它队列被允许的使用资源。管理员可以控制每个队列的容量，Capacity Scheduler 负责把作业提交到队列中。

### 5.3 Fair Scheduler

在Fair调度器中，我们不需要预先占用一定的系统资源，Fair调度器会为所有运行的job动态的调整系统资源。如下图所示，当第一个大job提交时，只有这一个job在运行，此时它获得了所有集群资源；当第二个小任务提交后，Fair调度器会分配一半资源给这个小任务，让这两个任务公平的共享集群资源。
需要注意的是，在下图Fair调度器中，从第二个任务提交到获得资源会有一定的延迟，因为它需要等待第一个任务释放占用的Container。小任务执行完成之后也会释放自己占用的资源，大任务又获得了全部的系统资源。最终效果就是Fair调度器即得到了高的资源利用率又能保证小任务及时完成。

![img](https://img-blog.csdnimg.cn/20200423083147334.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjUxODU0MQ==,size_16,color_FFFFFF,t_70)

公平调度器 Fair Scheduler 最初是由 Facebook 开发设计使得 Hadoop 应用能够被多用户公平地共享整个集群资源，现被 Cloudera CDH 所采用。
Fair Scheduler 不需要保留集群的资源，因为它会动态在所有正在运行的作业之间平衡资源。



##  6. yarn多租户资源隔离

在一个公司内部的Hadoop Yarn集群，肯定会被多个业务、多个用户同时使用，共享Yarn的资源，如果不做资源的管理与规划，那么整个Yarn的资源很容易被某一个用户提交的Application占满，其它任务只能等待，这种当然很不合理，我们希望每个业务都有属于自己的特定资源来运行MapReduce任务，Hadoop中提供的公平调度器–Fair Scheduler，就可以满足这种需求。
Fair Scheduler将整个Yarn的可用资源划分成多个资源池，每个资源池中可以配置最小和最大的可用资源（内存和CPU）、最大可同时运行Application数量、权重、以及可以提交和管理Application的用户等。
Fair Scheduler除了需要在yarn-site.xml文件中启用和配置之外，还需要一个XML文件fair-scheduler.xml来配置资源池以及配额，而该XML中每个资源池的配额可以动态更新，之后使用命令：yarn rmadmin –refreshQueues 来使得其生效即可，不用重启Yarn集群。
==需要注意的是：动态更新只支持修改资源池配额，如果是新增或减少资源池，则需要重启Yarn集群==



### 6.1 编辑yarn-site.xml

yarn集群主节点中yarn-site.xml添加以下配置
cd /export/servers/hadoop-2.6.0-cdh5.14.0/etc/hadoop
vim yarn-site.xml

```shell
<!--  指定使用fairScheduler的调度方式  -->
<property>
	<name>yarn.resourcemanager.scheduler.class</name>
	<value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler</value>
</property>

<!--  指定配置文件路径  -->
<property>
	<name>yarn.scheduler.fair.allocation.file</name>
	<value>/export/servers/hadoop-2.6.0-cdh5.14.0/etc/hadoop/fair-scheduler.xml</value>
</property>

<!-- 是否启用资源抢占，如果启用，那么当该队列资源使用
yarn.scheduler.fair.preemption.cluster-utilization-threshold 这么多比例的时候，就从其他空闲队列抢占资源
  -->
<property>
	<name>yarn.scheduler.fair.preemption</name>
	<value>true</value>
</property>

<property>
	<name>yarn.scheduler.fair.preemption.cluster-utilization-threshold</name>
	<value>0.8f</value>
</property>

<!-- 默认提交到default队列  -->
<property>
	<name>yarn.scheduler.fair.user-as-default-queue</name>
	<value>true</value>
</property>

<!-- 如果提交一个任务没有到任何的队列，是否允许创建一个新的队列，设置false不允许  -->
<property>
	<name>yarn.scheduler.fair.allow-undeclared-pools</name>
	<value>false</value>
</property>
```

### 6.2． 添加fair-scheduler.xml配置文件

yarn主节点执行以下命令，添加faie-scheduler.xml的配置文件
cd /export/servers/hadoop-2.6.0-cdh5.14.0/etc/hadoop
vim fair-scheduler.xml
详细见附件资料。

### 6.3． scp分发配置文件、重启yarn集群

cd /export/servers/hadoop-2.6.0-cdh5.14.0/etc/hadoop
scp yarn-site.xml fair-scheduler.xml node02: P W D s c p y a r n − s i t e . x m l f a i r − s c h e d u l e r . x m l n o d e 03 : PWD scp yarn-site.xml fair-scheduler.xml node03: PWDscpyarn−site.xmlfair−scheduler.xmlnode03:PWD

stop-yarn.sh
start-yarn.sh

### 6.4． 创建普通用户hadoop

node-1执行以下命令添加普通用户
useradd hadoop
passwd hadoop
 

### 6.5． 赋予hadoop用户权限

修改hdfs上面tmp文件夹的权限，不然普通用户执行任务的时候会抛出权限不足的异常。以下命令在root用户下执行。
groupadd supergroup
usermod -a -G supergroup hadoop 修改用户所属的附加群主
su - root -s /bin/bash -c “hdfs dfsadmin -refreshUserToGroupsMappings”
刷新用户组信息

### 6.6． 使用hadoop用户提交程序

su hadoop
hadoop jar /export/servers/hadoop-2.6.0-cdh5.14.0/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.0-cdh5.14.0.jar pi 10 20

### 6.7． 浏览器查看结果

http://node-1:8088/cluster/scheduler
浏览器界面访问，查看Scheduler，可以清晰的看到任务提交到了hadoop队列里面去了。

![img](https://img-blog.csdnimg.cn/20200423083407762.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjUxODU0MQ==,size_16,color_FFFFFF,t_70)
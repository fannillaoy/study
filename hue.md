# hue的简介及基本原理

## 1. hue定义

HUE=Hadoop User Experience(Hadoop用户体验)

- **官方定义**：Hue是一个能够与Apache Hadoop交互的Web应用程序。一个开源的Apache Hadoop UI。
- 特性：一个HDFS的文件浏览器，一个MapReduce/YARN的Job浏览器，一个 HBas的浏览器，Hive，Pig，Cloudera Impala 和 Sqoop2 的查询编辑器。它还附带了一个Oozie的应用程序，用于创建和监控工作流程，一个Zookeeper浏览器和SDK。
- **历史演变**：Hue是一个开源的Apache Hadoop UI系统，由Cloudera Desktop演化而来，最后Cloudera公司将其贡献给Apache基金会的Hadoop社区，它是基于Python Web框架Django实现的。

## 2. hue的核心功能

1. **SQL编辑器**：支持Hive, Impala, MySQL, Oracle, PostgreSQL, SparkSQL, Solr SQL, Phoenix…
2. 搜索引擎Solr的各种图表
3. Spark和Hadoop的友好界面支持
4. 支持调度系统Apache Oozie，可进行workflow的编辑、查看

HUE提供的这些功能相比==Hadoop==生态各组件提供的==界面更加友好==，但是一些需要==debug的场景==可能还是需要使用原生系统才能更加深入的找到错误的原因。



## 3.hue的架构

Hue 是一个Web应用，用来简化用户和Hadoop集群的交互。Hue技术架构，如下图所示，从总体上来讲，Hue应用采用的是B/S架构，该web应用的后台采用python编程语言别写的。大体上可以分为三层，分别是前端view层、Web服务层和Backend服务层。Web服务层和Backend服务层之间使用RPC的方式调用。



## 4.hue与其他技术的整合

由于大数据框架很多，为了解决某个问题，一般来说会用到多个框架，但是每个框架又都有自己的web UI监控界面，对应着不同的端口号。比如HDFS(50070)、YARN(8088)、MapReduce(19888)等。这个时候有一个统一的web UI界面去管理各个大数据常用框架是非常方便的。这就使得对大数据的开发、监控和运维更加的方便。
![img](https://img-blog.csdnimg.cn/img_convert/bbddb41f8a803937da989fe6f81cb1d8.png)

从上图可以看出，Hue几乎可以支持所有大数据框架，包含有HDFS文件系统对的页面(调用HDFS API，进行增删改查的操作)，有HIVE UI界面(使用HiveServer2，JDBC方式连接，可以在页面上编写HQL语句，进行数据分析查询)，YARN监控及Oozie工作流任务调度页面等等。Hue通过把这些大数据技术栈整合在一起，通过统一的Web UI来访问和管理，极大地提高了大数据用户和管理员的工作效率。这里总结一下Hue支持哪些功能：

1. 默认基于轻量级sqlite数据库管理会话数据，用户认证和授权，可以自定义为MySQL、Postgresql，以及Oracle
2. 基于==文件浏览器==（File Browser）访问HDFS
3. 基于==Hive编辑器==来开发和运行Hive查询
4. 支持==基于Solr进行搜索的应用==，并提供可视化的数据视图，以及仪表板（Dashboard）
5. 支持==基于Impala的应用==进行交互式查询
6. 支持==Spark编辑器==和仪表板（Dashboard）
7. 支持==Pig编辑器==，并能够提交脚本任务
8. 支持==Oozie编辑器==，可以通过仪表板提交和监控Workflow、Coordinator和Bundle
9. 支持==HBase浏览器==，能够可视化数据、查询数据、修改HBase表
10. 支持==Job浏览器==，能够访问MapReduce Job（MR1/MR2-YARN）
11. 支持==Sqoop 2编辑器==和仪表板（Dashboard）
12. 支持==ZooKeeper浏览器==和编辑器
13. 支持MySql、PostGresql、Sqlite和Oracle==数据库查询编辑器==
14. 使用sentry基于==角色的授权==以及==多租户的管理==.（Hue 2.x or 3.x）

```
hue支持的框架
          -> hadoop
               -> HDFS
                    -> CRUD
               -> yarn
                    -> 任务的监控
                         -> 自动刷新，权限管理
          -> oozie
               -> 任务的监控及调度
               -> 便捷的任务流的图形化的编写
          -> PIG
          -> hive
               -> 提供简洁的图形化操作界面
               -> 提供报表的生成
          -> impala
          -> hbase
          -> sqoop2
          -> RDBMS
               -> MySQL
               -> oracle
```


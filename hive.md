# hive

Hive是基于Hadoop的一个数据仓库工具，可以将结构化文件映射为一张数据库表，并提供简单的SQL查询功能，可以将SQL语句转换为MapReduce任务进行运行。



## 1.1 为什么要用Hive

### 1.1.1 为什么要使用Hive

1. 直接使用hadoop所面临的问题
   - 人员学习成本太高
   - 项目周期要求太短
   - MapReduce实现复杂查询逻辑开发难度太大
2. 操作接口采用类SQL语法，提供快速开发的能力
3. 避免了去写MapReduce，减少开发人员的学习成本。
4. 功能扩展很方便。

### 1.1.2 Hive的特点

1. 可扩展

   Hive可以自由的扩展集群的规模，一般情况下不需要重启服务

2. 延展性

   Hive支持用户自定义函数，用户可以根据自己的需求来实现自己的函数

3. 容错

   良好的容错性，节点出现问题SQL仍可完成执行

## 1.2 Hive架构

![img](https://img-blog.csdn.net/20170610193705929?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdG90b3R1enVvcXVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

```Jobtracker是hadoop1.x中的组件，它的功能相当于： Resourcemanager+AppMaster```

```TaskTracker 相当于：  Nodemanager +  yarnchild```

### 1.2.1基本组成

- 用户接口：包括 CLI(CLI为shell命令行)、JDBC/ODBC(JDBC/ODBC是Hive的JAVA实现，与传统数据库JDBC类似)、WebGUI(WebGUI是通过浏览器访问Hive)。
- 元数据存储：通常是存储在关系数据库如 mysql , derby中.(Hive将元数据存储在数据库中。Hive 中的元数据包括表的名字，表的列和分区及其属性，表的属性（是否为外部表等），表的数据所在目录等。)
- 解释器、编译器、优化器、执行器.( 解释器、编译器、优化器完成 HQL 查询语句从词法分析、语法分析、编译、优化以及查询计划的生成。生成的查询计划存储在HDFS 中，并在随后有 MapReduce 调用执行)



## 1.3 Hive与Hadoop的关系

Hive利用HDFS存储数据，利用MapReduce查询分析数据

![img](https://img-blog.csdn.net/20170610194939213?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdG90b3R1enVvcXVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



## 1.4 Hive与传统数据库对比

hive用于海量数据的离线数据分析

![img](https://img-blog.csdn.net/20170610195004561?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdG90b3R1enVvcXVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

==*总结：hive具有sql数据库的外表，但应用场景完全不同，hive只适合用来做批量数据统计分析*==



## 1.5Hive的数据存储

1. Hive中所有的数据都存储在 HDFS 中，没有专门的数据存储格式（可支持Text，SequenceFile，ParquetFile，RCFILE等）
2. 只需要在创建表的时候告诉 Hive 数据中的列分隔符和行分隔符，Hive 就可以解析数据
3. Hive 中包含以下数据模型：DB、Table，External Table，Partition，Bucket
   - db：在hdfs中表现为${hive.metastore.warehouse.dir}目录下一个文件夹
   - table：在hdfs中表现所属db目录下一个文件夹
   - external table：与table类似，不过其数据存放位置可以在任意指定路径
   - partition：在hdfs中表现为table目录下的子目录
   - bucket：在hdfs中表现为同一个表目录下根据hash散列之后的多个文件



## 1.6 总结

1. 查询语言。由于 SQL 被广泛的应用在==数据仓库==中，因此，专门针对 Hive 的特性设计了类 SQL 的查询语言 HQL。熟悉SQL 开发的开发者可以很方便的使用 Hive 进行开发
2. 数据存储位置。Hive 是建立在 Hadoop 之上的，所有 Hive 的数据都是存储在 HDFS 中的。而数据库则可以将数据保存在块设备或者本地文件系统中
3. 数据格式。Hive 中没有定义专门的数据格式，数据格式可以由用户指定，用户定义数据格式需要指定三个属性：列分隔符（通常为空格、”\t”、”\x001″）、行分隔符（”\n”）以及读取文件数据的方法（Hive 中默认有三个文件格式 TextFile，SequenceFile 以及 ORCFile）。由于在加载数据的过程中，不需要从用户数据格式到 Hive 定义的数据格式的转换，因此，Hive 在加载的过程中不会对数据本身进行任何修改，而只是将数据内容复制或者移动到相应的 HDFS 目录中。而在数据库中，不同的数据库有不同的存储引擎，定义了自己的数据格式。所有数据都会按照一定的组织存储，因此，数据库加载数据的过程会比较耗时。
4. 数据更新。由于 Hive 是针对数据仓库应用设计的，而数据仓库的内容是读多写少的。因此，Hive 中不支持对数据的改写和添加，所有的数据都是在加载的时候中确定好的。而数据库中的数据通常是需要经常进行修改的，因此可以使用 INSERT INTO ...  VALUES 添加数据，使用 UPDATE... SET 修改数据。
5. 索引。之前已经说过，Hive 在加载数据的过程中不会对数据进行任何处理，甚至不会对数据进行扫描，因此也没有对数据中的某些 Key 建立索引。Hive 要访问数据中满足条件的特定值时，需要暴力扫描整个数据，因此访问延迟较高。由于 MapReduce 的引入， Hive 可以并行访问数据，因此即使没有索引，对于大数据量的访问，Hive 仍然可以体现出优势。数据库中，通常会针对一个或者几个列建立索引，因此对于少量的特定条件的数据的访问，数据库可以有很高的效率，较低的延迟。由于数据的访问延迟较高，决定了 Hive 不适合在线数据查询
6. 执行。Hive 中大多数查询的执行是通过 Hadoop 提供的 MapReduce 来实现的，而数据库通常有自己的执行引擎。
7. 执行延迟。之前提到，Hive 在查询数据的时候，由于没有索引，需要扫描整个表，因此延迟较高。另外一个导致Hive 执行延迟高的因素是 MapReduce 框架。由于MapReduce 本身具有较高的延迟，因此在利用 MapReduce 执行 Hive 查询时，也会有较高的延迟。相对的，数据库的执行延迟较低。当然，这个低是有条件的，即数据规模较小，当数据规模大到超过数据库的处理能力的时候，Hive 的并行计算显然能体现出优势。
8. 可扩展性。由于 Hive 是建立在 Hadoop 之上的，因此 Hive 的可扩展性是和 Hadoop 的可扩展性是一致的（世界上最大的 Hadoop 集群在 Yahoo!，2009年的规模在 4000 台节点左右）。而数据库由于 ACID 语义的严格限制，扩展行非常有限。目前最先进的并行数据库 Oracle 在理论上的扩展能力也只有 100 台左右
9. 数据规模。由于 Hive 建立在集群上并可以利用 MapReduce 进行并行计算，因此可以支持很大规模的数据；对应的，数据库可以支持的数据规模较小。
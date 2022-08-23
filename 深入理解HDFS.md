如果原来的 Active NameNode 恢复正常之后再向 JournalNode 写 EditLog，那么因为它的 Epoch 肯定比新生成的 Epoch 小，并且大多数的 JournalNode 都接受了这个新生成的 Epoch，所以拒绝写入的 JournalNode 数目至少是大多数，这样原来的 Active NameNode 写 EditLog 就肯定会失败，失败之后这个 NameNode 进程会直接退出，这样就实现了对原来的 Active NameNode 的隔离了。



# 深入理解HDFS

## 1.请简述HDFS的HA.

### 1.1 HDFS NameNode 的高可用整体架构如图：

![img](https://pics1.baidu.com/feed/a8014c086e061d953a77ffbabe0c24d863d9ca05.jpeg?token=0f2287febe4863a3aca1175229db660a)

### 1.2 NameNode 的高可用架构主要分为下面几个部分：

Active NameNode 和 Standby NameNode：两台 NameNode 形成互备，一台处于 Active 状态，为主 NameNode，另外一台处于 Standby 状态，为备 NameNode，只有主 NameNode 才能对外提供读写服务。

主备切换控制器 ZKFailoverController：ZKFailoverController 作为独立的进程运行，对 NameNode 的主备切换进行总体控制。ZKFailoverController 能及时检测到 NameNode 的健康状况，在主 NameNode 故障时借助 Zookeeper 实现自动的主备选举和切换，当然 NameNode 目前也支持不依赖于 Zookeeper 的手动主备切换。

### 1.3 Zookeeper 集群：为主备切换控制器提供主备选举支持。

共享存储系统：共享存储系统是实现 NameNode 的高可用最为关键的部分，共享存储系统保存了 NameNode 在运行过程中所产生的 HDFS 的元数据。主 NameNode 和NameNode 通过共享存储系统实现元数据同步。在进行主备切换的时候，新的主 NameNode 在确认元数据完全同步之后才能继续对外提供服务。

DataNode 节点：除了通过共享存储系统共享 HDFS 的元数据信息之外，主 NameNode 和备 NameNode 还需要共享 HDFS 的数据块和 DataNode 之间的映射关系。DataNode 会同时向主 NameNode 和备 NameNode 上报数据块的位置信息。



NameNode 的主备切换实现:

NameNode 主备切换主要由 ZKFailoverController、HealthMonitor 和 ActiveStandbyElector 这 3 个组件来协同实现：





ZKFailoverController 作为 NameNode 机器上一个独立的进程启动 (在 hdfs 启动脚本之中的进程名为 zkfc)，启动的时候会创建 HealthMonitor 和 ActiveStandbyElector 这两个主要的内部组件，ZKFailoverController 在创建 HealthMonitor 和 ActiveStandbyElector 的同时，也会向 HealthMonitor 和 ActiveStandbyElector 注册相应的回调方法。





HealthMonitor 主要负责检测 NameNode 的健康状态，如果检测到 NameNode 的状态发生变化，会回调 ZKFailoverController 的相应方法进行自动的主备选举。





ActiveStandbyElector 主要负责完成自动的主备选举，内部封装了 Zookeeper 的处理逻辑，一旦 Zookeeper 主备选举完成，会回调 ZKFailoverController 的相应方法来进行 NameNode 的主备状态切换。



![img](https://pics6.baidu.com/feed/f11f3a292df5e0fe8b852f8b99981aa15cdf7298.png?token=8c57b16cad36270e2eae22ab13f02423)



NameNode 的主备切换流程





步骤如下：





1. HealthMonitor 初始化完成之后会启动内部的线程来定时调用对应 NameNode 的 HAServiceProtocol RPC 接口的方法，对 NameNode 的健康状态进行检测。





1. HealthMonitor 如果检测到 NameNode 的健康状态发生变化，会回调 ZKFailoverController 注册的相应方法进行处理。





1. 如果 ZKFailoverController 判断需要进行主备切换，会首先使用 ActiveStandbyElector 来进行自动的主备选举。





1. ActiveStandbyElector 与 Zookeeper 进行交互完成自动的主备选举。





1. ActiveStandbyElector 在主备选举完成后，会回调 ZKFailoverController 的相应方法来通知当前的 NameNode 成为主 NameNode 或备 NameNode。





1. ZKFailoverController 调用对应 NameNode 的 HAServiceProtocol RPC 接口的方法将 NameNode 转换为 Active 状态或 Standby 状态。





接着说一下 HealthMonitor、ActiveStandbyElector 和 ZKFailoverController 的实现细节。





HealthMonitor 实现分析





ZKFailoverController 在初始化的时候会创建 HealthMonitor，HealthMonitor 在内部会启动一个线程来循环调用 NameNode 的 HAServiceProtocol RPC 接口的方法来检测 NameNode 的状态，并将状态的变化通过回调的方式来通知 ZKFailoverController。





HealthMonitor 主要检测 NameNode 的两类状态，分别是 HealthMonitor.State 和 HAServiceStatus。HealthMonitor.State 是通过 HAServiceProtocol RPC 接口的 monitorHealth 方法来获取的，反映了 NameNode 节点的健康状况，主要是磁盘存储资源是否充足。HealthMonitor.State 包括下面几种状态：





INITIALIZING：HealthMonitor 在初始化过程中，还没有开始进行健康状况检测；





SERVICE_HEALTHY：NameNode 状态正常；





SERVICE_NOT_RESPONDING：调用 NameNode 的 monitorHealth 方法调用无响应或响应超时；





SERVICE_UNHEALTHY：NameNode 还在运行，但是 monitorHealth 方法返回状态不正常，磁盘存储资源不足；





HEALTH_MONITOR_FAILED：HealthMonitor 自己在运行过程中发生了异常，不能继续检测 NameNode 的健康状况，会导致 ZKFailoverController 进程退出；





HealthMonitor.State 在状态检测之中起主要的作用，在 HealthMonitor.State 发生变化的时候，HealthMonitor 会回调 ZKFailoverController 的相应方法来进行处理。





而 HAServiceStatus 则是通过 HAServiceProtocol RPC 接口的 getServiceStatus 方法来获取的，主要反映的是 NameNode 的 HA 状态，包括：





INITIALIZING：NameNode 在初始化过程中；





ACTIVE：当前 NameNode 为主 NameNode；





STANDBY：当前 NameNode 为备 NameNode；





STOPPING：当前 NameNode 已停止；





HAServiceStatus 在状态检测之中只是起辅助的作用，在 HAServiceStatus 发生变化时，HealthMonitor 也会回调 ZKFailoverController 的相应方法来进行处理，具体处理见后文 ZKFailoverController 部分所述。





ActiveStandbyElector 实现分析





Namenode(包括 YARN ResourceManager) 的主备选举是通过 ActiveStandbyElector 来完成的，ActiveStandbyElector 主要是利用了 Zookeeper 的写一致性和临时节点机制，具体的主备选举实现如下：





创建锁节点





如果 HealthMonitor 检测到对应的 NameNode 的状态正常，那么表示这个 NameNode 有资格参加 Zookeeper 的主备选举。





如果目前还没有进行过主备选举的话，那么相应的 ActiveStandbyElector 就会发起一次主备选举，尝试在 Zookeeper 上创建一个路径为/hadoop-ha/{dfs.nameservices}/ActiveStandbyElectorLock 的临时节点 ({dfs.nameservices} 为 Hadoop 的配置参数 dfs.nameservices 的值，下同)，Zookeeper 的写一致性会保证最终只会有一个 ActiveStandbyElector 创建成功，那么创建成功的 ActiveStandbyElector 对应的 NameNode 就会成为主 NameNode，ActiveStandbyElector 会回调 ZKFailoverController 的方法进一步将对应的 NameNode 切换为 Active 状态。





而创建失败的 ActiveStandbyElector 对应的 NameNode 成为备 NameNode，ActiveStandbyElector 会回调 ZKFailoverController 的方法进一步将对应的 NameNode 切换为 Standby 状态。





注册 Watcher 监听





不管创建/hadoop-ha/${dfs.nameservices}/ActiveStandbyElectorLock 节点是否成功，ActiveStandbyElector 随后都会向 Zookeeper 注册一个 Watcher 来监听这个节点的状态变化事件，ActiveStandbyElector 主要关注这个节点的 NodeDeleted 事件。





自动触发主备选举





如果 Active NameNode 对应的 HealthMonitor 检测到 NameNode 的状态异常时， ZKFailoverController 会主动删除当前在 Zookeeper 上建立的临时节点/hadoop-ha/${dfs.nameservices}/ActiveStandbyElectorLock，这样处于 Standby 状态的 NameNode 的 ActiveStandbyElector 注册的监听器就会收到这个节点的 NodeDeleted 事件。





收到这个事件之后，会马上再次进入到创建/hadoop-ha/{dfs.nameservices}/ActiveStandbyElectorLock 节点的流程，如果创建成功，这个本来处于 Standby 状态的 NameNode 就选举为主 NameNode 并随后开始切换为 Active 状态。 当然，如果是 Active 状态的 NameNode 所在的机器整个宕掉的话，那么根据 Zookeeper 的临时节点特性，/hadoop-ha/{dfs.nameservices}/ActiveStandbyElectorLock 节点会自动被删除，从而也会自动进行一次主备切换。





防止脑裂





Zookeeper 在工程实践的过程中经常会发生的一个现象就是 Zookeeper 客户端“假死”，所谓的“假死”是指如果 Zookeeper 客户端机器负载过高或者正在进行 JVM Full GC，那么可能会导致 Zookeeper 客户端到 Zookeeper 服务端的心跳不能正常发出，一旦这个时间持续较长，超过了配置的 Zookeeper Session Timeout 参数的话，Zookeeper 服务端就会认为客户端的 session 已经过期从而将客户端的 Session 关闭。





“假死”有可能引起分布式系统常说的双主或脑裂 (brain-split) 现象。





具体到本文所述的 NameNode，假设 NameNode1 当前为 Active 状态，NameNode2 当前为 Standby 状态。





如果某一时刻 NameNode1 对应的 ZKFailoverController 进程发生了“假死”现象，那么 Zookeeper 服务端会认为 NameNode1 挂掉了，根据前面的主备切换逻辑，NameNode2 会替代 NameNode1 进入 Active 状态。





但是此时 NameNode1 可能仍然处于 Active 状态正常运行，即使随后 NameNode1 对应的 ZKFailoverController 因为负载下降或者 Full GC 结束而恢复了正常，感知到自己和 Zookeeper 的 Session 已经关闭，但是由于网络的延迟以及 CPU 线程调度的不确定性，仍然有可能会在接下来的一段时间窗口内 NameNode1 认为自己还是处于 Active 状态。这样 NameNode1 和 NameNode2 都处于 Active 状态，都可以对外提供服务。





这种情况对于 NameNode 这类对数据一致性要求非常高的系统来说是灾难性的，数据会发生错乱且无法恢复。Zookeeper 社区对这种问题的解决方法叫做 fencing，中文翻译为隔离，也就是想办法把旧的 Active NameNode 隔离起来，使它不能正常对外提供服务。





ActiveStandbyElector 为了实现 fencing，会在成功创建 Zookeeper 节点 hadoop-ha/{dfs.nameservices}/ActiveStandbyElectorLock 从而成为 Active NameNode 之后，创建另外一个路径为/hadoop-ha/{dfs.nameservices}/ActiveBreadCrumb 的持久节点，这个节点里面保存了这个 Active NameNode 的地址信息。





Active NameNode 的 ActiveStandbyElector 在正常的状态下关闭 Zookeeper Session 的时候 (注意由于/hadoop-ha/{dfs.nameservices}/ActiveStandbyElectorLock 是临时节点，也会随之删除)，会一起删除节点/hadoop-ha/{dfs.nameservices}/ActiveBreadCrumb。





但是如果 ActiveStandbyElector 在异常的状态下 Zookeeper Session 关闭 (比如前述的 Zookeeper 假死)，那么由于/hadoop-ha/${dfs.nameservices}/ActiveBreadCrumb 是持久节点，会一直保留下来。





后面当另一个 NameNode 选主成功之后，会注意到上一个 Active NameNode 遗留下来的这个节点，从而会回调 ZKFailoverController 的方法对旧的 Active NameNode 进行 fencing，具体处理见后文 ZKFailoverController 部分所述。





ZKFailoverController 实现分析





ZKFailoverController 在创建 HealthMonitor 和 ActiveStandbyElector 的同时，会向 HealthMonitor 和 ActiveStandbyElector 注册相应的回调函数，ZKFailoverController 的处理逻辑主要靠 HealthMonitor 和 ActiveStandbyElector 的回调函数来驱动。





对 HealthMonitor 状态变化的处理





如前所述，HealthMonitor 会检测 NameNode 的两类状态，HealthMonitor.State 在状态检测之中起主要的作用，ZKFailoverController 注册到 HealthMonitor 上的处理 HealthMonitor.State 状态变化的回调函数主要关注 SERVICE_HEALTHY、SERVICE_NOT_RESPONDING 和 SERVICE_UNHEALTHY 这 3 种状态：





如果检测到状态为 SERVICE_HEALTHY，表示当前的 NameNode 有资格参加 Zookeeper 的主备选举，如果目前还没有进行过主备选举的话，ZKFailoverController 会调用 ActiveStandbyElector 的 joinElection 方法发起一次主备选举。





如果检测到状态为 SERVICE_NOT_RESPONDING 或者是 SERVICE_UNHEALTHY，就表示当前的 NameNode 出现问题了，ZKFailoverController 会调用 ActiveStandbyElector 的 quitElection 方法删除当前已经在 Zookeeper 上建立的临时节点退出主备选举，这样其它的 NameNode 就有机会成为主 NameNode。





而 HAServiceStatus 在状态检测之中仅起辅助的作用，在 HAServiceStatus 发生变化时，ZKFailoverController 注册到 HealthMonitor 上的处理 HAServiceStatus 状态变化的回调函数会判断 NameNode 返回的 HAServiceStatus 和 ZKFailoverController 所期望的是否一致，如果不一致的话，ZKFailoverController 也会调用 ActiveStandbyElector 的 quitElection 方法删除当前已经在 Zookeeper 上建立的临时节点退出主备选举。





对 ActiveStandbyElector 主备选举状态变化的处理





在 ActiveStandbyElector 的主备选举状态发生变化时，会回调 ZKFailoverController 注册的回调函数来进行相应的处理：





如果 ActiveStandbyElector 选主成功，那么 ActiveStandbyElector 对应的 NameNode 成为主 NameNode，ActiveStandbyElector 会回调 ZKFailoverController 的 becomeActive 方法，这个方法通过调用对应的 NameNode 的 HAServiceProtocol RPC 接口的 transitionToActive 方法，将 NameNode 转换为 Active 状态。





如果 ActiveStandbyElector 选主失败，那么 ActiveStandbyElector 对应的 NameNode 成为备 NameNode，ActiveStandbyElector 会回调 ZKFailoverController 的 becomeStandby 方法，这个方法通过调用对应的 NameNode 的 HAServiceProtocol RPC 接口的 transitionToStandby 方法，将 NameNode 转换为 Standby 状态。





如果 ActiveStandbyElector 选主成功之后，发现了上一个 Active NameNode 遗留下来的/hadoop-ha/${dfs.nameservices}/ActiveBreadCrumb 节点 (见“ActiveStandbyElector 实现分析”一节“防止脑裂”部分所述)，那么 ActiveStandbyElector 会首先回调 ZKFailoverController 注册的 fenceOldActive 方法，尝试对旧的 Active NameNode 进行 fencing，在进行 fencing 的时候，会执行以下的操作：





1. 首先尝试调用这个旧 Active NameNode 的 HAServiceProtocol RPC 接口的 transitionToStandby 方法，看能不能把它转换为 Standby 状态。





1. 如果 transitionToStandby 方法调用失败，那么就执行 Hadoop 配置文件之中预定义的隔离措施，Hadoop 目前主要提供两种隔离措施，通常会选择 sshfence：





sshfence：通过 SSH 登录到目标机器上，执行命令 fuser 将对应的进程杀死；





shellfence：执行一个用户自定义的 shell 脚本来将对应的进程隔离；





只有在成功地执行完成 fencing 之后，选主成功的 ActiveStandbyElector 才会回调 ZKFailoverController 的 becomeActive 方法将对应的 NameNode 转换为 Active 状态，开始对外提供服务。





NameNode 的共享存储实现





NameNode 的元数据存储概述





一个典型的 NameNode 的元数据存储目录结构如图 3 所示这里主要关注其中的 EditLog 文件和 FSImage 文件：



![img](https://pics1.baidu.com/feed/ac6eddc451da81cb7f531761969efe1f0b2431e2.jpeg?token=f27a6a671c193d8811b69a1fb52df29b)



NameNode 的元数据存储目录结构





NameNode 在执行 HDFS 客户端提交的创建文件或者移动文件这样的写操作的时候，会首先把这些操作记录在 EditLog 文件之中，然后再更新内存中的文件系统镜像。





内存中的文件系统镜像用于 NameNode 向客户端提供读服务，而 EditLog 仅仅只是在数据恢复的时候起作用。





记录在 EditLog 之中的每一个操作又称为一个事务，每个事务有一个整数形式的事务 id 作为编号。





EditLog 会被切割为很多段，每一段称为一个 Segment。





正在写入的 EditLog Segment 处于 in-progress 状态，其文件名形如 edits_inprogress*{start_txid}，其中{start_txid} 表示这个 segment 的起始事务 id，例如上图中的 edits_inprogress_0000000000000000020。 而已经写入完成的 EditLog Segment 处于 finalized 状态，其文件名形如 edits*{start_txid}-{end_txid}，其中{start_txid} 表示这个 segment 的起始事务 id，{end_txid} 表示这个 segment 的结束事务 id，例如上图中的 edits_0000000000000000001-0000000000000000019。





NameNode 会定期对内存中的文件系统镜像进行 checkpoint 操作，在磁盘上生成 FSImage 文件，FSImage 文件的文件名形如 fsimage_{end_txid}，其中{end_txid} 表示这个 fsimage 文件的结束事务 id，例如上图中的 fsimage_0000000000000000020。





在 NameNode 启动的时候会进行数据恢复，首先把 FSImage 文件加载到内存中形成文件系统镜像，然后再把 EditLog 之中 FsImage 的结束事务 id 之后的 EditLog 回放到这个文件系统镜像上。





基于 QJM 的共享存储系统的总体架构





基于 QJM 的共享存储系统主要用于保存 EditLog，并不保存 FSImage 文件。





FSImage 文件还是在 NameNode 的本地磁盘上。





QJM 共享存储的基本思想来自于 Paxos 算法，采用多个称为 JournalNode 的节点组成的 JournalNode 集群来存储 EditLog。





每个 JournalNode 保存同样的 EditLog 副本。





每次 NameNode 写 EditLog 的时候，除了向本地磁盘写入 EditLog 之外，也会并行地向 JournalNode 集群之中的每一个 JournalNode 发送写请求，只要大多数 (majority) 的 JournalNode 节点返回成功就认为向 JournalNode 集群写入 EditLog 成功。





如果有 2N+1 台 JournalNode，那么根据大多数的原则，最多可以容忍有 N 台 JournalNode 节点挂掉。





基于 QJM 的共享存储系统的内部实现架构图如图 4 所示，主要包含下面几个主要的组件：



![img](https://pics4.baidu.com/feed/b7fd5266d01609243acecdb011ff1bf3e4cd34b7.jpeg?token=72da29b308a54bf4343659b133ac3379)



基于 QJM 的共享存储系统的内部实现架构图





FSEditLog：这个类封装了对 EditLog 的所有操作，是 NameNode 对 EditLog 的所有操作的入口。





JournalSet： 这个类封装了对本地磁盘和 JournalNode 集群上的 EditLog 的操作，内部包含了两类 JournalManager，一类为 FileJournalManager，用于实现对本地磁盘上 EditLog 的操作。一类为 QuorumJournalManager，用于实现对 JournalNode 集群上共享目录的 EditLog 的操作。FSEditLog 只会调用 JournalSet 的相关方法，而不会直接使用 FileJournalManager 和 QuorumJournalManager。





FileJournalManager：封装了对本地磁盘上的 EditLog 文件的操作，不仅 NameNode 在向本地磁盘上写入 EditLog 的时候使用 FileJournalManager，JournalNode 在向本地磁盘写入 EditLog 的时候也复用了 FileJournalManager 的代码和逻辑。





QuorumJournalManager：封装了对 JournalNode 集群上的 EditLog 的操作，它会根据 JournalNode 集群的 URI 创建负责与 JournalNode 集群通信的类 AsyncLoggerSet， QuorumJournalManager 通过 AsyncLoggerSet 来实现对 JournalNode 集群上的 EditLog 的写操作，对于读操作，QuorumJournalManager 则是通过 Http 接口从 JournalNode 上的 JournalNodeHttpServer 读取 EditLog 的数据。





AsyncLoggerSet：内部包含了与 JournalNode 集群进行通信的 AsyncLogger 列表，每一个 AsyncLogger 对应于一个 JournalNode 节点，另外 AsyncLoggerSet 也包含了用于等待大多数 JournalNode 返回结果的工具类方法给 QuorumJournalManager 使用。





AsyncLogger：具体的实现类是 IPCLoggerChannel，IPCLoggerChannel 在执行方法调用的时候，会把调用提交到一个单线程的线程池之中，由线程池线程来负责向对应的 JournalNode 的 JournalNodeRpcServer 发送 RPC 请求。





JournalNodeRpcServer：运行在 JournalNode 节点进程中的 RPC 服务，接收 NameNode 端的 AsyncLogger 的 RPC 请求。





JournalNodeHttpServer：运行在 JournalNode 节点进程中的 Http 服务，用于接收处于 Standby 状态的 NameNode 和其它 JournalNode 的同步 EditLog 文件流的请求。





基于 QJM 的共享存储系统的数据同步机制分析





Active NameNode 和 StandbyNameNode 使用 JouranlNode 集群来进行数据同步的过程如图 5 所示，Active NameNode 首先把 EditLog 提交到 JournalNode 集群，然后 Standby NameNode 再从 JournalNode 集群定时同步 EditLog：



![img](https://pics0.baidu.com/feed/9f2f070828381f3060ee27246df962016c06f04e.jpeg?token=00f160b568496c2a4f2c8388f0c8b6aa)



基于 QJM 的共享存储的数据同步机制





Active NameNode 提交 EditLog 到 JournalNode 集群





当处于 Active 状态的 NameNode 调用 FSEditLog 类的 logSync 方法来提交 EditLog 的时候，会通过 JouranlSet 同时向本地磁盘目录和 JournalNode 集群上的共享存储目录写入 EditLog。





写入 JournalNode 集群是通过并行调用每一个 JournalNode 的 QJournalProtocol RPC 接口的 journal 方法实现的，如果对大多数 JournalNode 的 journal 方法调用成功，那么就认为提交 EditLog 成功，否则 NameNode 就会认为这次提交 EditLog 失败。





提交 EditLog 失败会导致 Active NameNode 关闭 JournalSet 之后退出进程，留待处于 Standby 状态的 NameNode 接管之后进行数据恢复。





从上面的叙述可以看出，Active NameNode 提交 EditLog 到 JournalNode 集群的过程实际上是同步阻塞的，但是并不需要所有的 JournalNode 都调用成功，只要大多数 JournalNode 调用成功就可以了。





如果无法形成大多数，那么就认为提交 EditLog 失败，NameNode 停止服务退出进程。





如果对应到分布式系统的 CAP 理论的话，虽然采用了 Paxos 的“大多数”思想对 C(consistency，一致性) 和 A(availability，可用性) 进行了折衷，但还是可以认为 NameNode 选择了 C 而放弃了 A，这也符合 NameNode 对数据一致性的要求。





Standby NameNode 从 JournalNode 集群同步 EditLog





当 NameNode 进入 Standby 状态之后，会启动一个 EditLogTailer 线程。





这个线程会定期调用 EditLogTailer 类的 doTailEdits 方法从 JournalNode 集群上同步 EditLog，然后把同步的 EditLog 回放到内存之中的文件系统镜像上 (并不会同时把 EditLog 写入到本地磁盘上)。





这里需要关注的是：从 JournalNode 集群上同步的 EditLog 都是处于 finalized 状态的 EditLog Segment。





“NameNode 的元数据存储概述”一节说过 EditLog Segment 实际上有两种状态，处于 in-progress 状态的 Edit Log 当前正在被写入，被认为是处于不稳定的中间态，有可能会在后续的过程之中发生修改，比如被截断。





Active NameNode 在完成一个 EditLog Segment 的写入之后，就会向 JournalNode 集群发送 finalizeLogSegment RPC 请求，将完成写入的 EditLog Segment finalized，然后开始下一个新的 EditLog Segment。





一旦 finalizeLogSegment 方法在大多数的 JournalNode 上调用成功，表明这个 EditLog Segment 已经在大多数的 JournalNode 上达成一致。





一个 EditLog Segment 处于 finalized 状态之后，可以保证它再也不会变化。





从上面描述的过程可以看出，虽然 Active NameNode 向 JournalNode 集群提交 EditLog 是同步的，但 Standby NameNode 采用的是定时从 JournalNode 集群上同步 EditLog 的方式，那么 Standby NameNode 内存中文件系统镜像有很大的可能是落后于 Active NameNode 的，所以 Standby NameNode 在转换为 Active NameNode 的时候需要把落后的 EditLog 补上来。





基于 QJM 的共享存储系统的数据恢复机制分析





处于 Standby 状态的 NameNode 转换为 Active 状态的时候，有可能上一个 Active NameNode 发生了异常退出，那么 JournalNode 集群中各个 JournalNode 上的 EditLog 就可能会处于不一致的状态，所以首先要做的事情就是让 JournalNode 集群中各个节点上的 EditLog 恢复为一致。





另外如前所述，当前处于 Standby 状态的 NameNode 的内存中的文件系统镜像有很大的可能是落后于旧的 Active NameNode 的，所以在 JournalNode 集群中各个节点上的 EditLog 达成一致之后，接下来要做的事情就是从 JournalNode 集群上补齐落后的 EditLog。





只有在这两步完成之后，当前新的 Active NameNode 才能安全地对外提供服务。





补齐落后的 EditLog 的过程复用了前面描述的 Standby NameNode 从 JournalNode 集群同步 EditLog 的逻辑和代码，最终调用 EditLogTailer 类的 doTailEdits 方法来完成 EditLog 的补齐。





使 JournalNode 集群上的 EditLog 达成一致的过程是一致性算法 Paxos 的典型应用场景，QJM 对这部分的处理可以看做是 Single Instance Paxos 算法的一个实现，在达成一致的过程中，Active NameNode 和 JournalNode 集群之间的交互流程如图 6 所示，具体描述如下：



![img](https://pics5.baidu.com/feed/b219ebc4b74543a9581cfd3bdaefa48bbb0114c7.jpeg?token=def18d8506fe36e25c9c146e91245fe0)



Active NameNode 和 JournalNode 集群的交互流程图





生成一个新的 Epoch





Epoch 是一个单调递增的整数，用来标识每一次 Active NameNode 的生命周期，每发生一次 NameNode 的主备切换，Epoch 就会加 1。





这实际上是一种 fencing 机制，为什么需要 fencing 已经在前面“ActiveStandbyElector 实现分析”一节的“防止脑裂”部分进行了说明。





产生新 Epoch 的流程与 Zookeeper 的 ZAB(Zookeeper Atomic Broadcast) 协议在进行数据恢复之前产生新 Epoch 的过程完全类似：





Active NameNode 首先向 JournalNode 集群发送 getJournalState RPC 请求，每个 JournalNode 会返回自己保存的最近的那个 Epoch(代码中叫 lastPromisedEpoch)。





NameNode 收到大多数的 JournalNode 返回的 Epoch 之后，在其中选择最大的一个加 1 作为当前的新 Epoch，然后向各个 JournalNode 发送 newEpoch RPC 请求，把这个新的 Epoch 发给各个 JournalNode。





每一个 JournalNode 在收到新的 Epoch 之后，首先检查这个新的 Epoch 是否比它本地保存的 lastPromisedEpoch 大，如果大的话就把 lastPromisedEpoch 更新为这个新的 Epoch，并且向 NameNode 返回它自己的本地磁盘上最新的一个 EditLogSegment 的起始事务 id，为后面的数据恢复过程做好准备。如果小于或等于的话就向 NameNode 返回错误。





NameNode 收到大多数 JournalNode 对 newEpoch 的成功响应之后，就会认为生成新的 Epoch 成功。





在生成新的 Epoch 之后，每次 NameNode 在向 JournalNode 集群提交 EditLog 的时候，都会把这个 Epoch 作为参数传递过去。





每个 JournalNode 会比较传过来的 Epoch 和它自己保存的 lastPromisedEpoch 的大小，如果传过来的 epoch 的值比它自己保存的 lastPromisedEpoch 小的话，那么这次写相关操作会被拒绝。





一旦大多数 JournalNode 都拒绝了这次写操作，那么这次写操作就失败了。





如果原来的 Active NameNode 恢复正常之后再向 JournalNode 写 EditLog，那么因为它的 Epoch 肯定比新生成的 Epoch 小，并且大多数的 JournalNode 都接受了这个新生成的 Epoch，所以拒绝写入的 JournalNode 数目至少是大多数，这样原来的 Active NameNode 写 EditLog 就肯定会失败，失败之后这个 NameNode 进程会直接退出，这样就实现了对原来的 Active NameNode 的隔离了。





选择需要数据恢复的 EditLog Segment 的 id





需要恢复的 Edit Log 只可能是各个 JournalNode 上的最后一个 Edit Log Segment，如前所述，JournalNode 在处理完 newEpoch RPC 请求之后，会向 NameNode 返回它自己的本地磁盘上最新的一个 EditLog Segment 的起始事务 id，这个起始事务 id 实际上也作为这个 EditLog Segment 的 id。





NameNode 会在所有这些 id 之中选择一个最大的 id 作为要进行数据恢复的 EditLog Segment 的 id。





向 JournalNode 集群发送 prepareRecovery RPC 请求





NameNode 接下来向 JournalNode 集群发送 prepareRecovery RPC 请求，请求的参数就是选出的 EditLog Segment 的 id。





JournalNode 收到请求后返回本地磁盘上这个 Segment 的起始事务 id、结束事务 id 和状态 (in-progress 或 finalized)。





这一步对应于 Paxos 算法的 Phase 1a 和 Phase 1b(参见参考文献 [3]) 两步。





Paxos 算法的 Phase1 是 prepare 阶段，这也与方法名 prepareRecovery 相对应。





并且这里以前面产生的新的 Epoch 作为 Paxos 算法中的提案编号 (proposal number)。只要大多数的 JournalNode 的 prepareRecovery RPC 调用成功返回，NameNode 就认为成功。





选择进行同步的基准数据源，向 JournalNode 集群发送 acceptRecovery RPC 请求 NameNode 根据 prepareRecovery 的返回结果，选择一个 JournalNode 上的 EditLog Segment 作为同步的基准数据源。





选择基准数据源的原则大致是：在 in-progress 状态和 finalized 状态的 Segment 之间优先选择 finalized 状态的 Segment。





如果都是 in-progress 状态的话，那么优先选择 Epoch 比较高的 Segment(也就是优先选择更新的)，如果 Epoch 也一样，那么优先选择包含的事务数更多的 Segment。





在选定了同步的基准数据源之后，NameNode 向 JournalNode 集群发送 acceptRecovery RPC 请求，将选定的基准数据源作为参数。





JournalNode 接收到 acceptRecovery RPC 请求之后，从基准数据源 JournalNode 的 JournalNodeHttpServer 上下载 EditLog Segment，将本地的 EditLog Segment 替换为下载的 EditLog Segment。





这一步对应于 Paxos 算法的 Phase 2a 和 Phase 2b两步。





Paxos 算法的 Phase2 是 accept 阶段，这也与方法名 acceptRecovery 相对应。





只要大多数 JournalNode 的 acceptRecovery RPC 调用成功返回，NameNode 就认为成功。





向 JournalNode 集群发送 finalizeLogSegment RPC 请求，数据恢复完成





上一步执行完成之后，NameNode 确认大多数 JournalNode 上的 EditLog Segment 已经从基准数据源进行了同步。





接下来，NameNode 向 JournalNode 集群发送 finalizeLogSegment RPC 请求，JournalNode 接收到请求之后，将对应的 EditLog Segment 从 in-progress 状态转换为 finalized 状态，实际上就是将文件名从 edits_inprogress*{startTxid} 重命名为 edits*{startTxid}-${endTxid}，见“NameNode 的元数据存储概述”一节的描述。





只要大多数 JournalNode 的 finalizeLogSegment RPC 调用成功返回，NameNode 就认为成功。





此时可以保证 JournalNode 集群的大多数节点上的 EditLog 已经处于一致的状态，这样 NameNode 才能安全地从 JournalNode 集群上补齐落后的 EditLog 数据。





### NameNode 在进行状态转换时对共享存储的处理

1. NameNode 初始化启动，进入 Standby 状态

2. 在 NameNode 以 HA 模式启动的时候，NameNode 会认为自己处于 Standby 模式，在 NameNode 的构造函数中会加载 FSImage 文件和 EditLog Segment 文件来恢复自己的内存文件系统镜像。

3. 在加载 EditLog Segment 的时候，调用 FSEditLog 类的 initSharedJournalsForRead 方法来创建只包含了在 JournalNode 集群上的共享目录的 JournalSet，也就是说，这个时候==只会从 JournalNode 集群之中加载 EditLog，而不会加载本地磁盘上的 EditLog==。

   ```另外值得注意的是，加载的 EditLog Segment 只是处于 finalized 状态的 EditLog Segment，而处于 in-progress 状态的 Segment 需要后续在切换为 Active 状态的时候，进行一次数据恢复过程，将 in-progress 状态的 Segment 转换为 finalized 状态的 Segment 之后再进行读取。```

4. 加载完 FSImage 文件和共享目录上的 EditLog Segment 文件之后，NameNode 会启动 EditLogTailer 线程和 StandbyCheckpointer 线程，正式进入 Standby 模式。

5. 如前所述，EditLogTailer 线程的作用是定时从 JournalNode 集群上同步 EditLog。

6. 而 StandbyCheckpointer 线程的作用其实是为了替代 Hadoop 1.x 版本之中的 Secondary NameNode 的功能，StandbyCheckpointer 线程会在 Standby NameNode 节点上定期进行 Checkpoint，将 Checkpoint 之后的 FSImage 文件上传到 Active NameNode 节点。

   ### NameNode 从 Standby 状态切换为 Active 状态

   1. 当 NameNode 从 Standby 状态切换为 Active 状态的时候，首先需要做的就是停止它在 Standby 状态的时候启动的线程和相关的服务，包括上面提到的 EditLogTailer 线程和 StandbyCheckpointer 线程，然后关闭用于读取 JournalNode 集群的共享目录上的 EditLog 的 JournalSet，接下来会调用 FSEditLog 的 initJournalSetForWrite 方法重新打开 JournalSet。

      ```不同的是，这个 JournalSet 内部同时包含了本地磁盘目录和 JournalNode 集群上的共享目录。```

   2. 这些工作完成之后，就开始执行“基于 QJM 的共享存储系统的数据恢复机制分析”一节所描述的流程，调用 FSEditLog 类的 recoverUnclosedStreams 方法让 JournalNode 集群中各个节点上的 EditLog 达成一致。

   3. 然后调用 EditLogTailer 类的 catchupDuringFailover 方法从 JournalNode 集群上补齐落后的 EditLog。

   4. 最后打开一个新的 EditLog Segment 用于新写入数据，同时启动 Active NameNode 所需要的线程和服务。

### NameNode 从 Active 状态切换为 Standby 状态

1. 当 NameNode 从 Active 状态切换为 Standby 状态的时候，首先需要做的就是停止它在 Active 状态的时候启动的线程和服务，然后关闭用于读取本地磁盘目录和 JournalNode 集群上的共享目录的 EditLog 的 JournalSet。
2. 接下来会调用 FSEditLog 的 initSharedJournalsForRead 方法重新打开用于读取 JournalNode 集群上的共享目录的 JournalSet。
3. 这些工作完成之后，就会启动 EditLogTailer 线程和 StandbyCheckpointer 线程，EditLogTailer 线程会定时从 JournalNode 集群上同步 Edit Log。





# HDFS存储策略

#### 讲存储策略之前，我想先补充下冷热数据的概念

##### 冷热数据从访问频次来看可分为：

**热数据**：是需要被计算节点==频繁访问==的在线类数据。 
**冷数据**：是对于离线类==不经常访问==的数据，比如企业备份数据、业务与操作日志数据、话单与统计数据。
==热数据就近计算，冷数据集中存储==

##### 从数据分析层面来看可分：

冷数据、温数据和热数据。 冷数据——性别、兴趣、常住地、职业、年龄等数据画像，表征“这是什么样的人”； 温数据——近期活跃应用、近期去过的地方等具有一定时效性的行为数据，表征“最近对什么感兴趣”； 热数据——当前地点、打开的应用等场景化明显的、稍纵即逝的营销机会，表征“正在哪里干什么”。

根据对数据的冷热理解，我们能更好的规划我们的hdfs存储策略，那接下来主要讲HDFS存储策略的==异构存储==和==副本放置==两个方面。

## 异构存储策略

Hadoop从2.6.0版本开始支持异构存储功能。我们知道HDFS**默认**的存储策略，对于每个数据块，采用==三个副本的存储方式，保存在不同节点的磁盘上==。异构存储的作用在于利用服务器不同类型的存储介质（包括HDD硬盘、SSD、内存等）提供更多的存储策略（例如三个副本一个保存在SSD介质，剩下两个仍然保存在HDD硬盘），从而使得HDFS的存储能够更灵活高效地应对各种应用场景。 HDFS中预定义支持的各种存储包括：

**ARCHIVE**：高存储密度但耗电较少的存储介质，例如磁带，通常用来>存储冷数据 

**DISK**：磁盘介质，这是HDFS最早支持的存储介质 

**SSD**：固态硬盘，是一种新型存储介质，目前被不少互联网公司使用 

**RAM_DISK** ：数据被写入内存中，同时会往该存储介质中再（异步）写一份



![img](https://pics7.baidu.com/feed/7a899e510fb30f24564f39550d6dff4aaf4b0394.png?token=38cae95f1d4738ddd4b1ff37ac58bf38)



### 异构存储

#### HDFS中支持的存储策略包括：

**Lazy_persist**：一个副本保存在**内存RAM_DISK**中，其余副本保存在磁盘中 

**ALL_SSD**：**所有副本**都保存在SSD中 

**One_SSD**：**一个**副本保存在SSD中，其余副本保存在磁盘中 

**Hot**：**所有副本保存在磁盘**中，这也是默认的存储策略 

**Warm**：一个副本保存在磁盘上，其余副本保存在归档存储上 

**Cold**：所有副本都保存在归档存储上



#### 一个存储策略，包含以下部分

| 策略                                               | 名称                                   |
| -------------------------------------------------- | -------------------------------------- |
| Policy ID                                          | 策略                                   |
| ID Policy name                                     | 策略名称                               |
| A list of storage types for block placement        | 块存放的有关存储类型（可以多个）       |
| A list of fallback storage types for file creation | 如果创建失败的替代存储类型（可以多个） |
| A list of fallback storage types for replication   | 如果复制失败的替代存储类型(可以多个）  |

![img](https://pics1.baidu.com/feed/dc54564e9258d1091d2a9cbf18a0e2b66e814df7.png?token=a6b04d4e0c202c15e4f5fa16dcdd3a7e)

```当有足够空间的时候，块复制使用下表中第三列所列出的存储类型。 如果第三列的空间不够，则考虑用第四列的（创建的时候）或者第五列的（复制的时候）```

配置： dfs.storage.policy.enabled - 启用/关闭存储策略特性。默认是true(开启) dfs.datanode.data.dir - 数据路径，多个以逗号分隔，但必须在前面带上存储类型。

​		总体上HDFS异构存储的价值在于，根据数据热度采用不同策略从而提升集群整体资源使用效率。对于频繁访问的数据，将其全部或部分保存在更高访问性能的存储介质（内存或SSD）上，提升其读写性能；对于几乎不会访问的数据，保存在归档存储介质上，降低其存储成本。但是HDFS异构存储的配置需要用户对目录指定相应的策略，即用户需要预先知道每个目录下的文件的访问热度，在实际大数据平台的应用中，这是比较困难的一点





补充一点，在hadoop3.0之后又引入了支持HDFS文件块级别的纠删码，底层采用Reed-Solomon（k,m）算法。RS是一种常用的纠删码算法，通过矩阵运算，可以为k位数据生成m位校验位，根据k和m的取值不同，可以实现不同程度的容错能力，是一种比较灵活的纠删码算法。

![img](https://pic.rmb.bdstatic.com/bjh/down/7aa314e085b76621b073f29b35591f1b.png)Reed-Solomon（k,m）

HDFS纠删码技术能够降低数据存储的冗余度，以RS(3,2)为例，其数据冗余度为67%，相比Hadoop默认的200%大为减少。但是纠删码技术存储数据和数据恢复都需要消耗cpu进行计算，实际上是一种以时间换空间的选择，因此比较适用的场景是对冷数据的存储。冷数据存储的数据往往一次写入之后长时间没有访问，这种情况下可以通过纠删码技术减少副本数。



#### 副本放置策略

NameNode在挑选合适的DataNode去存储Block的时候，不仅仅考虑了DataNode的存储空间够不够，还会考虑这些DataNode在不在同一个机架上。 这就需要NameNode必须知道所有的DataNode分别位于哪个机架上(所以也称为机架感知)。 当然，默认情况下NameNode是不会知道机架的存在的，也就是说，默认情况下，NameNode会认为所有的DataNode都在同一个机架上(/defaultRack)。 除非我们在hdfs-site.xml里面配置topology.script.file.name选项，这个选项的值是一个可执行文件的位置，而该只执行文件的作用是将输入的DataNode的ip地址按照一定规则计算，然后输出它所在的机架的名字，如/rack1, /rack2之类。借助这个文件，NameNode就具备了机架感知了。当它在挑选DataNode去存储Block的时候，它会遵循以下原则：

1. st replica. 如果写请求方所在机器是其中一个个DataNode,则直接存放在本地,否则随机在集群中选择一个DataNode。
2. nd replica. 第二个副本存放于不同第一个副本的所在的机架。
3. rd replica. 第三个副本存放于第二个副本所在的机架,但是属于不同的节点。

other replica. 如果还有更多的副本，则随机放在节点中。

![img](https://pics7.baidu.com/feed/86d6277f9e2f0708274009572cdc9690a801f225.png?token=50c8701f1ffcd403acfb7808eea8c3e1)



### 单独说说Parquet列式存储：

首先强调，paruqet是列式存储（以前一直以为是行式存储，踩了好多坑，全是泪）。一开始是由twitter和cloudera共同完成并开源，后来被apache孵化为顶级项目。语言无关，可以配合多种hadoop组件使用。

parquet架构:

![img](https://pics3.baidu.com/feed/1ad5ad6eddc451da2e250e7172057c6fd216325e.png?token=b983a8be4524e3ae931431d3550661d7)



###### 如上图所示，parquet序列化和反序列化主要由三大部分组成：

```存储格式（Parquet file format，定义了其内部数据类型和存储格式）、对象模型转换器（Conerters，完成内外部对象映射）和对象模型（object model，内存中的数据表示）组成。```

其列式存储是将某一列的数据连续存储，每一行中的不同列散列分布，这样查询的时候不用扫描全部数据，而且因为列是同构数据可以进行有效压缩，所以能有效减少IO。 

###### 列式存储还有两个概念：映射下推、谓词下推 

映射下推（Project PushDown），是将以前关系型数据库拿出所有数据后再筛选有用的字段，变为直接读取需要的列。而且parquet还能会考虑列是否连续，可以一次性读出多个列的数据到内存。 

谓词下推（Predicate PushDown），是将where或on中的过滤条件尽可能的交给底层执行，从而减少每层交互的数据量。

parquet通过在每个列块存储的时候计算统计信息（最大最小值、空值个数），来判断rowgroup是否需要扫描。 parquet数据模型如下所示：

```shell
message AddressBook {
required string owner;
repeated string ownerPhoneNumbers;
repeated group contacts {
required string name;
optional string phoneNumber;}
}
```

根为message，message包含多个field，没分field包含三个属性：repetition、type、name，其中repetition的取值为required（必须1次）、optional（至多1次）或repeated（任意多次）。 parquet 没有实现复杂的map、list、set，而是使用repeated field（实现set、list） 和 groups（key-value对groups来实现map）来实现。 parquet树型结构如下：



![img](https://pics2.baidu.com/feed/8ad4b31c8701a18bab02d1ae5ad729012a38fe6c.png?token=3b4aaed5da9a1287e22a3d2d4c2afe8c)



parquet树 系统在存储的时候只存叶子节点，但会添加一些辅助字段来显示结构。 如：

![img](https://pics3.baidu.com/feed/6a63f6246b600c3313be5506d3b47f06dbf9a13f.png?token=cebd340dd0685d0f11765a6aebb417cf)

样例数据

其存储结构为：



![img](https://pics4.baidu.com/feed/6c224f4a20a446233e3e9b1a53da5c070ef3d7d7.png?token=6e24e87f3909a276a1d304985d352f01)



## 存储结构

其中r和d字段分别代表repetition level 和 definition level。 repetition level：只针对repeated field，指明其定义在哪一级（级数按repeated计算）重复，且第一次出现因为不存在重复，所以为0。 definition level：指明该列路径上有多少可选（optional、repeated）field被定义。 最终converter通过stripping和assembly这些subfield实现序列化和反序列化。 parquet的文件格式如下：



![img](https://pics3.baidu.com/feed/c995d143ad4bd113f5cbf7c49e578a0649fb05c7.png?token=5b1105c8a544547d2a1afe12fff9ef4f)



## 文件格式

parquet数据会按行组进行切分（官方建议调整行组大小和HDFS块大小到1G以实现最优性能），每个行组包含拥有所有列数据的列块，列块中包含有分页（官方建议8k），分页为parquet压缩和编码的单元，不同的页面允许使用不同的编码方式，分页对于数据模型透明。 性能方面，通过对比不同的存储格式，parquet在数据压缩率和查询速度方面都有明显优势，相较于ORC可嵌套数据结构，但不支持数据修改和ACID,所以更适用于OLAP领域，是一种工业界广泛认可的优化方案。 如果说HDFS是大数据时代文件系统的实际标准的话，parquet就是大数据时代存储格式的实际标准。
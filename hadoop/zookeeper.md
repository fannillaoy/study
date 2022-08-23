# zookeeper

1. ## zookeep介绍

   ​	首先需要了解zookeeper是什么，zookeeper是一个[分布式](https://so.csdn.net/so/search?q=分布式&spm=1001.2101.3001.7020)协调服务。所谓分布式协调主要是来解决分布式系统中多个进程之间的同步限制，防止出险脏读，例如我们常说的分布式锁。

   ​	zookeeper中的数据是存储在内存当中的，因此它的效率十分高效。它内部的存储方式十分类似于文件存储结构，采用了分层存储结构。但是它和文件存储结构的区别是，它的各个节点中是允许存储数据的，需要注意的是zk的每个节点存储数据不能超过1M。它的内存数据结果如下图：

![img](https://img-blog.csdnimg.cn/3bde0addea04491982c2c22515d8654d.png)

我们可以通过不同的路径访问到不同的节点，因为它是分层结构，我们也可以通过某一个父节点，获取到该节点下的所有子节点信息.

​	zk只提供了几个简单的api，但是我们可以通过灵活使用这些api的组合，来实现我们复杂的业务要求：

1. create：创建一个新节点，通过指定路径的方式创建节点，例如创建路径为/A/A1/demo，则会在A1节点下创建一个demo节点；

2. delete：删除节点，通过路径的方式删除节点，如果删除路径为/A/A1/demo，则会删除A1节点下的demo节点；

3. exists：判断指定路径下的节点是否存在，例如判断路径为/A/A1/demo，则会判断A1节点下的demo节点是否存在；

4. get：获取指定路径下某个节点的值是什么，例如获取路径为/A/A1/demo，则会获取A1节点下的demo节点的值什么；

5. set：为指定路径的节点进行赋值操作，例如修改路径为/A/A1/demo，则会修改A1节点下的demo节点的值；

6. get children：获取指定路径节点下的子节点信息，例如获取路径为/A，则会获取A节点下的A1和A2节点；

7. sync：获取到同步数据，这个涉及到了zk的原理，zk集群属于最终一致性，调用该方法，可以获取到最终的结果值，如果不使用该方法，在查询的时候可能获取到的值是中间值；

   

   ​	zk中创建的节点分为两种：永久性节点和临时性节点。永久性节点即创建以后，在不执行delete命令的前提下，该节点是永久存在的；而临时节点与session有关，每个客户端与zk建立链接的时候会生成一个session，这个session不会因为链接zk服务器节点的变化而变化，只有当客户端断开连接以后，该session才会消失，而临时节点会随着session的消失而消失

   ​	zk拥有watch机制，也就是监视机制，可以支持响应式编程模式，它可以对某个路径的终节点及其子节点的变更进行监视，当其发生变更以后，会调用注册的callback方法，然后进行具体的业务逻辑。例如监测路径为/A/A1，那么它会加测A1节点，以及附属于A1的所有子节点，这个子不单单只一层子节点，是指所有层的子节点。

### zk拥有以下几个重要特性:

1. 顺序一致性：来自客户端的相关指令会按照顺序执行，不会出现乱序的情况，客户端发送到服务的指令1->2->3->4，那个这些指令就会按照顺序执行；
2. 原子性：更新只有成功和失败，没有中间状态；
3. 可靠性：也可以称之为持久性，节点更新以后，在下次更新之前，它的数据不会发生变更；
4. 准实时性：也可以称之为最终一致性，在zk集群中，一个客户端修改了其中的一个节点，一定时间以后，所有可用的服务对应的节点都会变成更新以后的值。



## 2.zookeep选主流程

​	zk的设计目标就是高可用性，那么也就意味着，在使用zk的时候一般都是使用集群而不是单点模式。首先来看一下zk的集群模式,如下图：

![img](https://img-blog.csdnimg.cn/87bd693cef4e446db9d8e2d2e7873a22.png)

该图为zk集群的可用状态，从上图中可以看到，zk的集群是主从集群，客户端可以随意与任何zk服务节点进行连接，并且==各个客户端都可以进行读写操作==，这是一个和==redis主从集群的区别==，redis的主从集群，如果客户端是写操作，那么==只能连接redis的主节点才可以==。zk的每个客户端是随机连接到zk服务节点的，并且每个客户端都可以进行读写操作，读操作都是在客户端连接的zk节点进行操作；而写操作是有区别的，如果该客户端连接的是==leader节点==，那么==直接进行写操作==；如果该客户端连接的是==follower节点==，那么zk的服务节点会==自动==将该写操作==转到leader节点进行==。
	 zk的集群为主从集群，那么也就意味着主节点只有一个，那么当==主节点挂了==以后，该zk集群则会处于==不可用状态==，既然zk的设计目的是高可用，也就意味着当主节点挂了以后，zk会有一定的方式来快速的==选出主节点==，让服务恢复可用状态，zk的官方文档中给出的压测报告，7台zk服务，选主耗时大概200ms。
	介绍zk的选举流程之前需要先解释两个概念：zxid以及myid。zxid指的是当前节点的事物id，通俗点说就是当前节点完成的数据同步情况，该值越大，越能说明该节点的数据同步情况越完整，丢失数据的情况越小或者丢失数据越少。myid是在创建zk集群的时候，我们给它的赋值。

​	zk的==follower==节点和==leader==节点是通过==心跳==，来查看服务是否可用。在这其中，只要有有一台follower节点发现==主节点挂掉==，他就开始向其它follower节点==发送选主请求==，整个集群进入选主流程，==不再向外提供服务==。

​	先假设现在有4个zk节点，分别为node1，node2，node3，node4，他们的myid分别为1,2,3,4选主流程主要分为以下两种情况:

1. 初始启动，在启动阶段时，此时各个服务节点的zxid都为0，只与myid有关。假设启动顺序为node1->node2->node3->node4，当启动动1和2的时候，该zk集群是不可用状态，因为zk的选主必须是过半服务节点同意（包含自己），==最低需要启动三个节点才可以进行选举==，因此只有node1和node2启动的时候，此时==只有两台服务，不满足条件==，当第三台节点启动以后，才满足了选主的最低条件，然后进入到选举流程，因为node3的myid最大，所以此时3号节点为leader，然后启动node4，由于此时已经选举出3位leader节点并且过半通过，则==不再选取新的主节点==。则该集群的leader节点为node3。

2. 运行过程中，初始启动过程中的leader（node3）节点挂掉，假设此时只有node4节点发现leader已经挂掉，node1和node2的Zxid都是10，node4的Zxid为9，选主的时候需要比较zxid和myid，需要注意他们的优先级，zxid为第一优先级，myid为第二优先级，选举流程大致分为以下几步：

   

   1)node4节点给自己投票，然后将自己的zxid和myid发送给node1和node2节点

![img](https://img-blog.csdnimg.cn/f236cf5cd1b649d3ba04c7007d30665f.png)

​         2）node1和node2通过比较zxid和myid，发现node4不能成为leader节点，将各自的zxid和myid发送给node4，然后node4接收到以后，发现node1和node2都比自己时候成为leader节点，会给它们进行投票

![img](https://img-blog.csdnimg.cn/a213f7bcbe384c2b9a203f25ecc357db.png)

​		3）node1和node2反驳完node4的选主请求以后，开始进行各自的选主流程，起过程与node4的过程一致，通过上面的优先级，我们可以知道最终node2会成为leader节点，那么以node2为例说一下接下来的流程。node2首先给自己投票，然后将自己zxid和myid推送给node1和node4，此时会发现node2适合成为主节点，则会给node2节点进行投票，最终选出node2成为主节点，zk集群恢复成可用状态。



## 3. zookeep数据一致性

​	 zk服务一般上是以集群状态提供服务，多个zk节点之间的数据一致性是通过zap（原子广播）协议来保证的。zk的数据一致性为最终一致性，需要注意的是他不是实时的，比如node1，node2，node3，其中node3为leader，node1和node2为follower，当node1进行节点创建以后，leader节点肯定为实时更新，但是follower节点不一定为实时更新，因为只要过半通过就算节点已经创建成功，可能会有的节点当前的数据还不是最终态，但是它的更新指令是存在，只是可能还没执行。我们的客户端如果想要读取最终态的数据，那么可以通过使用上面的sync命令，来获取最终数据。

先看一下下面的流程图，然后再进行详细解释：

![img](https://img-blog.csdnimg.cn/e4274e933296414594ccec6fae8c6baa.png)

1. 首先由客户端发送创建节点的指令给到zk节点，假设这个zk节点为follower1节点

2. follower1节点发现是写操作节点，则将该指令通过2888端口转发到leader节点执行

3. leader节点更新自己zxid信息，也就是事务id信息

4. leader节点先将创建节点信息同步到log日志中，然后再follower1和follower2各自的队列中放入创建节点写日志的指令，当follower节点接收到指令以后，执行写日志操作，写入日志成功以后，告诉leader写入完成；leader会判断目前是否已经有过半的节点（包含自己）已经写入完成，如果完成，则先在自己的内存中创建节点，然后将在follower对应的节点中加入在内存中创建节点的指令，然后follower接收到指令以后进行内存操作，操作完成以后告诉leader写入完成，同样需要过半完成

5. 将创建结束的消息返回给调用的follower，然后返回给客户端，节点创建结束

   

   

   上面步骤中的第四步其实就是对原子广播协议的一个大致解释，原子广播协议可以看成两部分，首先原子就代表这只有成功或者失败，没有中间状态；而广播就是并不意味着所有节点都完成相关操作才算完成，只要过半节点是成功的，那么本次操作就算成功完成了。在第四步中提到的队列就是对最终一致性的一个解释，leader会将所有指令按照顺序放入每个follower对应的队列中，每个follower按顺序去执行队列中的指令，达到一个最终一致性的结果。
   

## 4. zookeep分布式锁

​		 zk作为分布式协调服务，它的一个很大的作用就是用来实现分布式锁。zk节点存在临时节点，它的生命周期与session有关，它会随着session的消失而消失，这就比较完美的解决了使用redis作为分布式锁时可能出现的死锁问题。

​		下面看一下简单的分布式锁代码编写。

​		第一部分代码为连接zk时的watch代码，用于监测zk的连接情况，它只需要实现Watcher即可。可以根据不同连接状态，进行不同的处理，我们本次只关心连接状态，因为zk是异步连接，为了保证zk连接成功以后再做接下来的加锁操作，通过CountDownLatch进行阻塞。

```java
/**
 * 连接watcher，主要用来监测zk连接状态
 */
public class ConnectionWatch implements Watcher {
    /**
     * 由于zk获取信息为异步，通过countDownLatch进行阻塞，保证连接成功
     */
    private CountDownLatch countDownLatch;
 
    public void setCountDownLatch(CountDownLatch countDownLatch){
        this.countDownLatch = countDownLatch;
    }
 
    @Override
    public void process(WatchedEvent watchedEvent) {
        System.out.println(watchedEvent.toString());
        switch (watchedEvent.getState()){
 
            case Unknown:
                break;
            case Disconnected:
                break;
            case NoSyncConnected:
                break;
            case SyncConnected:
                // 连接成功，去除阻塞
                countDownLatch.countDown();
                break;
            case AuthFailed:
                break;
            case ConnectedReadOnly:
                break;
            case SaslAuthenticated:
                break;
            case Expired:
                break;
            case Closed:
                break;
        }
 
    }
}
```

第二部分代码为zk的工具类，用于获取zk实例，用于业务代码调用。

```java
public class ZkUtils {
 
    private static volatile ZooKeeper zooKeeper;
    /**
     * zk服务器节点地址，以及锁的主目录
      */
    private final static String url = "127.0.0.1:2181,127.0.0.2:2181,127.0.0.3:2181/orderLock";
 
    private static ConnectionWatch watch = new ConnectionWatch();
 
    private static CountDownLatch countDownLatch = new CountDownLatch(1);
 
    /**
     * 采创建zk
     * @return
     * @throws IOException
     * @throws InterruptedException
     */
    public static ZooKeeper getInstance() throws IOException, InterruptedException {
        watch.setCountDownLatch(countDownLatch);
        // 创建zk实例，1000代表的是session过期时间
        zooKeeper = new ZooKeeper(url, 1000, watch);
        // 在zk连接成功之前进行阻塞
        countDownLatch.await();
        return zooKeeper;
 
    }
}
```

 第三部分为在加锁过程中相关操作的watch以及callback操作，主要功能有创建节点，获取子节点，检查节点是否存在。zk的加锁过程就是创建节点的过程，当创建节点成功并且成功返回，则证明该线程加锁成功，继续进行业务逻辑处理，在加锁的时候，一定要考虑锁的可重入性。下面这段代码实现的是公平锁，谁先创建了临时节点，那么谁就能先获得锁。加锁的大致逻辑是：1）先创建带有序列的临时节点；2）在回调函数中获取父节点的所有子节点，判断当前线程创建的临时节点是否位于第一个，如果是则获取锁，如果不是则判断前一个节点是否存在，然后一直循环该逻辑。

```java
public class LockWatch implements Watcher, AsyncCallback.StringCallback, AsyncCallback.Children2Callback, AsyncCallback.StatCallback {
    private ZooKeeper zooKeeper;
    /**
     * 当前线程名称
      */
    private String threadName;
    /**
     * 当前线程创建的节点名称
     */
    private String nodeName;
    /**
     * 用来进行锁阻塞，只有获取到锁，才放行，否则进行阻塞
     */
    private CountDownLatch countDownLatch = new CountDownLatch(1);
 
    public void setZooKeeper(ZooKeeper zooKeeper) {
        this.zooKeeper = zooKeeper;
    }
 
    public void setThreadName(String threadName) {
        this.threadName = threadName;
    }
 
    /**
     * 加锁操作，也就是往zk的指定目录下插入带有序列的临时节点
     * 需要考虑锁的可重入
     */
    public void tryLock() throws InterruptedException {
        zooKeeper.create("/lock", threadName.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL,
        this, "orderLock");
        countDownLatch.await();
 
    }
 
    /**
     * 解锁操作
     * @throws InterruptedException
     * @throws KeeperException
     */
    public void unLock() throws InterruptedException, KeeperException {
        // -1代表不考虑版本号，在zk中获取，删除等相关操作允许版本号的传入
        zooKeeper.delete(nodeName, -1);
    }
 
    /**
     * 节点创建回调方法
     * @param i
     * @param path
     * @param ctx
     * @param name
     */
    @Override
    public void processResult(int i, String path, Object ctx, String name) {
        if (Objects.nonNull(name) && !"".equals(name)){
            System.out.println(threadName + " create node:" + name);
            nodeName = name.substring(1);
            zooKeeper.getChildren("/", false, this, ctx);
        }
    }
 
    /**
     * 获取子节点信息的回调方法
     * @param i
     * @param path
     * @param o
     * @param children
     * @param stat
     */
    @Override
    public void processResult(int i, String path, Object o, List<String> children, Stat stat) {
        if (children == null || children.isEmpty()){
            System.out.println("children is null......");
            return;
        }
        // 将子节点进行排序，找序号由低到高
        Collections.sort(children);
        // 获取当前创建节点排序以后的下标
        int index = children.indexOf(nodeName);
        // 如果当前节点为第一个节点，则加锁成功
        try {
            if (index < 1){
                System.out.println(threadName +" get lock...");
                // -1代表不考虑版本
                zooKeeper.setData("/", threadName.getBytes(), -1);
                countDownLatch.countDown();
            } else {
                System.out.println(threadName +" not get lock...");
                // 判断该节点的前一个节点是否存在，目的是为了注册节点监控事件，监测删除节点操作
                zooKeeper.exists("/" + children.get(i-1), this, this, o);
            }
        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
 
    }
 
    /**
     * 判断节点是否存在的watch方法
     * @param watchedEvent
     */
    @Override
    public void process(WatchedEvent watchedEvent) {
        switch (watchedEvent.getType()){
 
            case None:
                break;
            case NodeCreated:
                break;
            case NodeDeleted:
                // 当节点删除的时候，代表着解锁，触发后续的抢锁操作
                zooKeeper.getChildren("/", false, this, "orderLock");
                break;
            case NodeDataChanged:
                break;
            case NodeChildrenChanged:
                break;
            case DataWatchRemoved:
                break;
            case ChildWatchRemoved:
                break;
            case PersistentWatchRemoved:
                break;
        }
    }
 
    /**
     * 校验节点是否存在回调方法
     * @param i
     * @param s
     * @param o
     * @param stat
     */
    @Override
    public void processResult(int i, String s, Object o, Stat stat) {
 
    }
}
```

第四部分则为锁的简单应用，使用了junit进行测试，代码如下：

```java
public class ZkLock {
 
    private ZooKeeper zooKeeper;
    /**
     * 没有业务意义，只是为了阻塞主线程
     */
    private CountDownLatch countDownLatch = new CountDownLatch(1);
 
    /**
     * 初始化的时候，首先保证获取到zk的链接实例
     * @throws IOException
     * @throws InterruptedException
     */
    @BeforeAll
    public void connect() throws IOException, InterruptedException {
        zooKeeper = ZkUtils.getInstance();
    }
 
    /**
     * 使用完以后，关闭zk连接
     * @throws InterruptedException
     */
    @AfterAll
    public void close() throws InterruptedException {
        zooKeeper.close();
    }
 
    /**
     * 使用10个线程模拟抢锁
     */
    @Test
    public void testLock() throws InterruptedException {
        for(int i = 0; i < 10; i++){
            new Thread(){
                @Override
                public void run() {
                    String threadName =Thread.currentThread().getName();
                    LockWatch lockWatch = new LockWatch();
                    lockWatch.setThreadName(threadName);
                    lockWatch.setZooKeeper(zooKeeper);
 
                    try {
                        lockWatch.tryLock();
                        System.out.println(threadName + "deal business...");
                        lockWatch.unLock();
                    } catch (InterruptedException | KeeperException e) {
                        e.printStackTrace();
                    }
                }
            }.start();
        }
 
        countDownLatch.await();
    }
}
```

补充：zk服务中，2888端口用于follower调用leader进行写操作，3888端口为选主使用端口，2181端口为客户端连接zk服务节点端口。



https://adong.blog.csdn.net/article/details/82633588?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2~default~CTRLIST~Rate-1-82633588-blog-125216821.pc_relevant_aa&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2~default~CTRLIST~Rate-1-82633588-blog-125216821.pc_relevant_aa&utm_relevant_index=1


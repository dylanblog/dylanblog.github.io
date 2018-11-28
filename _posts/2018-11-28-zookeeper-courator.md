---
layout: post
title: "Zookeeper客户端Curator使用详解"
subtitle: ''
author: "DylanYu"
header-style: text
tags:
  - Zookeeper
  - Curator
---

<a href="https://www.jianshu.com/p/70151fc0ef5d" target="_blank">原文地址</a>

#### 简介
Curator是Netflix公司开源的一套zookeeper客户端框架，解决了很多Zookeeper客户端非常底层的细节开发工作，包括连接重连、反复注册Watcher和NodeExistsException异常等等。
Patrixck Hunt（Zookeeper）以一句“Guava is to Java that Curator to Zookeeper”给Curator予高度评价。

引子和趣闻：
> Zookeeper名字的由来是比较有趣的，下面的片段摘抄自《从PAXOS到ZOOKEEPER分布式一致性原理与实践》一书：
> Zookeeper最早起源于雅虎的研究院的一个研究小组。在当时，研究人员发现，在雅虎内部很多大型的系统需要依赖一个类似的系统进行分布式协调，但是这些系统往往存在分布式单点问题。
所以雅虎的开发人员就试图开发一个通用的无单点问题的分布式协调框架。在立项初期，考虑到很多项目都是用动物的名字来命名的(例如著名的Pig项目)，雅虎的工程师希望给这个项目也取一个动物的名字。
时任研究院的首席科学家Raghu Ramakrishnan开玩笑说：再这样下去，我们这儿就变成动物园了。此话一出，大家纷纷表示就叫动物园管理员吧——因为各个以动物命名的分布式组件放在一起，
雅虎的整个分布式系统看上去就像一个大型的动物园了，而Zookeeper正好用来进行分布式环境的协调——于是，Zookeeper的名字由此诞生了。

Curator无疑是Zookeeper客户端中的瑞士军刀，它译作"馆长"或者''管理者''，不知道是不是开发小组有意而为之，笔者猜测有可能这样命名的原因是说明Curator就是Zookeeper的馆长(脑洞有点大：Curator就是动物园的园长)。

Curator包含了几个包：
- curator-framework：对zookeeper的底层api的一些封装
- curator-client：提供一些客户端的操作，例如重试策略等
- curator-recipes：封装了一些高级特性，如：Cache事件监听、选举、分布式锁、分布式计数器、分布式Barrier等

Maven依赖(使用curator的版本：2.12.0，对应Zookeeper的版本为：3.4.x，如果跨版本会有兼容性问题，很有可能导致节点操作失败)：
```
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>2.12.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>2.12.0</version>
</dependency>
```
#### Curator的基本Api

创建会话
1.使用静态工程方法创建客户端

一个例子如下：

```
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
CuratorFramework client =
CuratorFrameworkFactory.newClient(
                        connectionInfo,
                        5000,
                        3000,
                        retryPolicy);
newClient静态工厂方法包含四个主要参数：
```

参数名|	说明
---|---
connectionString	|服务器列表，格式host1:port1,host2:port2,...
retryPolicy|	重试策略,内建有四种重试策略,也可以自行实现RetryPolicy接口
sessionTimeoutMs|	会话超时时间，单位毫秒，默认60000ms
connectionTimeoutMs|	连接创建超时时间，单位毫秒，默认60000ms

2.使用Fluent风格的Api创建会话
核心参数变为流式设置，一个列子如下：
```
 RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
 CuratorFramework client =
 CuratorFrameworkFactory.builder()
         .connectString(connectionInfo)
         .sessionTimeoutMs(5000)
         .connectionTimeoutMs(5000)
         .retryPolicy(retryPolicy)
         .build();
```

3.创建包含隔离命名空间的会话
为了实现不同的Zookeeper业务之间的隔离，需要为每个业务分配一个独立的命名空间（NameSpace），即指定一个Zookeeper的根路径（官方术语：为Zookeeper添加“Chroot”特性）。例如（下面的例子）当客户端指定了独立命名空间为“/base”，那么该客户端对Zookeeper上的数据节点的操作都是基于该目录进行的。通过设置Chroot可以将客户端应用与Zookeeper服务端的一课子树相对应，在多个应用共用一个Zookeeper集群的场景下，这对于实现不同应用之间的相互隔离十分有意义。

```
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
CuratorFramework client =
CuratorFrameworkFactory.builder()
        .connectString(connectionInfo)
        .sessionTimeoutMs(5000)
        .connectionTimeoutMs(5000)
        .retryPolicy(retryPolicy)
        .namespace("base")
        .build();
```

##### 启动客户端

当创建会话成功，得到client的实例然后可以直接调用其start( )方法：
```
client.start();
```

##### 数据节点操作

创建数据节点
Zookeeper的节点创建模式：

- PERSISTENT：持久化
- PERSISTENT_SEQUENTIAL：持久化并且带序列号
- EPHEMERAL：临时
- EPHEMERAL_SEQUENTIAL：临时并且带序列号

**创建一个节点，初始内容为空**

```
client.create().forPath("path");
注意：如果没有设置节点属性，节点创建模式默认为持久化节点，内容默认为空
```

创建一个节点，附带初始化内容

```
client.create().forPath("path","init".getBytes());
创建一个节点，指定创建模式（临时节点），内容为空

client.create().withMode(CreateMode.EPHEMERAL).forPath("path");
创建一个节点，指定创建模式（临时节点），附带初始化内容

client.create().withMode(CreateMode.EPHEMERAL).forPath("path","init".getBytes());
创建一个节点，指定创建模式（临时节点），附带初始化内容，并且自动递归创建父节点

client.create()
      .creatingParentContainersIfNeeded()
      .withMode(CreateMode.EPHEMERAL)
      .forPath("path","init".getBytes());
```

这个creatingParentContainersIfNeeded()接口非常有用，因为一般情况开发人员在创建一个子节点必须判断它的父节点是否存在，
如果不存在直接创建会抛出NoNodeException，使用creatingParentContainersIfNeeded()之后Curator能够自动递归创建所有所需的父节点。

##### 删除数据节点

删除一个节点

```
client.delete().forPath("path");
注意，此方法只能删除叶子节点，否则会抛出异常。

删除一个节点，并且递归删除其所有的子节点

client.delete().deletingChildrenIfNeeded().forPath("path");
删除一个节点，强制指定版本进行删除

client.delete().withVersion(10086).forPath("path");
删除一个节点，强制保证删除

client.delete().guaranteed().forPath("path");
guaranteed()接口是一个保障措施，只要客户端会话有效，那么Curator会在后台持续进行删除操作，直到删除节点成功。
```

注意：上面的多个流式接口是可以自由组合的，例如：

```
client.delete().guaranteed().deletingChildrenIfNeeded().withVersion(10086).forPath("path");
```

##### 读取数据节点数据

读取一个节点的数据内容

```
client.getData().forPath("path");
注意，此方法返的返回值是byte[ ];

读取一个节点的数据内容，同时获取到该节点的stat

Stat stat = new Stat();
client.getData().storingStatIn(stat).forPath("path");
```

##### 更新数据节点数据

更新一个节点的数据内容

```
client.setData().forPath("path","data".getBytes());
注意：该接口会返回一个Stat实例

更新一个节点的数据内容，强制指定版本进行更新

client.setData().withVersion(10086).forPath("path","data".getBytes());
检查节点是否存在
client.checkExists().forPath("path");
注意：该方法返回一个Stat实例，用于检查ZNode是否存在的操作. 可以调用额外的方法(监控或者后台处理)并在最后调用forPath( )指定要操作的ZNode

获取某个节点的所有子节点路径
client.getChildren().forPath("path");
注意：该方法的返回值为List<String>,获得ZNode的子节点Path列表。 可以调用额外的方法(监控、后台处理或者获取状态watch, background or get stat) 并在最后调用forPath()指定要操作的父ZNode
```

##### 事务

CuratorFramework的实例包含inTransaction( )接口方法，调用此方法开启一个ZooKeeper事务. 可以复合create, setData, check, and/or delete 等操作然后调用commit()作为一个原子操作提交。一个例子如下：
```
client.inTransaction().check().forPath("path")
      .and()
      .create().withMode(CreateMode.EPHEMERAL).forPath("path","data".getBytes())
      .and()
      .setData().withVersion(10086).forPath("path","data2".getBytes())
      .and()
      .commit();
```
##### 异步接口

上面提到的创建、删除、更新、读取等方法都是同步的，Curator提供异步接口，引入了BackgroundCallback接口用于处理异步接口调用之后服务端返回的结果信息。
BackgroundCallback接口中一个重要的回调值为CuratorEvent，里面包含事件类型、响应吗和节点的详细信息。

CuratorEventType

事件类型	|对应CuratorFramework实例的方法
---|---
CREATE	|#create()
DELETE	|#delete()
EXISTS	|#checkExists()
GET_DATA|	#getData()
SET_DATA|	#setData()
CHILDREN|	#getChildren()
SYNC	|#sync(String,Object)
GET_ACL	|#getACL()
SET_ACL	|#setACL()
WATCHED	|#Watcher(Watcher)
CLOSING	|#close()

响应码(#getResultCode())

响应码	|意义
---|---
0	|OK，即调用成功
-4	|ConnectionLoss，即客户端与服务端断开连接
-110	|NodeExists，即节点已经存在
-112	|SessionExpired，即会话过期

一个异步创建节点的例子如下：

```
Executor executor = Executors.newFixedThreadPool(2);
client.create()
      .creatingParentsIfNeeded()
      .withMode(CreateMode.EPHEMERAL)
      .inBackground((curatorFramework, curatorEvent) -> {
        System.out.println(String.format("eventType:%s,resultCode:%s",curatorEvent.getType(),curatorEvent.getResultCode()));
      },executor)
      .forPath("path");
```

注意：如果#inBackground()方法不指定executor，那么会默认使用Curator的EventThread去进行异步处理。

#### Curator食谱(高级特性)

提醒：首先你必须添加curator-recipes依赖，下文仅仅对recipes一些特性的使用进行解释和举例，不打算进行源码级别的探讨
```
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>2.12.0</version>
</dependency>
```
重要提醒：强烈推荐使用ConnectionStateListener监控连接的状态，当连接状态为LOST，curator-recipes下的所有Api将会失效或者过期，尽管后面所有的例子都没有使用到ConnectionStateListener。

##### 缓存
Zookeeper原生支持通过注册Watcher来进行事件监听，但是开发者需要反复注册(Watcher只能单次注册单次使用)。Cache是Curator中对事件监听的包装，
可以看作是对事件监听的本地缓存视图，能够自动为开发者处理反复注册监听。Curator提供了三种Watcher(Cache)来监听结点的变化。

##### Path Cache

Path Cache用来监控一个ZNode的子节点. 当一个子节点增加， 更新，删除时， Path Cache会改变它的状态， 会包含最新的子节点， 子节点的数据和状态，而状态的更变将通过PathChildrenCacheListener通知。

实际使用时会涉及到四个类：

- PathChildrenCache
- PathChildrenCacheEvent
- PathChildrenCacheListener

ChildData
通过下面的构造函数创建Path Cache:

```
public PathChildrenCache(CuratorFramework client, String path, boolean cacheData)
```

想使用cache，必须调用它的start方法，使用完后调用close方法。 可以设置StartMode来实现启动的模式，

StartMode有下面几种：

- NORMAL：正常初始化。
- BUILD_INITIAL_CACHE：在调用start()之前会调用rebuild()。
- POST_INITIALIZED_EVENT： 当Cache初始化数据后发送一个PathChildrenCacheEvent.Type#INITIALIZED事件

public void addListener(PathChildrenCacheListener listener)可以增加listener监听缓存的变化。

getCurrentData()方法返回一个List<ChildData>对象，可以遍历所有的子节点。

##### 设置/更新、移除其实是使用client (CuratorFramework)来操作, 不通过PathChildrenCache操作：
```
public class PathCacheDemo {

    private static final String PATH = "/example/pathCache";

    public static void main(String[] args) throws Exception {
        TestingServer server = new TestingServer();
        CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
        client.start();
        PathChildrenCache cache = new PathChildrenCache(client, PATH, true);
        cache.start();
        PathChildrenCacheListener cacheListener = (client1, event) -> {
            System.out.println("事件类型：" + event.getType());
            if (null != event.getData()) {
                System.out.println("节点数据：" + event.getData().getPath() + " = " + new String(event.getData().getData()));
            }
        };
        cache.getListenable().addListener(cacheListener);
        client.create().creatingParentsIfNeeded().forPath("/example/pathCache/test01", "01".getBytes());
        Thread.sleep(10);
        client.create().creatingParentsIfNeeded().forPath("/example/pathCache/test02", "02".getBytes());
        Thread.sleep(10);
        client.setData().forPath("/example/pathCache/test01", "01_V2".getBytes());
        Thread.sleep(10);
        for (ChildData data : cache.getCurrentData()) {
            System.out.println("getCurrentData:" + data.getPath() + " = " + new String(data.getData()));
        }
        client.delete().forPath("/example/pathCache/test01");
        Thread.sleep(10);
        client.delete().forPath("/example/pathCache/test02");
        Thread.sleep(1000 * 5);
        cache.close();
        client.close();
        System.out.println("OK!");
    }
}
```

注意：如果new PathChildrenCache(client, PATH, true)中的参数cacheData值设置为false，则示例中的event.getData().getData()、data.getData()将返回null，cache将不会缓存节点数据。

注意：示例中的Thread.sleep(10)可以注释掉，但是注释后事件监听的触发次数会不全，这可能与PathCache的实现原理有关，不能太过频繁的触发事件！

#### Node Cache

Node Cache与Path Cache类似，Node Cache只是监听某一个特定的节点。它涉及到下面的三个类：

- NodeCache - Node Cache实现类
- NodeCacheListener - 节点监听器
- ChildData - 节点数据

注意：使用cache，依然要调用它的start()方法，使用完后调用close()方法。

getCurrentData()将得到节点当前的状态，通过它的状态可以得到当前的值。

```
public class NodeCacheDemo {

    private static final String PATH = "/example/cache";

    public static void main(String[] args) throws Exception {
        TestingServer server = new TestingServer();
        CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
        client.start();
        client.create().creatingParentsIfNeeded().forPath(PATH);
        final NodeCache cache = new NodeCache(client, PATH);
        NodeCacheListener listener = () -> {
            ChildData data = cache.getCurrentData();
            if (null != data) {
                System.out.println("节点数据：" + new String(cache.getCurrentData().getData()));
            } else {
                System.out.println("节点被删除!");
            }
        };
        cache.getListenable().addListener(listener);
        cache.start();
        client.setData().forPath(PATH, "01".getBytes());
        Thread.sleep(100);
        client.setData().forPath(PATH, "02".getBytes());
        Thread.sleep(100);
        client.delete().deletingChildrenIfNeeded().forPath(PATH);
        Thread.sleep(1000 * 2);
        cache.close();
        client.close();
        System.out.println("OK!");
    }
}
```

注意：示例中的Thread.sleep(10)可以注释，但是注释后事件监听的触发次数会不全，这可能与NodeCache的实现原理有关，不能太过频繁的触发事件！

注意：NodeCache只能监听一个节点的状态变化。

#### Tree Cache

Tree Cache可以监控整个树上的所有节点，类似于PathCache和NodeCache的组合，主要涉及到下面四个类：

- TreeCache - Tree Cache实现类
- TreeCacheListener - 监听器类
- TreeCacheEvent - 触发的事件类
- ChildData - 节点数据

```
public class TreeCacheDemo {

    private static final String PATH = "/example/cache";

    public static void main(String[] args) throws Exception {
        TestingServer server = new TestingServer();
        CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
        client.start();
        client.create().creatingParentsIfNeeded().forPath(PATH);
        TreeCache cache = new TreeCache(client, PATH);
        TreeCacheListener listener = (client1, event) ->
                System.out.println("事件类型：" + event.getType() +
                        " | 路径：" + (null != event.getData() ? event.getData().getPath() : null));
        cache.getListenable().addListener(listener);
        cache.start();
        client.setData().forPath(PATH, "01".getBytes());
        Thread.sleep(100);
        client.setData().forPath(PATH, "02".getBytes());
        Thread.sleep(100);
        client.delete().deletingChildrenIfNeeded().forPath(PATH);
        Thread.sleep(1000 * 2);
        cache.close();
        client.close();
        System.out.println("OK!");
    }
}
```

注意：在此示例中没有使用Thread.sleep(10)，但是事件触发次数也是正常的。

注意：TreeCache在初始化(调用start()方法)的时候会回调TreeCacheListener实例一个事TreeCacheEvent，而回调的TreeCacheEvent对象的Type为INITIALIZED，ChildData为null，此时event.getData().getPath()很有可能导致空指针异常，这里应该主动处理并避免这种情况。

#### Leader选举

在分布式计算中， leader elections是很重要的一个功能， 这个选举过程是这样子的： 指派一个进程作为组织者，将任务分发给各节点。 在任务开始前， 哪个节点都不知道谁是leader(领导者)或者coordinator(协调者). 当选举算法开始执行后，
每个节点最终会得到一个唯一的节点作为任务leader. 除此之外， 选举还经常会发生在leader意外宕机的情况下，新的leader要被选举出来。

在zookeeper集群中，leader负责写操作，然后通过Zab协议实现follower的同步，leader或者follower都可以处理读操作。

##### Curator 有两种leader选举的recipe,分别是LeaderSelector和LeaderLatch。

前者是所有存活的客户端不间断的轮流做Leader，大同社会。后者是一旦选举出Leader，除非有客户端挂掉重新触发选举，否则不会交出领导权。某党?

##### LeaderLatch

LeaderLatch有两个构造函数：

```
public LeaderLatch(CuratorFramework client, String latchPath)
public LeaderLatch(CuratorFramework client, String latchPath,  String id)
```

LeaderLatch的启动：

```
leaderLatch.start( );
```

一旦启动，LeaderLatch会和其它使用相同latch path的其它LeaderLatch交涉，然后其中一个最终会被选举为leader，可以通过hasLeadership方法查看LeaderLatch实例是否leader：

```
leaderLatch.hasLeadership( ); //返回true说明当前实例是leader
```

类似JDK的CountDownLatch， LeaderLatch在请求成为leadership会block(阻塞)，一旦不使用LeaderLatch了，必须调用close方法。 如果它是leader,会释放leadership， 其它的参与者将会选举一个leader。

```
public void await() throws InterruptedException,EOFException
/*Causes the current thread to wait until this instance acquires leadership
unless the thread is interrupted or closed.*/
public boolean await(long timeout,TimeUnit unit)throws InterruptedException
```

异常处理： LeaderLatch实例可以增加ConnectionStateListener来监听网络连接问题。 当 SUSPENDED 或 LOST 时, leader不再认为自己还是leader。
当LOST后连接重连后RECONNECTED,LeaderLatch会删除先前的ZNode然后重新创建一个。LeaderLatch用户必须考虑导致leadership丢失的连接问题。

强烈推荐你使用ConnectionStateListener。

一个LeaderLatch的使用例子：
```
public class LeaderLatchDemo extends BaseConnectionInfo {
    protected static String PATH = "/francis/leader";
    private static final int CLIENT_QTY = 10;


    public static void main(String[] args) throws Exception {
        List<CuratorFramework> clients = Lists.newArrayList();
        List<LeaderLatch> examples = Lists.newArrayList();
        TestingServer server=new TestingServer();
        try {
            for (int i = 0; i < CLIENT_QTY; i++) {
                CuratorFramework client
                        = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(20000, 3));
                clients.add(client);
                LeaderLatch latch = new LeaderLatch(client, PATH, "Client #" + i);
                latch.addListener(new LeaderLatchListener() {

                    @Override
                    public void isLeader() {
                        // TODO Auto-generated method stub
                        System.out.println("I am Leader");
                    }

                    @Override
                    public void notLeader() {
                        // TODO Auto-generated method stub
                        System.out.println("I am not Leader");
                    }
                });
                examples.add(latch);
                client.start();
                latch.start();
            }
            Thread.sleep(10000);
            LeaderLatch currentLeader = null;
            for (LeaderLatch latch : examples) {
                if (latch.hasLeadership()) {
                    currentLeader = latch;
                }
            }
            System.out.println("current leader is " + currentLeader.getId());
            System.out.println("release the leader " + currentLeader.getId());
            currentLeader.close();

            Thread.sleep(5000);

            for (LeaderLatch latch : examples) {
                if (latch.hasLeadership()) {
                    currentLeader = latch;
                }
            }
            System.out.println("current leader is " + currentLeader.getId());
            System.out.println("release the leader " + currentLeader.getId());
        } finally {
            for (LeaderLatch latch : examples) {
                if (null != latch.getState())
                CloseableUtils.closeQuietly(latch);
            }
            for (CuratorFramework client : clients) {
                CloseableUtils.closeQuietly(client);
            }
        }
    }
}
```

可以添加test module的依赖方便进行测试，不需要启动真实的zookeeper服务端：
```
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-test</artifactId>
    <version>2.12.0</version>
</dependency>
```

首先我们创建了10个LeaderLatch，启动后它们中的一个会被选举为leader。 因为选举会花费一些时间，start后并不能马上就得到leader。

通过hasLeadership查看自己是否是leader， 如果是的话返回true。

可以通过.getLeader().getId()可以得到当前的leader的ID。

只能通过close释放当前的领导权。

await是一个阻塞方法， 尝试获取leader地位，但是未必能上位。

##### LeaderSelector
LeaderSelector使用的时候主要涉及下面几个类：

- LeaderSelector
- LeaderSelectorListener
- LeaderSelectorListenerAdapter
- CancelLeadershipException

核心类是LeaderSelector，它的构造函数如下：

```
public LeaderSelector(CuratorFramework client, String mutexPath,LeaderSelectorListener listener)
public LeaderSelector(CuratorFramework client, String mutexPath, ThreadFactory threadFactory, Executor executor, LeaderSelectorListener listener)
```

类似LeaderLatch,LeaderSelector必须start: leaderSelector.start(); 一旦启动，当实例取得领导权时你的listener的takeLeadership()方法被调用。
而takeLeadership()方法只有领导权被释放时才返回。 当你不再使用LeaderSelector实例时，应该调用它的close方法。

异常处理 LeaderSelectorListener类继承ConnectionStateListener。LeaderSelector必须小心连接状态的改变。如果实例成为leader, 它应该响应SUSPENDED 或 LOST。 当 SUSPENDED 状态出现时，
 实例必须假定在重新连接成功之前它可能不再是leader了。 如果LOST状态出现， 实例不再是leader， takeLeadership方法返回。

重要: 推荐处理方式是当收到SUSPENDED 或 LOST时抛出CancelLeadershipException异常.。这会导致LeaderSelector实例中断并取消执行takeLeadership方法的异常.。
这非常重要， 你必须考虑扩展LeaderSelectorListenerAdapter. LeaderSelectorListenerAdapter提供了推荐的处理逻辑。

下面的一个例子摘抄自官方：
```
public class LeaderSelectorAdapter extends LeaderSelectorListenerAdapter implements Closeable {
    private final String name;
    private final LeaderSelector leaderSelector;
    private final AtomicInteger leaderCount = new AtomicInteger();

    public LeaderSelectorAdapter(CuratorFramework client, String path, String name) {
        this.name = name;
        leaderSelector = new LeaderSelector(client, path, this);
        leaderSelector.autoRequeue();
    }

    public void start() throws IOException {
        leaderSelector.start();
    }

    @Override
    public void close() throws IOException {
        leaderSelector.close();
    }

    @Override
    public void takeLeadership(CuratorFramework client) throws Exception {
        final int waitSeconds = (int) (5 * Math.random()) + 1;
        System.out.println(name + " is now the leader. Waiting " + waitSeconds + " seconds...");
        System.out.println(name + " has been leader " + leaderCount.getAndIncrement() + " time(s) before.");
        try {
            Thread.sleep(TimeUnit.SECONDS.toMillis(waitSeconds));
        } catch (InterruptedException e) {
            System.err.println(name + " was interrupted.");
            Thread.currentThread().interrupt();
        } finally {
            System.out.println(name + " relinquishing leadership.\n");
        }
    }
}
```

你可以在takeLeadership进行任务的分配等等，并且不要返回，如果你想要要此实例一直是leader的话可以加一个死循环。调用 leaderSelector.autoRequeue();保证在此实例释放领导权之后还可能获得领导权。
在这里我们使用AtomicInteger来记录此client获得领导权的次数， 它是”fair”， 每个client有平等的机会获得领导权。

```
public class LeaderSelectorDemo {

    protected static String PATH = "/francis/leader";
    private static final int CLIENT_QTY = 10;


    public static void main(String[] args) throws Exception {
        List<CuratorFramework> clients = Lists.newArrayList();
        List<LeaderSelectorAdapter> examples = Lists.newArrayList();
        TestingServer server = new TestingServer();
        try {
            for (int i = 0; i < CLIENT_QTY; i++) {
                CuratorFramework client
                        = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(20000, 3));
                clients.add(client);
                LeaderSelectorAdapter selectorAdapter = new LeaderSelectorAdapter(client, PATH, "Client #" + i);
                examples.add(selectorAdapter);
                client.start();
                selectorAdapter.start();
            }
            System.out.println("Press enter/return to quit\n");
            new BufferedReader(new InputStreamReader(System.in)).readLine();
        } finally {
            System.out.println("Shutting down...");
            for (LeaderSelectorAdapter exampleClient : examples) {
                CloseableUtils.closeQuietly(exampleClient);
            }
            for (CuratorFramework client : clients) {
                CloseableUtils.closeQuietly(client);
            }
            CloseableUtils.closeQuietly(server);
        }
    }
}
```

对比可知，LeaderLatch必须调用close()方法才会释放领导权，而对于LeaderSelector，通过LeaderSelectorListener可以对领导权进行控制， 在适当的时候释放领导权，这样每个节点都有可能获得领导权。从而，LeaderSelector具有更好的灵活性和可控性，建议有LeaderElection应用场景下优先使用LeaderSelector。

##### 分布式锁

提醒：

1.推荐使用ConnectionStateListener监控连接的状态，因为当连接LOST时你不再拥有锁

2.分布式的锁全局同步， 这意味着任何一个时间点不会有两个客户端都拥有相同的锁。

##### 可重入共享锁—Shared Reentrant Lock

Shared意味着锁是全局可见的， 客户端都可以请求锁。 Reentrant和JDK的ReentrantLock类似，即可重入， 意味着同一个客户端在拥有锁的同时，可以多次获取，不会被阻塞。 它是由类InterProcessMutex来实现。 它的构造函数为：

```
public InterProcessMutex(CuratorFramework client, String path)
通过acquire()获得锁，并提供超时机制：

public void acquire()
Acquire the mutex - blocking until it's available. Note: the same thread can call acquire
re-entrantly. Each call to acquire must be balanced by a call to release()

public boolean acquire(long time,TimeUnit unit)
Acquire the mutex - blocks until it's available or the given time expires. Note: the same thread can call acquire re-entrantly. Each call to acquire that returns true must be balanced by a call to release()
```

Parameters:
- time - time to wait
- unit - time unit
Returns:
- true if the mutex was acquired, false if not

通过release()方法释放锁。 InterProcessMutex 实例可以重用。

Revoking ZooKeeper recipes wiki定义了可协商的撤销机制。 为了撤销mutex, 调用下面的方法：

```
public void makeRevocable(RevocationListener<T> listener)
将锁设为可撤销的. 当别的进程或线程想让你释放锁时Listener会被调用。
```

Parameters:
- listener - the listener

如果你请求撤销当前的锁， 调用attemptRevoke()方法,注意锁释放时RevocationListener将会回调。

```
public static void attemptRevoke(CuratorFramework client,String path) throws Exception
Utility to mark a lock for revocation. Assuming that the lock has been registered
with a RevocationListener, it will get called and the lock should be released. Note,
however, that revocation is cooperative.
```

Parameters:
- client - the client
- path - the path of the lock - usually from something like InterProcessMutex.getParticipantNodes()

二次提醒：错误处理 还是强烈推荐你使用ConnectionStateListener处理连接状态的改变。 当连接LOST时你不再拥有锁。

首先让我们创建一个模拟的共享资源， 这个资源期望只能单线程的访问，否则会有并发问题。

```
public class FakeLimitedResource {
    private final AtomicBoolean inUse = new AtomicBoolean(false);

    public void use() throws InterruptedException {
        // 真实环境中我们会在这里访问/维护一个共享的资源
        //这个例子在使用锁的情况下不会非法并发异常IllegalStateException
        //但是在无锁的情况由于sleep了一段时间，很容易抛出异常
        if (!inUse.compareAndSet(false, true)) {
            throw new IllegalStateException("Needs to be used by one client at a time");
        }
        try {
            Thread.sleep((long) (3 * Math.random()));
        } finally {
            inUse.set(false);
        }
    }
}
```

然后创建一个InterProcessMutexDemo类， 它负责请求锁， 使用资源，释放锁这样一个完整的访问过程。
```
public class InterProcessMutexDemo {

    private InterProcessMutex lock;
    private final FakeLimitedResource resource;
    private final String clientName;

    public InterProcessMutexDemo(CuratorFramework client, String lockPath, FakeLimitedResource resource, String clientName) {
        this.resource = resource;
        this.clientName = clientName;
        this.lock = new InterProcessMutex(client, lockPath);
    }

    public void doWork(long time, TimeUnit unit) throws Exception {
        if (!lock.acquire(time, unit)) {
            throw new IllegalStateException(clientName + " could not acquire the lock");
        }
        try {
            System.out.println(clientName + " get the lock");
            resource.use(); //access resource exclusively
        } finally {
            System.out.println(clientName + " releasing the lock");
            lock.release(); // always release the lock in a finally block
        }
    }

    private static final int QTY = 5;
    private static final int REPETITIONS = QTY * 10;
    private static final String PATH = "/examples/locks";

    public static void main(String[] args) throws Exception {
        final FakeLimitedResource resource = new FakeLimitedResource();
        ExecutorService service = Executors.newFixedThreadPool(QTY);
        final TestingServer server = new TestingServer();
        try {
            for (int i = 0; i < QTY; ++i) {
                final int index = i;
                Callable<Void> task = new Callable<Void>() {
                    @Override
                    public Void call() throws Exception {
                        CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
                        try {
                            client.start();
                            final InterProcessMutexDemo example = new InterProcessMutexDemo(client, PATH, resource, "Client " + index);
                            for (int j = 0; j < REPETITIONS; ++j) {
                                example.doWork(10, TimeUnit.SECONDS);
                            }
                        } catch (Throwable e) {
                            e.printStackTrace();
                        } finally {
                            CloseableUtils.closeQuietly(client);
                        }
                        return null;
                    }
                };
                service.submit(task);
            }
            service.shutdown();
            service.awaitTermination(10, TimeUnit.MINUTES);
        } finally {
            CloseableUtils.closeQuietly(server);
        }
    }
}
```

代码也很简单，生成10个client， 每个client重复执行10次 请求锁–访问资源–释放锁的过程。每个client都在独立的线程中。 结果可以看到，锁是随机的被每个实例排他性的使用。

既然是可重用的，你可以在一个线程中多次调用acquire(),在线程拥有锁时它总是返回true。

你不应该在多个线程中用同一个InterProcessMutex， 你可以在每个线程中都生成一个新的InterProcessMutex实例，它们的path都一样，这样它们可以共享同一个锁。

#### 不可重入共享锁—Shared Lock

这个锁和上面的InterProcessMutex相比，就是少了Reentrant的功能，也就意味着它不能在同一个线程中重入。这个类是InterProcessSemaphoreMutex,使用方法和InterProcessMutex类似
```
public class InterProcessSemaphoreMutexDemo {

    private InterProcessSemaphoreMutex lock;
    private final FakeLimitedResource resource;
    private final String clientName;

    public InterProcessSemaphoreMutexDemo(CuratorFramework client, String lockPath, FakeLimitedResource resource, String clientName) {
        this.resource = resource;
        this.clientName = clientName;
        this.lock = new InterProcessSemaphoreMutex(client, lockPath);
    }

    public void doWork(long time, TimeUnit unit) throws Exception {
        if (!lock.acquire(time, unit))
        {
            throw new IllegalStateException(clientName + " 不能得到互斥锁");
        }
        System.out.println(clientName + " 已获取到互斥锁");
        if (!lock.acquire(time, unit))
        {
            throw new IllegalStateException(clientName + " 不能得到互斥锁");
        }
        System.out.println(clientName + " 再次获取到互斥锁");
        try {
            System.out.println(clientName + " get the lock");
            resource.use(); //access resource exclusively
        } finally {
            System.out.println(clientName + " releasing the lock");
            lock.release(); // always release the lock in a finally block
            lock.release(); // 获取锁几次 释放锁也要几次
        }
    }

    private static final int QTY = 5;
    private static final int REPETITIONS = QTY * 10;
    private static final String PATH = "/examples/locks";

    public static void main(String[] args) throws Exception {
        final FakeLimitedResource resource = new FakeLimitedResource();
        ExecutorService service = Executors.newFixedThreadPool(QTY);
        final TestingServer server = new TestingServer();
        try {
            for (int i = 0; i < QTY; ++i) {
                final int index = i;
                Callable<Void> task = new Callable<Void>() {
                    @Override
                    public Void call() throws Exception {
                        CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
                        try {
                            client.start();
                            final InterProcessSemaphoreMutexDemo example = new InterProcessSemaphoreMutexDemo(client, PATH, resource, "Client " + index);
                            for (int j = 0; j < REPETITIONS; ++j) {
                                example.doWork(10, TimeUnit.SECONDS);
                            }
                        } catch (Throwable e) {
                            e.printStackTrace();
                        } finally {
                            CloseableUtils.closeQuietly(client);
                        }
                        return null;
                    }
                };
                service.submit(task);
            }
            service.shutdown();
            service.awaitTermination(10, TimeUnit.MINUTES);
        } finally {
            CloseableUtils.closeQuietly(server);
        }
        Thread.sleep(Integer.MAX_VALUE);
    }
}
```

运行后发现，有且只有一个client成功获取第一个锁(第一个acquire()方法返回true)，然后它自己阻塞在第二个acquire()方法，获取第二个锁超时；其他所有的客户端都阻塞在第一个acquire()方法超时并且抛出异常。

这样也就验证了InterProcessSemaphoreMutex实现的锁是不可重入的。

#### 可重入读写锁—Shared Reentrant Read Write Lock

类似JDK的ReentrantReadWriteLock。一个读写锁管理一对相关的锁。一个负责读操作，另外一个负责写操作。读操作在写锁没被使用时可同时由多个进程使用，而写锁在使用时不允许读(阻塞)。

此锁是可重入的。一个拥有写锁的线程可重入读锁，但是读锁却不能进入写锁。这也意味着写锁可以降级成读锁， 比如请求写锁 --->请求读锁--->释放读锁 ---->释放写锁。从读锁升级成写锁是不行的。

可重入读写锁主要由两个类实现：InterProcessReadWriteLock、InterProcessMutex。使用时首先创建一个InterProcessReadWriteLock实例，然后再根据你的需求得到读锁或者写锁，读写锁的类型是InterProcessMutex。

```
public class ReentrantReadWriteLockDemo {

    private final InterProcessReadWriteLock lock;
    private final InterProcessMutex readLock;
    private final InterProcessMutex writeLock;
    private final FakeLimitedResource resource;
    private final String clientName;

    public ReentrantReadWriteLockDemo(CuratorFramework client, String lockPath, FakeLimitedResource resource, String clientName) {
        this.resource = resource;
        this.clientName = clientName;
        lock = new InterProcessReadWriteLock(client, lockPath);
        readLock = lock.readLock();
        writeLock = lock.writeLock();
    }

    public void doWork(long time, TimeUnit unit) throws Exception {
        // 注意只能先得到写锁再得到读锁，不能反过来！！！
        if (!writeLock.acquire(time, unit)) {
            throw new IllegalStateException(clientName + " 不能得到写锁");
        }
        System.out.println(clientName + " 已得到写锁");
        if (!readLock.acquire(time, unit)) {
            throw new IllegalStateException(clientName + " 不能得到读锁");
        }
        System.out.println(clientName + " 已得到读锁");
        try {
            resource.use(); // 使用资源
            Thread.sleep(1000);
        } finally {
            System.out.println(clientName + " 释放读写锁");
            readLock.release();
            writeLock.release();
        }
    }

    private static final int QTY = 5;
    private static final int REPETITIONS = QTY ;
    private static final String PATH = "/examples/locks";

    public static void main(String[] args) throws Exception {
        final FakeLimitedResource resource = new FakeLimitedResource();
        ExecutorService service = Executors.newFixedThreadPool(QTY);
        final TestingServer server = new TestingServer();
        try {
            for (int i = 0; i < QTY; ++i) {
                final int index = i;
                Callable<Void> task = new Callable<Void>() {
                    @Override
                    public Void call() throws Exception {
                        CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
                        try {
                            client.start();
                            final ReentrantReadWriteLockDemo example = new ReentrantReadWriteLockDemo(client, PATH, resource, "Client " + index);
                            for (int j = 0; j < REPETITIONS; ++j) {
                                example.doWork(10, TimeUnit.SECONDS);
                            }
                        } catch (Throwable e) {
                            e.printStackTrace();
                        } finally {
                            CloseableUtils.closeQuietly(client);
                        }
                        return null;
                    }
                };
                service.submit(task);
            }
            service.shutdown();
            service.awaitTermination(10, TimeUnit.MINUTES);
        } finally {
            CloseableUtils.closeQuietly(server);
        }
    }
}
```

#### 信号量—Shared Semaphore

一个计数的信号量类似JDK的Semaphore。 JDK中Semaphore维护的一组许可(permits)，而Curator中称之为租约(Lease)。 有两种方式可以决定semaphore的最大租约数。
第一种方式是用户给定path并且指定最大LeaseSize。第二种方式用户给定path并且使用SharedCountReader类。如果不使用SharedCountReader,
必须保证所有实例在多进程中使用相同的(最大)租约数量,否则有可能出现A进程中的实例持有最大租约数量为10，但是在B进程中持有的最大租约数量为20，此时租约的意义就失效了。

这次调用acquire()会返回一个租约对象。 客户端必须在finally中close这些租约对象，否则这些租约会丢失掉。 但是， 但是，如果客户端session由于某种原因比如crash丢掉，
那么这些客户端持有的租约会自动close， 这样其它客户端可以继续使用这些租约。 租约还可以通过下面的方式返还：

```
public void returnAll(Collection<Lease> leases)
public void returnLease(Lease lease)
注意你可以一次性请求多个租约，如果Semaphore当前的租约不够，则请求线程会被阻塞。 同时还提供了超时的重载方法。

public Lease acquire()
public Collection<Lease> acquire(int qty)
public Lease acquire(long time, TimeUnit unit)
public Collection<Lease> acquire(int qty, long time, TimeUnit unit)
Shared Semaphore使用的主要类包括下面几个：

InterProcessSemaphoreV2
Lease
SharedCountReader
public class InterProcessSemaphoreDemo {

    private static final int MAX_LEASE = 10;
    private static final String PATH = "/examples/locks";

    public static void main(String[] args) throws Exception {
        FakeLimitedResource resource = new FakeLimitedResource();
        try (TestingServer server = new TestingServer()) {

            CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
            client.start();

            InterProcessSemaphoreV2 semaphore = new InterProcessSemaphoreV2(client, PATH, MAX_LEASE);
            Collection<Lease> leases = semaphore.acquire(5);
            System.out.println("get " + leases.size() + " leases");
            Lease lease = semaphore.acquire();
            System.out.println("get another lease");

            resource.use();

            Collection<Lease> leases2 = semaphore.acquire(5, 10, TimeUnit.SECONDS);
            System.out.println("Should timeout and acquire return " + leases2);

            System.out.println("return one lease");
            semaphore.returnLease(lease);
            System.out.println("return another 5 leases");
            semaphore.returnAll(leases);
        }
    }
}
```

首先我们先获得了5个租约， 最后我们把它还给了semaphore。 接着请求了一个租约，因为semaphore还有5个租约，所以请求可以满足，返回一个租约，还剩4个租约。 然后再请求一个租约，
因为租约不够，阻塞到超时，还是没能满足，返回结果为null(租约不足会阻塞到超时，然后返回null，不会主动抛出异常；如果不设置超时时间，会一致阻塞)。

上面说讲的锁都是公平锁(fair)。 总ZooKeeper的角度看， 每个客户端都按照请求的顺序获得锁，不存在非公平的抢占的情况。

#### 多共享锁对象 —Multi Shared Lock

Multi Shared Lock是一个锁的容器。 当调用acquire()， 所有的锁都会被acquire()，如果请求失败，所有的锁都会被release。 同样调用release时所有的锁都被release(失败被忽略)。
 基本上，它就是组锁的代表，在它上面的请求释放操作都会传递给它包含的所有的锁。

主要涉及两个类：

- InterProcessMultiLock
- InterProcessLock
它的构造函数需要包含的锁的集合，或者一组ZooKeeper的path。

```
public InterProcessMultiLock(List<InterProcessLock> locks)
public InterProcessMultiLock(CuratorFramework client, List<String> paths)
```

用法和Shared Lock相同。

```
public class MultiSharedLockDemo {

    private static final String PATH1 = "/examples/locks1";
    private static final String PATH2 = "/examples/locks2";

    public static void main(String[] args) throws Exception {
        FakeLimitedResource resource = new FakeLimitedResource();
        try (TestingServer server = new TestingServer()) {
            CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
            client.start();

            InterProcessLock lock1 = new InterProcessMutex(client, PATH1);
            InterProcessLock lock2 = new InterProcessSemaphoreMutex(client, PATH2);

            InterProcessMultiLock lock = new InterProcessMultiLock(Arrays.asList(lock1, lock2));

            if (!lock.acquire(10, TimeUnit.SECONDS)) {
                throw new IllegalStateException("could not acquire the lock");
            }
            System.out.println("has got all lock");

            System.out.println("has got lock1: " + lock1.isAcquiredInThisProcess());
            System.out.println("has got lock2: " + lock2.isAcquiredInThisProcess());

            try {
                resource.use(); //access resource exclusively
            } finally {
                System.out.println("releasing the lock");
                lock.release(); // always release the lock in a finally block
            }
            System.out.println("has got lock1: " + lock1.isAcquiredInThisProcess());
            System.out.println("has got lock2: " + lock2.isAcquiredInThisProcess());
        }
    }
}
```

新建一个InterProcessMultiLock， 包含一个重入锁和一个非重入锁。 调用acquire()后可以看到线程同时拥有了这两个锁。 调用release()看到这两个锁都被释放了。

**最后再重申一次， 强烈推荐使用ConnectionStateListener监控连接的状态，当连接状态为LOST，锁将会丢失。**
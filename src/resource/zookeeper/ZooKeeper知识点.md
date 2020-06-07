## 痛点

在没有注册中心时候，服务间调用需要知道被调方的地址或者代理地址，当服务更换部署地址，就不得不修改调用当中指定的地址或者修改代理配置。

而有了注册中心之后，每个服务在调用别人的时候只需要知道服务名称就好，继续地址都会通过注册中心同步过来。

因此注册中心解决了服务之间的自动发现。举个栗子：

你去商场逛街，你作为消费端，商家作为服务端，你怎么找到想去的商家？

没有注册中心就类似靠记忆去寻找，假设商家从一楼搬到了二楼，你之前记得商家是在一楼，去了后发现商家没了，然后给商家打电话问搬哪了，你才知道商家搬到二楼了

注册中心就类似一个商家门店导览图，你从导览图上可以清晰了解商家在商场几楼，假设商家从一楼搬到二楼，导览图也会更新，你会直接走到二楼

## 背景

以下一些操作或作业是任意一个分布式系统都必须的：

**分布式系统是一个硬件或软件组件分布在不同的网络计算机中，彼此之间仅仅通过消息传递进行通信和协调的系统**

- leader选举：对于leader/follower或master/slave结构的分布式系统而言，leader选举就是必须的。特别是现有leader崩溃后如何从剩余节点中选择新的leader

- KV元数据存储：分布式系统中有一些元数据信息要保存在所有的节点上以维持系统的一致性

- 成员管理：分布式系统中节点的新增、移除都需要依托单独的组件来管理

Zookeeper把这些功能独立地实现在一个单独的协调服务软件中，ZooKeeper 最常用的使用场景就是用于担任服务生产者和服务消费者的注册中心，ZooKeeper底层提供两个主要功能，一是管理（存储、读取）用户程序提交的数据；二是为用户程序提交数据节点监听服务。

## 简介

ZooKeeper 是个针对大型分布式系统的高可用、高性能且具有一致性的开源协调服务，专业于任务协作

分布式系统中统一配置、协调资源、统一命名服务，保证分布式一致性
https://en.wikipedia.org/wiki/Apache_ZooKeeper

顺序一致性:同一个客户端发出的事务请求，最终会严格按照其发起顺序被应用到ZooKeeper中去，这个编号也叫做时间戳—zxid（ZooKeeper Transaction Id）

原子性:同数据库原子性

单一视图:无论客户端连接的是哪个ZK服务器，看到服务端的数据是一致的

可靠性:类似数据库的持久性

实时性:ZK仅仅保证一定时间段内（“写”会导致所有的服务器间同步），客户端最终一定能够从服务端上读取到最新的数据状态

### 节点

数据结构为树型,无论客户端连接的是哪个ZK服务器，看到服务端的数据是一致的，引用方式为路径引用，类似 /水果/苹果 这样的路径

ZNode存储在内存中，ZNode中包含了数据、子节点引用（左右子节点），由于存储在内存中ZooKeeper可以实现高吞吐量和低延时

ACL(AccessControlLists)策略来进行权限控制，类似UNIX的文件权限控制，ZK定义了5中权限，CREATE：创建子节点权限，READ：读取数据节点和子节点的权限，WRITE：更新数据节点的权，DELETE：删除子节点的权限，ADMIN：设置节点ACL的权限

状态Stat（如事务ID、版本号、时间戳、大小）其中版本中包括ZNode的版本，还包括ZNode子节点的版本以及ZNode ACL的版本

每个ZNode 只是用来存储少量状态和配置信息，所以ZNode最大不超过1MB

ZNode分为两种类型：

临时节点（Ephemeral）与会话生命周期绑定，当客户端和服务端断开连接后（会话失效），ZNode节点会自动删除，此类型的目录节点不能有子节点目录，EPHEMERAL_SEQUENTIAL临时有序

持久化节点（Persistent）当客户端和服务端断开连接后，ZNode节点不会自动删除，要想删除只能主动delete，主要目的是为应用保存数据，PERSISTENT_SEQUENTIAL 持久有序

SEQUENTIAL  加上这个后缀节点叫做顺序节点，一旦节点被标记上这个属性，那么这个节点被创建时自动在其后面追加一个整型数字，这个整型数字由父节点维护的自增数字



### 会话

Session指的是ZooKeeper服务端与客户端会话，客户端连接是指客户端和服务端之间一个TCP长连接，可以向服务端发送请求并接受响应，同时还能够通过该连接接收来自服务器的Watch事件通知

SessionID:用来全局唯一识别会话
SessionTimeOut:当由于服务器压力太大、网络故障、或是客户端主动断开连接等原因导致客户端连接断开时，只要在SessionTimeOut规定时间内重新连接上集群中任意一台服务器，那么之前创建的会话任然有效
TickTime:下次会话超时时间点
isClosing:当服务端如果检测到会话超时失效，通过设置这个舒心将会话关闭

### ZooKeeper如何保证主从节点状态同步——原子广播机制

实现这个机制为Zab（ZooKeeper Atomatic Broadcast）协议，作为ZooKeeper的核心实现算法Zab，就是解决了分布式系统下数据如何在多个服务之间保持同步问题的，即解决集群崩溃恢复（恢复模式）和以及数据同步（广播模式）问题

ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services。

### 选举机制

Zookeeper的客户端和服务器通信采用长链接方式，每个客户端和服务器通过心跳来保持连接

更新数据时，先更新Leader节点再更新至Follower节点

读取数据时，直接读取随机Follower节点

引入Observer的一个主要原因是提高读请求的可扩展性，不同于Follower的是，观察者不参与选举过程，每一个新加入的观察者将对应于每一个已提交事务点引入的一条额外消息

另一个原因是进行跨多个数据中心部署。由于数据中心之间的网络链接延时，将服务器分散于多个数据中心将明显地降低系统的速度

Leader选举 (Leader Election)

#### 第一次投票：

所有机器都处于Looking状态，投票中包含SID（服务器唯一标示）和ZXID（事务ID），用（SID，ZXID）来标示一此投票

举个例子，有1、2、3、4、5 SID机器，其中2为Leader，但是1 ，2号机器崩溃，此时3，4，5号机器开始第一次投票，每台机器都会将自己作为投票对象，投票内容为（3，9）、（4，8）、（5，8）

#### 变更投票:

每台机器发出投票后，也会收到其他机器的投票，会按照一定算法规则确定是否需要变更自己的投票

vote_sid：接收到的投票中所推举Leader服务器的SID。

vote_zxid：接收到的投票中所推举Leader服务器的ZXID。

self_sid：当前服务器自己的SID。

self_zxid：当前服务器自己的ZXID。

变更算法如下

规则一：如果vote_zxid大于self_zxid，就认可当前收到的投票，并再次将该投票发送出去。

规则二：如果vote_zxid小于self_zxid，那么坚持自己的投票，不做任何变更。

规则三：如果vote_zxid等于self_zxid，那么就对比两者的SID，如果vote_sid大于self_sid，那么就认可当前收到的投票，并再次将该投票发送出去。

规则四：如果vote_zxid等于self_zxid，并且vote_sid小于self_sid，那么坚持自己的投票，不做任何变更。

经过第二轮投票后，统计机器收到的投票，如果一台机器收到超过半数相同的投票，那么投票对应的SID机器被选举为Leader

只有最新的服务器将赢得选举，因为其拥有最近一次的 zxid,如果多个服务器拥有的最新的 zxid 值，其中的 sid 值最大的将会赢得选举。

一般说来Zookeeper leader的选择过程都非常快，通常<200ms

注意：使用的服务器最好为奇数台

### 发现阶段(Discovery)

用于在从节点中发现最新的ZXID和事务日志，

这一阶段，Leader接收所有Follower发来各自的最新epoch值。Leader从中选出最大的epoch，基于此值加1，生成新的epoch分发给各个Follower。

各个Follower收到全新的epoch后，返回ACK给Leader，带上各自最大的ZXID和历史事务日志。Leader选出最大的ZXID，并更新自身历史日志。

同步阶段(synchronization)

把Leader刚才收集得到的最新历史事务日志，同步给集群中所有的Follower。只有当半数Follower同步成功，这个准Leader才能成为正式的Leader。

### 数据广播

如何确认一个事务是否已经提交，ZooKeeper 由此引入了 zab 协议，两提交阶段

Leader采用两阶段提交，首先发送Propose给Follower

Follower收到Propose消息，写到日志后，返回ACK，通知群首其已接受该提案

Leader接收到半数以上的ACK后，返回成功给客户端，并返回Commit请求给Follower

在接收到一个写请求操作后，追随者会将请求转发给群首，群首将会探索性的执行该请求，并将执行结果以事务的方式对状态更新进行广播

Leader生成一个新的事务并为这个事务生成一个唯一的ZXID

Leader采用两阶段提交，发送该事务Propose给所有的Follower节点

Follower节点将收到的事务请求加入到历史队列中和日志中,并发送ack给Leader  ，通知Leader其已接收提案

当Leader收到超过一半数量Follower的ACK消息后，会发送commit请求 

当Follower收到commit请求时，会判断该事务的ZXID是不是比历史队列中的任何事务的ZXID都小，如果是则提交，如果不是则等待比它更小的事务的commit

### ZooKeeper 都有哪些功能——应用场景

#### 统一命名服务

分布式系统中客户端使用命名服务，能够制定名字获取资源或服务地址，提供者信息，这些信息统称为名字，其中较常见的为RPC服务地址列表

通过ZooKeeper提供的创建节点API，可创建一个全局唯一path，这个path就可以作为一个名字

#### 数据发布与订阅（配置中心）

发布者将数据发布到ZooKeeper节点，供订阅者动态获取数据，实现配置信息的集中式管理和动态更新

分布式环境下配置文件管理和同步问题，一个集群中，所有节点的配置信息是一致的，可将配置信息写入ZooKeeper上一个ZNode，各个客户端监听这个ZNode,一旦 ZNode中的数据被修改，ZooKeeper将通知各个节点

#### 统一集群管理

分布式环境中，实时掌握每个机器的状态，可将每个机器的状态信息写入ZooKeeper上一个ZNode,监听这个ZNode可获取它的实时状态变化

#### 软负载均衡

注册服务下每一个ZNode存储IP地址，客户端机器监听注册服务

#### 分布式协调/通知

ZooKeeper中Watch注册和异步通知机制，能够很好实现分布式环境下不同机器，甚至不同系统之间的协调和通知，从而实现对数据变更的实时处理

心跳检测中让检测系统和被检测系统之间并不直接关联起来，而是通过ZK上某个节点关联，减少系统耦合，同理系统调度系统

客户端获得服务端数据变化，不是通过轮询，而是通过通知机制，客户端向 ZooKeeper 服务器端注册需要接收通知的 ZNode，通过对 ZNode 设置监视点来接收通知

ZooKeeper的Watch特性

- Watch是一次性的，每次都需要重新注册，并且客户端在会话异常结束时不会收到任何通知，而快速重连接时仍不影响接收通知。

客户端调用 getData("/znode", true) 并且/znode1 节点上的数据发生了改变或者被删除了，客户端将会获取/znode1 发生变化的监视时间

- Watch的回调执行都是顺序执行的，并且客户端在没有收到关注数据的变化事件通知之前是不会看到最新的数据，另外需要注意不要在Watch回调逻辑中阻塞整个客户端的Watch回调

- Watch是轻量级的，WatchEvent是最小的通信单元，结构上只包含通知状态、事件类型和节点路径。ZooKeeper服务端只会通知客户端发生了什么，并不会告诉具体内容。
- 注册Watcher 其中getData、exists为数据监视，getChildren为子数据监视，触发Watcher create、delete、setData 。
  
  setData()会触发znode上设置的data watch（如果set成功的话）。一个成功的create() 操作会触发被创建的znode上的数据watch，以及其父节点上的child watch。而一个成功的delete()操作将会同时触发一个znode的data watch和child watch（因为这样就没有子节点了），同时也会触发其父节点的child watch

*注意*：3.6.0新增功能：客户端可以在ZNode上设置永久性的递归监视，这些监视在触发时不会删除，并且会以递归方式触发已注册ZNode以及所有子ZNode的修改

ZooKeeper所有读操作getData 、getChildren 、exists 都可以设置监视

#### 分布式锁

锁服务可以分为两类：

保持独占：只有一个可以成功获取到这把锁，把ZK上一个ZNode看作一把锁，通过 create znode的方式来实现，所有客户端都去创建/distribute_lock 节点，最终成功创建的那个客户端即拥有了这把锁

控制时序：全局时序，客户端在创建临时有序节点维持一份Sequence，保证子节点创建的时序性，从而也形成了每个客户端的全局时序。

疑问：

1、ZXID如何确定？ 

zxid 是一个 64 位的数字，它高 32 位是 epoch 用来标识 leader 关系是否改变，每次一个 leader 被选出来，它都会有一个新的 epoch，低 32 位是个递增计数。 zxid=epoch+counter

2、Leader有哪些方式失去Follower支持

3、ZooKeeper部署方式？

单机部署：一台集群上运行

集群部署：多台集群运行

伪集群部署：一台集群启动多个 Zookeeper 实例运行

4、通信方式

Client与Follower通信是通过NIO

系统默认提供了3种选择算法，AuthFastLeaderElection，FastLeaderElection，LeaderElection。其中AuthFastLeaderElection和LeaderElection采用UDP模式进行通信，而FastLeaderElection仍然采用TCP/IP
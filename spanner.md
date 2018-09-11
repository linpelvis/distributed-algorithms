Spanner

#### 介绍

* **数据分片**存储在许多Paxos状态机上，这些机器遍布全球的数据中心。

* **复制技术**服务于全球可用性和地理局部性，保障了高可用性。

* 客户端会自动在副本之间进行失败恢复。

* 随着数据的变化和服务器的变化，Spanner会自动把数据进行**重新分片**。

* Spanner的主要工作，管理跨越多个数据中心的数据副本。

* 特性，使得Spanner可以支持一致的备份、一致的MapReduce执行和原子模式的变更

  * **数据距离用户**有多远（控制用户读取数据的延迟）

  * 不同**数据副本之间**距离有多远（控制写操作的延迟）

  * 需要**维护多少个副本**（控制可用性和读操作性能）

  * 重要特性：**读与写操作的外部一致性**，**在一个时间戳下面的跨数据库的全球一致性的读操作**

* 这些特性关键技术是**TrueTime API\(借助现代时钟参考值：原子钟、GPS\)**，TrueTime API可以直接暴露时钟不确定性，Spanner时间戳保证API实现的界限。如果不确定性很大，Spanner就降低速度来等待这个不确定性结束。

#### 实现

* Spanner server org包括：universe master、placement driver、zone1\[zone2, \[zone3\]\]。**见论文图1**

* 在一个Spanner的universe中的服务器，zone包括一个zonemaster和上千个spanserver。zonemaster把数据分配给spanserver，spanserver把数据提供给客户端。客户端使用每个zone上面的location proxy来定位可以为自己提供数据的spanserver。universe master是控制台，显示关于zone的各种状态信息。placement driver会周期性地与spanserver进行交互，发现那些数据被转移\满足新副本约束条件\负载均衡

##### 软件栈

* **见论文图2**，每个spanserver负载管理100-1000个称为tablet的数据结构的实例。一个tablet类似于BigTable中的tablet，实现的映射：（key:string, timestamp:int64）-&gt; string。与BigTable不同的是，Spanner会把时间戳分配给数据，使得Spanner更像一个多版本数据库，而不是一个KV存储。一个tablet的状态是存储在类似于B-树的文件集合和write-ahead的日志中，所有这些会被保存在分布式文件系统中Colossus，它继承于Google File System。

* 支持复制，每个spanserver会在每个tablet上面实现一个单个的Paxos状态机。它的Paxos实现支持采用基于时间的leader租约的长寿命的领导者，时间范围\[0, 10\]。

* paxos状态机是被实现**一系列被一致性复制**的映射。每个副本的键值映射状态，都会被保存到相应的tablet中。**写操作**必须在领导者上初始化paxos协议，**读操作**可以直接从底层的任何副本的tablet中访问状态信息。

* 对于是primary的replica而言，每个spanserver会**实现一个锁表来实现并发控制**。这个锁表包含了**两阶段**锁机制的状态，把键值域映射到锁状态上面。

* 对于是primary的replica而言，每个spanserver会**实施一个事务管理器来支持分布式事务**，这个事务管理器被用来实现**participant leader**，其他副本则是作为**participant slaves**。

##### 目录和放置

* **见论文图3**，“目录”-&gt;桶抽象，就是包含公共前缀的连续键的集合。一个目录是数据放置的基本单元。属于一个目录的所有数据，都具有相同的副本配置。当数据在不同的paxos组之间进行移动时，会一个目录一个目录地转移。

* 一个paxos组可以包含多个目录，这意味着一个spanner tablet是不同于一个BigTable tablet的。一个spanner tablet可以是行空间内的多个分区。这样可以让多个频繁一起访问的目录被整合到一起。

* Movedir是一个后台任务，用于在不同的paxos组之间转移目录。它也可以用paxos组增加和删除副本。

* 当一个目录变大时，spanner会把它分片存储，每个分片可能会被保存到不同的paxos组上（意味着来自不容的服务器）。movedir在不同组转移的分片。

##### 数据模型

* **见论文图4**，数据模型不是纯粹关系型，行必须有名称。每个表都需要由包含一个或者多个主键列的排序集合。这种结构可以让应用通过选择键来控制数据的局部性。

#### TureTime

* TureTime的实现方法

  ```
  TT.now() 
    return TTinterval:[earliest, latest]
  TT.afert(t)
    return true if has definitely passed
  TT.before(t)
    return true if t has definitely not arrived
  ```

* TureTime使用的时间是GPS和原子钟，GPS弱点是天线和接收器失效、局部电磁干扰等。原子钟存储频率误差。

* **大多数master都有GPS接受器**，这些master在物理上是相互隔离（减少天线失效、电磁干扰和电子欺骗）。**剩余的master（Armageddon master）则配备原子钟**。**所有master**的时间参考值都会进行彼此校对。每个master也会交叉检查时间参考值和本地时间的比值。在同步期间，**Armageddon master**会表现出一个逐渐**增加的时间不确定性**。GPS master表现出的时间不确定性几乎接近于0。

* 每个daemon会从许多master中收集投票，获得时间参考值，从而减少误差。**被选中的master中，有些master是GPS master，是从附近的数据中心获取的，剩余的GPS master是从远处的数据中心获得的**

* 在同步期间，一个daemon会表现出逐渐增加的时间不确定性。ε是取决于timer master的不确定性+time master之间的通讯延迟。在每个投票间隔中，ε范围\[1ms, 7ms\]。因此在大多数的情况下不确定是0到6ms的边界，通讯延迟是1ms。

#### 并发控制

* TrueTime可以用来保证并发控制的正确性，并可实现一些关键特性，比如外部一致性的事务、无锁机制的只读事务、针对历史数据的非阻塞读。

##### 时间戳管理

* Spanner支持读写事务、只读事务、快照读。

* 一个只读事务具备快照隔离的性能优势。它必须声明不会包含任何写操作。在进行只读事务的读操作时候，会采用一个系统选择的时间戳。不包含锁机制。因此写操作不会被阻塞。

* 一个快照读操作，是针对历史数据的读取，不需要锁机制，它可以是客户端提供一个时间戳，或者提供一个世界范围让spanner来自动选择时间戳。

* 当一个服务器失效时候，客户端可使用同一的时间戳和当前的读位置，在另外一个服务器上继续执行读操作。

##### paxos领导者租约

* spanner的paxos使用了时间化的租约，来实现长时间的leader地位（默认是10秒）。可以理解为候选节点发起选主请求，收到足够多的投票后，这个阶段确定自己拥有一个租约。

* 一个副本在成功完成了一个写操作后，会隐式地延期自己的租约。

* 一个leader而言，如果它的租约快到期了，需要显示的请求租约延期。投票有一个时间区间，时间区间内需达指定的数量。这表明投票是有时间有效期的。

* spanner对于每个paxos组，每个paxos领导者的租约时间区间，是和其他领导者的时间区间完全隔离的。

##### 为读写事务分配时间戳

* 事务读和写采用两端锁协议

* spanner依赖单调性：spanner会以**单调增加**的顺序给每个paxos写操作分配时间戳。

* spanner外部一致性：如果一个事务T2在事务T1提交以后开始执行，那么事务T2的时间戳一定比事务T1的时间戳大。

* **Start**：T2的时间戳需大于在T1 commit时候的TT.now\(\).latest

* **Commit Wait**：TT.after是确保客户端不能看到任何被Ti提交的数据，直到TT.after为true。

##### 在某个时间戳下的读操作

* 根据单调性，spanner可以正确地确定一个副本是否足够新，从而能够满足一个读操作的请求。每个副本都会跟踪记录一个值，这个值被称为安全事件tsaft，它是副本最近更新后的最大时间戳。如果读操作的时间戳是t，当满足t&lt;=tsaft时，这个副本就可以被这个读操作读取。

* tsaft=min\(tPaxossaft， tTMsaft\)，tTMsaft是引用副本的领导者的事务管理器。tPaxossaft是paxos**写操作**的时间戳。在小于tPaxossaft以后，写操作就不会发生。

##### 为只读事务分配时间戳

* 一个只读事务分为两个阶段执行：分配一个时间戳Sread，然后当成Sread时刻的快照读来执行事务读操作。Sread=TT.now\(\).lastest。

#### 事务实现的细节

* 待后整理。




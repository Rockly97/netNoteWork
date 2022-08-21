# Redis 数据类型

## 五种基础数据

### String字符串

String类型是二进制安全的，意思是 redis 的 string 可以包含任何数据。如数字，字符串，jpg图片或者序列化的对象

### List列表

使用List结构，我们可以轻松地实现最新消息排队功能（比如新浪微博的TimeLine）。List的另一个应用就是消息队列，可以利用List的 PUSH 操作，将任务存放在List中，然后工作线程再用 POP 操作将任务取出进行执行。

### Set集合

Redis 的 Set 是 String 类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据。

### Hash散列

Redis hash 是一个 string 类型的 field（字段） 和 value（值） 的映射表，hash 特别适合用于存储对象。

### Zset有序集合

Redis 有序集合和集合一样也是 string 类型元素的集合,且不允许重复的成员。不同的是每个元素都会关联一个 double 类型的分数。redis 正是通过分数来为集合中的成员进行从小到大的排序。

## 三种特殊类型

### HyperLogLogs（基数统计）

**基数**

A = {1, 2, 3, 4, 5}， B = {3, 5, 6, 7, 9}；那么基数（不重复的元素）= 1, 2, 4, 6, 7, 9； （允许容错，即可以接受一定误差）

#### 应用场景

这个结构可以非常省内存的去统计各种计数，比如注册 IP 数、每日访问 IP 数、页面实时UV、在线用户数，共同好友数等。

**它一个基于基数估算的算法，只能比较准确的估算出基数（并不一定准确，是一个带有 0.81% 标准错误的近似值）**

### Bitmap （位存储）

Bitmap 即位图数据结构，都是操作二进制位来进行记录，只有0 和 1 两个状态。

#### 应用场景

统计用户信息，活跃，不活跃！ 登录，未登录！ 打卡，不打卡！ **两个状态的，都可以使用 Bitmaps**

### geospatial (地理位置)

出现在 Redis 3.2 版本里，可以推算地理位置的信息: 两地之间的距离, 方圆几里的人

#### 应用场景

geoadd 添加地理位置

# Redis 事件机制

Redis 采用事件驱动机制来处理大量的网络IO，采用的Redis自己实现的 ae_event

## 事件机制

**文件事件**(file  event)：用于处理 Redis 服务器和客户端之间的网络IO。

**时间事件**(time  eveat)：Redis 服务器中的一些操作（比如serverCron函数）需要在给定的时间点执行，而时间事件就是处理这类定时操作的。

事件驱动库的代码主要是在`src/ae.c`中实现的，其示意图如下图

![img](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220026186.png)

`aeEventLoop`是整个事件驱动的核心，它管理着文件事件表和时间事件列表，不断地循环处理着就绪的文件事件和到期的时间事件。

### 文件事件

Redis基于**Reactor模式**开发了自己的网络事件处理器，文件事件处理器使用**IO多路复用技术**，同时监听多个套接字，并为套接字关联不同的事件处理函数。当套接字的可读或者可写事件触发时，就会调用相应的事件处理函数。

**1. 为什么单线程的 Redis 能那么快**？

Redis的瓶颈主要在IO而不是CPU，所以为了省开发量，在6.0版本前是单线程模型；其次，Redis 是单线程主要是指 **Redis 的网络 IO 和键值对读写是由一个线程来完成的**

Redis 的其他功能，比如持久化、异步删除、集群数据同步等，其实是由额外的线程执行的

**2. Redis事件响应框架ae_event及文件事件处理器**

Redis 使用的IO多路复用技术主要有：`select`、`epoll`、`evport`和`kqueue`等。每个IO多路复用函数库在 Redis 源码中都对应一个单独的文件，比如`ae_select.c`，`ae_epoll.c`， `ae_kqueue.c`等。Redis 会根据不同的操作系统，按照不同的优先级选择多路复用技术。事件响应框架一般都采用该架构，比如 netty 和 libevent。![img](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220031651.png)

文件事件处理器有四个组成部分，它们分别是套接字、I/O多路复用程序、文件事件分派器以及事件处理器![img](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220031258.png)

一次 Redis 客户端与服务器进行连接并且发送命令的过程如下图所示：![img](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220032302.png)

- 客户端向服务端发起**建立 socket 连接的请求**，那么监听套接字将产生 AE_READABLE 事件，触发连接应答处理器执行。处理器会对客户端的连接请求
- 进行**应答**，然后创建客户端套接字，以及客户端状态，并将客户端套接字的 AE_READABLE 事件与命令请求处理器关联。
- 客户端建立连接后，向服务器**发送命令**，那么客户端套接字将产生 AE_READABLE 事件，触发命令请求处理器执行，处理器读取客户端命令，然后传递给相关程序去执行。
- **执行命令获得相应的命令回复**，为了将命令回复传递给客户端，服务器将客户端套接字的 AE_WRITEABLE 事件与命令回复处理器关联。当客户端试图读取命令回复时，客户端套接字产生 AE_WRITEABLE 事件，触发命令回复处理器将命令回复全部写入到套接字中。



**3. Redis IO多路复用模型**

在 Redis 只运行单线程的情况下，**该机制允许内核中，同时存在多个监听套接字和已连接套接字**。内核会一直监听这些套接字上的连接请求或数据请求。一旦有请求到达，就会交给 Redis 线程处理，这就实现了一个 Redis 线程处理多个 IO 流的效果。

![img](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220035528.jpeg)

以连接请求和读数据请求为例，这两个请求分别对应 Accept 事件和 Read 事件，Redis 分别对这两个事件注册 accept 和 get 回调函数。当 Linux 内核监听到有连接请求或读数据请求时，就会触发 Accept 事件和 Read 事件，此时，内核就会回调 Redis 相应的 accept 和 get 函数进行处理。

### 时间事件

- **定时事件**：让一段程序在指定的时间之后执行一次。
- **周期性事件**：让一段程序每隔指定时间就执行一次。

服务器所有的时间事件都放在一个无序链表中，每当时间事件执行器运行时，它就遍历整个链表，查找所有已到达的时间事件，并调用相应的事件处理器。正常模式下的Redis服务器只使用serverCron一个时间事件，而在benchmark模式下，服务器也只使用两个时间事件，所以不影响事件执行的性能。

# Redis事务

Redis 事务的本质是一组命令的集合。事务支持一次执行多个命令，一个事务中所有命令都会被序列化。在事务执行过程，会按照顺序串行化执行队列中的命令，其他客户端提交的命令请求不会插入到事务执行命令序列中。





## Redis事务相关命令和使用

> MULTI 、 EXEC 、 DISCARD 和 WATCH 是 Redis 事务相关的命令。

- MULTI ：开启事务，redis会将后续的命令逐个放入队列中，然后使用EXEC命令来原子化执行这个命令系列。
- EXEC：执行事务中的所有操作命令。
- DISCARD：取消事务，放弃执行事务块中的所有命令。
- WATCH：监视一个或多个key,如果事务在执行前，这个key(或多个key)被其他命令修改，则事务被中断，不会执行事务中的任何命令。
- UNWATCH：取消WATCH对所有key的监视。

[TOC]



# Redis 持久化

Reids 是基于内存中执行的，服务器宕机，内存中的数据将不会保存。解决这一问题是进行将数据保存下来，而从后端数据库中进行恢复会存在性能瓶颈，所以产生了两种持久化的方式`AOF日志`和` RDB（快照）`

### AOF持久化

AOF是记录的Redis执行的每个命令，而记录的步骤为 `命令追加(append)` `文件写入(write)` `文件同步(sync)`

AOF 文件写入与同步具有三种策略

![redis-x-aof-4](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/redis-x-aof-4.jpg)

#### AOF重写机制

当AOF的文件不停的进行记录命令，AOF文件会逐渐变大，AOF恢复需要一个一个的执行命令，文件太大会影响性能以及效率变低，会导致数据恢复变慢，Redis提出了AOF重写机制来进行对AOF文件重写。

AOF重写是进行创建一个新的AOF文件来替换现有的AOF，两个文件的数据是保存的相同，进行多个重复命令进行整合，对于新的文件没有了冗余的命令，从而减轻文件的大小。

![redis-x-aof-1](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/redis-x-aof-1.jpg)

AOF重写过程 `一处拷贝，两处日志`

![redis-x-aof-2](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/redis-x-aof-2.jpg)



不在主线程中进行的，但在fork子进程时会有阻塞产生，其原因是在进行fork时会拷贝父进程的页表，会消耗掉大量的`CPU资源`

#### redis.conf配置

在Reids中配置AOF是在文件redis.conf中配置，AOF的配置如下：
```
# appendonly参数开启AOF持久化
appendonly no

# AOF持久化的文件名，默认是appendonly.aof
appendfilename "appendonly.aof"

# AOF文件的保存位置和RDB文件的位置相同，都是通过dir参数设置的
dir ./

# 同步策略
# appendfsync always
appendfsync everysec
# appendfsync no

# aof重写期间是否同步
no-appendfsync-on-rewrite no

# 重写触发配置
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 加载aof出错如何处理
aof-load-truncated yes

# 文件重写策略
aof-rewrite-incremental-fsync yes
```

### RDB持久化
RDB持久化是把当前进程数据生成快照保存到磁盘上的过程，由于是某一时刻的快照，那么快照中的值要早于或者等于内存中的值。

两种触发方式：`手动`    `自动` 

 **手动触发：**

 执行`save` 和 `bgsave` 命令

- **save命令**：阻塞当前Redis服务器，直到RDB过程完成为止，对于内存 比较大的实例会造成长时间**阻塞**，线上环境不建议使用

- **bgsave命令**：Redis进程执行fork操作创建子进程，RDB持久化过程由子 进程负责，完成后自动结束。阻塞只发生在fork阶段，一般时间很短

  bgsave 是创建了子进程 专门写RDB文件避免了主线程的阻塞，这也是生成RDB的默认配置，其中的核心思想：`copy-on-writer `

  

bgsave流程图如下

![redis-x-rdb-1](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/redis-x-rdb-1.png)

在遇见大量的数据操作时，同步时间过长，在这中间发生数据读写操作，保证数据一致性，而实现一致性如下图所示

![redis-x-aof-42](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/redis-x-aof-42.jpg)



RDB的频繁地执行全量快照，会带来两方面的开销

- 频繁将全量数据写入磁盘，会给磁盘带来很大压力

- bgsave 子进程需要通过 fork 操作从主线程创建出来，fork 这个创建过程本身会阻塞主线程，而且主线程的内存越大，阻塞时间越长。频繁 fork 出 bgsave 子进程，这就会频繁**阻塞主线程**了。

  

**自动触发：**

在4种情况下进行触发

- redis.conf中配置`save m n`，即在m秒内有n次修改时，自动触发bgsave生成rdb文件；
- 主从复制时，从节点要从主节点进行全量复制时也会触发bgsave操作，生成当时的快照发送到从节点；
- 执行debug reload命令重新加载redis时也会触发bgsave操作；
- 默认情况下执行shutdown命令时，如果没有开启aof持久化，那么也会触发bgsave操作



#### 在redis.conf中进行配置RDB

```
# 周期性执行条件的设置格式为
save <seconds> <changes>

# 默认的设置为：
save 900 1
save 300 10
save 60 10000

# 以下设置方式为关闭RDB快照功能
save ""
```



### 持久化恢复数据
先进行查看AOF文件，有执行AOF命令，AOF不存在加载RDB文件

AOF的数据保存最完整

# Redis 主从复制

> 主从复制，是指将一台Redis服务器的数据，复制到其他的Redis服务器。前者称为主节点(master)，后者称为从节点(slave)；数据的复制是单向的，只能由主节点到从节点。

**主要作用：**

**数据冗余**：主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式。

**故障恢复**：当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复；实际上是一种服务的冗余。

**负载均衡**：在主从复制的基础上，配合读写分离，可以由主节点提供写服务，由从节点提供读服务（即写Redis数据时应用连接主节点，读Redis数据时应用连接从节点），分担服务器负载；尤其是在写少读多的场景下，通过多个从节点分担读负载，可以大大提高Redis服务器的并发量。

**高可用基石**：除了上述作用以外，主从复制还是哨兵和集群能够实施的基础，因此说主从复制是Redis高可用的基础。

采用`读写分离`模式

- 读操作：主库、从库都可以接收；
- 写操作：首先到主库执行，然后，主库将写操作同步给从库。

###    

### 主从复制原理

- `全量（同步）复制`：比如第一次同步时
- `增量（同步）复制`：只会把主从库网络断连期间主库收到的命令，同步给从库



#### 全量复制

存在多个Redis实例时，互相之间通过replicaof命令形成主从关系，按照三个阶段进行第一次同步

-  确认关系

```bash
replicaof 172.16.19.3 6379
```

- 三个阶段

![db-redis-copy-2](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/db-redis-copy-2.jpg)



1. 建主从库间建立连接、协商同步的过程

   从库给主库发送psync命令，表示要进行数据同步，主库根据这个命令的参数来启动复制。psync命令包含了主库的runID和复制进度offset两个参数

2. 主库同步数据给从库

   主库收到psync命令后，会用FULLRESYNC响应命令带上两个参数：主库runID和主库目前的复制进度offset，返回给从库。从库收到响应后，会记录下这两个参数

3. 主库会把第二阶段执行过程中新收到的写命令，再发送给从库

   当主库完成 RDB 文件发送后，就会把此时 replication buffer 中的修改操作发给从库，从库再重新执行这些操作。这样一来，主从库就实现同步了

   

#### 增量复制

![db-redis-copy-3](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/db-redis-copy-3.jpg)

# Redis 哨兵模式

其主要功能是可以进行故障转移，可以实现主库从库的自动切换机制
哨兵逻辑图：

![db-redis-sen-1](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/db-redis-sen-1.png)

哨兵对应的功能：
**监控（Monitoring）**：监控主节点与从节点是否正常。

**自动故障转移（Automatic failover）**：当主节点的故障时，哨兵会主动进行将其中的失效的主节点的从节点升级为从节点。

**配置服务提供者（Configuration provider）**：在客户端进行初始化时，将连接哨兵的获得当前Redis服务主节点

**通知（Notification）**：哨兵可以将故障转移的结果通知给客户端

 

### 哨兵集群的建立

哨兵之间的发现，需要Redis提供的pub/sub机制。

在主从集群中，主库上有一个名为`__sentinel__:hello`的频道，不同哨兵就是通过它来相互发现，实现互相通信的。在下图中，哨兵 1 把自己的 IP（172.16.19.3）和端口（26579）发布到`__sentinel__:hello`频道上，哨兵 2 和 3 订阅了该频道。那么此时，哨兵 2 和 3 就可以从这个频道直接获取哨兵 1 的 IP 地址和端口号。然后，哨兵 2、3 可以和哨兵 1 建立网络连接。

![db-redis-sen-6](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/db-redis-sen-6.jpg)

 ### 哨兵监控

对每个主从节点进行监控，访问主节点主节点将从节点的列表返回给哨兵节点，哨兵节点根据列表对从库进行持续监控。

![db-redis-sen-7](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/db-redis-sen-7.jpg)

### 下线判断

下线判断具有两种概念，`主观下线`和`客观下线`

**主观**：任何一个哨兵都可以监控探测，对Reids节点进行下线判断。

**客观**：哨兵集群进行共同决定Redis客观下线

单个哨兵进行判断”主观下线”后，就会给其他哨兵进行发送`is-master-down-by-addr`命令

根据主库连接情况进行做出反应

![db-redis-sen-2](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/db-redis-sen-2.jpg)



### 集群选举

选举机制采用的是Raft选举算法：**选举的票数大于等于num(sentinels)/2+1时，将成为领导者，如果没有超过，继续选举**

但是哨兵不能完成主从切换

#### 主库选举

- 过滤掉不健康的（下线或断线），没有回复过哨兵ping响应的从节点

- 选择`salve-priority`从节点优先级最高（redis.conf）的

- 选择复制偏移量最大，只复制最完整的从节点

  ![db-redis-sen-3](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/db-redis-sen-3.jpg)

  

# Redis Cluster （分片技术）

在Reids3.0版本中推出这个技术， 主要解决读写分离时虽然slave节点扩展了主从的读并发能力，但是**写能力**和**存储能力**是无法进行扩展，就只能是master节点能够承载的上限。

## 主要模块

### 哈希槽(Hash Slot)

Redis-cluster没有使用一致性hash，而是引入了**哈希槽**的概念。Redis-cluster中有16384(即2的14次方）个哈希槽，每个key通过CRC16校验后对16383取模来决定放置哪个槽。Cluster中的每个节点负责一部分hash槽（hash slot）。

比如集群中存在三个节点，则可能存在的一种分配如下：

- 节点A包含0到5500号哈希槽；
- 节点B包含5501到11000号哈希槽；
- 节点C包含11001 到 16384号哈希槽。

### Keys hash tags

Hash tags提供了一种途径，**用来将多个(相关的)key分配到相同的hash slot中**。这时Redis Cluster中实现multi-key操作的基础。

hash tag规则如下，如果满足如下规则，{和}之间的字符将用来计算HASH_SLOT，以保证这样的key保存在同一个slot中。

- key包含一个{字符
- 并且 如果在这个{的右面有一个}字符
- 并且 如果在{和}之间存在至少一个字符

例如：

- {user1000}.following和{user1000}.followers这两个key会被hash到相同的hash slot中，因为只有user1000会被用来计算hash slot值。
- foo{}{bar}这个key不会启用hash tag因为第一个{和}之间没有字符。
- foozap这个key中的{bar部分会被用来计算hash slot
- foo{bar}{zap}这个key中的bar会被用来计算计算hash slot，而zap不会

### Cluster nodes属性

每个**节点在cluster中有一个唯一的名字**。这个名字由160bit随机十六进制数字表示，并在节点启动时第一次获得(通常通过/dev/urandom)。节点在配置文件中保留它的ID，并永远地使用这个ID，直到被管理员使用CLUSTER RESET HARD命令hard reset这个节点。

节点ID被用来在整个cluster中标识每个节点。一个节点可以修改自己的IP地址而不需要修改自己的ID。Cluster可以检测到IP /port的改动并通过运行在cluster bus上的gossip协议重新配置该节点。

节点ID不是唯一与节点绑定的信息，但是他是唯一的一个总是保持全局一致的字段。每个节点都拥有一系列相关的信息。一些信息时关于本节点在集群中配置细节，并最终在cluster内部保持一致的。而其他信息，比如节点最后被ping的时间，是节点的本地信息。

每个节点维护着集群内其他节点的以下信息：`node id`, `节点的IP和port`，`节点标签`，`master node id`（如果这是一个slave节点），`最后被挂起的ping的发送时间`(如果没有挂起的ping则为0)，`最后一次收到pong的时间`，`当前的节点configuration epoch` ，`链接状态`，以及最后是该节点服务的`hash slots`。

```shell
$ redis-cli cluster nodes 
d1861060fe6a534d42d8a19aeb36600e18785e04 127.0.0.1:6379 myself - 0 1318428930 1 connected 0-1364 
3886e65cc906bfd9b1f7e7bde468726a052d1dae 127.0.0.1:6380 master - 1318428930 1318428931 2 connected 1365-2729
```

其对应上面结果含义为：

```sh
node id, address:port, flags, last ping sent, last pong received, configuration epoch, link state, slots.
```



### Cluster总线

每个Redis Cluster节点有一个额外的TCP端口用来接受其他节点的连接。这个端口与用来接收client命令的普通TCP端口有一个固定的offset。该端口等于普通命令端口加上10000.例如，一个Redis街道口在端口6379坚挺客户端连接，那么它的集群总线端口16379也会被打开。

节点到节点的通讯只使用集群总线，同时使用集群总线协议：有不同的类型和大小的帧组成的二进制协议。集群总线的二进制协议没有被公开文档话，因为他不希望被外部软件设备用来预计群姐点进行对话。

### 集群拓扑

Redis Cluster是一张全网拓扑，节点与其他每个节点之间都保持着TCP连接。 在一个拥有N个节点的集群中，每个节点由N-1个TCP传出连接，和N-1个TCP传入连接。 这些TCP连接总是保持活性(be kept alive)。当一个节点在集群总线上发送了ping请求并期待对方回复pong，（如果没有得到回复）在等待足够成时间以便将对方标记为不可达之前，它将先尝试重新连接对方以刷新与对方的连接。 而在全网拓扑中的Redis Cluster节点，节点使用gossip协议和配置更新机制来避免在正常情况下节点之间交换过多的消息，因此集群内交换的消息数目(相对节点数目)不是指数级的。

### 节点握手

节点总是接受集群总线端口的链接，并且总是会回复ping请求，即使ping来自一个不可信节点。然而，如果发送节点被认为不是当前集群的一部分，所有其他包将被抛弃。

节点认定其他节点是当前集群的一部分有两种方式：

1. 如果一个节点出现在了一条MEET消息中。一条meet消息非常像一个PING消息，但是它会强制接收者接受一个节点作为集群的一部分。节点只有在接收到系统管理员的如下命令后，才会向其他节点发送MEET消息：

   ``` sh
   CLUSTER MEET ip port
   ```

2. 如果一个被信任的节点gossip了某个节点，那么接收到gossip消息的节点也会那个节点标记为集群的一部分。也就是说，如果在集群中，A知道B，而B知道C，最终B会发送gossip消息到A，告诉A节点C是集群的一部分。这时，A会把C注册未网络的一部分，并尝试与C建立连接。

这意味着，一旦我们把某个节点加入了连接图(connected graph)，它们最终会自动形成一张全连接图(fully connected graph)。只要系统管理员强制加入了一条信任关系（在某个节点上通过meet命令加入了一个新节点），集群可以自动发现其他节点。

## 请求重定向

cluster采用去中心化的架构，集群的主节点各自负责一部分槽，客户端的key的节点映射就是对应的重定向

### 节点对请求的处理过程

检查当前key是否存在当前NODE？

- 通过crc16（key）/16384计算出slot
- 查询负责该slot负责的节点，得到节点指针
- 该指针与自身节点比较

若slot不是由自身负责，则返回MOVED重定向

若slot由自身负责，且key在slot中，则返回该key对应结果

若key不存在此slot中，检查该slot是否正在迁出（MIGRATING）？

若key正在迁出，返回ASK错误重定向客户端到迁移的目的服务器上

若Slot未迁出，检查Slot是否导入中？

若Slot导入中且有ASKING标记，则直接操作

否则返回MOVED重定向

#### MOVED重定向

![img](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220003259.png)

- 槽命中：直接返回结果
- 槽不命中：即当前键命令所请求的键不在当前请求的节点中，则当前节点会向客户端发送一个Moved 重定向，客户端根据Moved 重定向所包含的内容找到目标节点，再一次发送命令。

从下面可以看出 php 的槽位9244不在当前节点中，所以会重定向到节点 192.168.2.23:7001中。redis-cli会帮你自动重定向（如果没有集群方式启动，即没加参数 -c，redis-cli不会自动重定向），并且编写程序时，寻找目标节点的逻辑需要交予程序员手动完成。

``` sh
cluster keyslot keyName  # 得到keyName的槽
```

#### ASK 重定向

Ask重定向发生于集群伸缩时，集群伸缩会导致槽迁移，当我们去源节点访问时，此时数据已经可能已经迁移到了目标节点，使用Ask重定向来解决此种情况。

![img](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220003708.png)

#### smart客户端

这两种重定向的机制使得客户端的实现更加复杂，所以就有了smart客户端（JedisCluster）来**减低复杂性，追求更好的性能**。客户端内部负责计算/维护键-> 槽 -> 节点映射，用于快速定位目标节点。

``` sh
cluster slots # 
```

- 将上述映射关系存到本地，通过映射关系就可以直接对目标节点进行操作（CRC16(key) -> slot -> node），很好地避免了Moved重定向，并为每个节点创建JedisPool
- 至此就可以用来进行命令操作

![img](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220003072.png)

## 故障恢复

master节点挂了之后，故障恢复方案

当slave发现自己的master变为FAIL状态时，便尝试进行Failover，以期成为新的master。由于挂掉的master可能会有多个slave。Failover的过程需要经过类Raft协议的过程在整个集群内达到一致， 其过程如下：

- slave发现自己的master变为FAIL
- 将自己记录的集群currentEpoch加1，并广播Failover Request信息
- 其他节点收到该信息，只有master响应，判断请求者的合法性，并发送FAILOVER_AUTH_ACK，对每一个epoch只发送一次ack
- 尝试failover的slave收集FAILOVER_AUTH_ACK
- 超过半数后变成新Master
- 广播Pong通知其他集群节点

![img](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220003848.png)

## 扩容&缩容

### 扩容

当集群出现容量限制或者其他一些原因需要扩容时，redis cluster提供了比较优雅的集群扩容方案。

1. 首先将新节点加入到集群中，可以通过在集群中任何一个客户端执行cluster meet 新节点ip:端口，或者通过redis-trib add node添加，新添加的节点默认在集群中都是主节点。

2. 迁移数据 迁移数据的大致流程是，首先需要确定哪些槽需要被迁移到目标节点，然后获取槽中key，将槽中的key全部迁移到目标节点，然后向集群所有主节点广播槽（数据）全部迁移到了目标节点。直接通过redis-trib工具做数据迁移很方便。 现在假设将节点A的槽10迁移到B节点，过程如下：

   ```sh
   B:cluster setslot 10 importing A.nodeId
   A:cluster setslot 10 migrating B.nodeId
   ```

   循环获取槽中key，将key迁移到B节点

   ``` sh
   A:cluster getkeysinslot 10 100
   A:migrate B.ip B.port "" 0 5000 keys key1[ key2....]
   ```

   向集群广播槽已经迁移到B节点

   ``` sh
   cluster setslot 10 node B.nodeId
   ```

### 缩容

缩容的大致过程与扩容一致，需要判断下线的节点是否是主节点，以及主节点上是否有槽，若主节点上有槽，需要将槽迁移到集群中其他主节点，槽迁移完成之后，需要向其他节点广播该节点准备下线（cluster forget nodeId）。最后需要将该下线主节点的从节点指向其他主节点，当然最好是先将从节点下线。

# 缓存对应问题

在并发场景下，缓存出现的问题

## 缓存穿透

访问不存在数据库和缓存中的数据，进行大量请求不存在的key，导致压力过大

### 解决办法

- 接口层增加校验，如用户鉴权校验，id做基础校验，id<=0的直接拦截；
- 从缓存取不到的数据，在数据库中也没有取到，这时也可以将key-value对写为key-null，缓存有效时间可以设置短点，如30秒（设置太长会导致正常情况也没法使用）。这样可以防止攻击用户反复用同一个id暴力攻击
- 布隆过滤器。bloomfilter就类似于一个hash set，用于快速判某个元素是否存在于集合中，其典型的应用场景就是快速判断一个key是否存在于某容器，不存在就直接返回。布隆过滤器的关键就在于hash算法和容器大小

## 缓存击穿

在数据库中存在，缓存未存在或者过期后，大量并发请求访问，同时读缓存没读到数据，又同时去数据库去取数据，引起数据库压力瞬间增大，造成过大压力。

### 解决办法

- 设置热点数据永远不过期。
- 接口限流与熔断，降级。重要的接口一定要做好限流策略，防止用户恶意刷接口，同时要降级准备，当接口中的某些 服务  不可用时候，进行熔断，失败快速返回机制。
- 加互斥锁

## 缓存雪崩

缓存中的数据大批量过期，在这一时间段内面临大量的请求来进行数据请求

### 解决办法

- 缓存数据的过期时间设置随机，防止同一时间大量数据过期现象发生。
- 如果缓存数据库是分布式部署，将热点数据均匀分布在不同的缓存数据库中。
- 设置热点数据永远不过期。

## 缓存污染

缓存污染问题说的是缓存中一些只会被访问一次或者几次的的数据，被访问完后，再也不会被访问到，但这部分数据依然留存在缓存中，消耗缓存空间

### 解决办法

#### 最大缓存设置

建议把缓存容量设置为总数据量的 15% 到 30%，兼顾访问性能和内存空间开销

配置命令：

```bash
CONFIG SET maxmemory 4gb
```

但是缓存被写满是不可避免的, 所以需要数据淘汰策略

#### 数据淘汰策略

Redis共支持八种淘汰策略，分别为`noeviction`、`volatile-random`、`volatile-ttl`、`volatile-lru`、`volatile-lfu`、`allkeys-lru`、`allkeys-random`和` allkeys-lfu` 策略。

**分为3类型**

不淘汰

- noeviction （v4.0后默认的）

对设置了过期时间的数据中进行淘汰

- 随机：volatile-random
- ttl：volatile-ttl
- lru：volatile-lru
- lfu：volatile-lfu

全部数据进行淘汰

- 随机：allkeys-random
- lru：allkeys-lru
- lfu：allkeys-lfu



## 缓存和数据库一致性

问题产生的原因：Redis做一个缓冲操作，让请求先访问到Redis，而不是直接访问MySQL等数据库

![img](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220053113.jpeg)

**不管是先写MySQL数据库，再删除Redis缓存；还是先删除缓存，再写库，都有可能出现数据不一致的情况**

### 四种相关模式

#### Cache Aside Pattern

![](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220057826.png)

- **失效**：应用程序先从cache取数据，没有得到，则从数据库中取数据，成功后，放到缓存中。

- **命中**：应用程序从cache中取数据，取到后返回。

- **更新**：先把数据存到数据库中，成功后，再让缓存失效。

#### Read/Write Through Pattern

##### Read Through

Read Through 套路就是在查询操作中更新缓存，也就是说，当缓存失效的时候（过期或LRU换出），Cache Aside是由调用方负责把数据加载入缓存，而Read Through则用缓存服务自己来加载，从而对应用方是透明的。

##### Write Through

Write Through 套路和Read Through相仿，不过是在更新数据时发生。当有数据更新的时候，如果没有命中缓存，直接更新数据库，然后返回。如果命中了缓存，则更新缓存，然后再由Cache自己更新数据库（这是一个同步操作）

![](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220057944.png)



#### Write Behind Caching Pattern

![](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220057212.png)



Write Back套路，一句说就是，在更新数据的时候，只更新缓存，不更新数据库，而我们的缓存会异步地批量更新数据库。这个设计的好处就是让数据的I/O操作飞快无比（因为直接操作内存嘛 ），因为异步，write backg还可以合并对同一个数据的多次操作，所以性能的提高是相当可观的。



对应方案

### 方案：队列 + 重试机制

![img](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220101224.png)

- 更新数据库数据；
- 缓存因为种种问题删除失败
- 将需要删除的key发送至消息队列
- 自己消费消息，获得需要删除的key
- 继续重试删除操作，直到成功

### 方案：异步更新缓存(基于订阅binlog的同步机制)

![img](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/202208220101922.png)








# Redis 数据类型

### String
简单动态字符串

### List
双向列表
压缩列表

### Hash
压缩列表
Hash列表

### Sorted Set 
压缩列表
跳表

### Set
Hash表
整数数组

### BitMap 和 HyperLogLogs 


### Streams
流式处理

### Geospatial indexes
地区的

# Redis IO 
读写均由一个线程来进行完成
单线程进行读写
多线程进行访问具有并发问题进行争夺资源
采用I/O多路复用的机制 select/epoll机制
Socket的采用非阻塞模式

# Redis事务

Redis 事务的本质是一组命令的集合。事务支持一次执行多个命令，一个事务中所有命令都会被序列化。在事务执行过程，会按照顺序串行化执行队列中的命令，其他客户端提交的命令请求不会插入到事务执行命令序列中。





## Redis事务相关命令和使用

> MULTI 、 EXEC 、 DISCARD 和 WATCH 是 Redis 事务相关的命令。

- MULTI ：开启事务，redis会将后续的命令逐个放入队列中，然后使用EXEC命令来原子化执行这个命令系列。
- EXEC：执行事务中的所有操作命令。
- DISCARD：取消事务，放弃执行事务块中的所有命令。
- WATCH：监视一个或多个key,如果事务在执行前，这个key(或多个key)被其他命令修改，则事务被中断，不会执行事务中的任何命令。
- UNWATCH：取消WATCH对所有key的监视。

### 


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

选举机制采用的是[Raft选举算法](https://pdai.tech/md/algorithm/alg-domain-distribute-x-raft.html)：**选举的票数大于等于num(sentinels)/2+1时，将成为领导者，如果没有超过，继续选举**

但是哨兵不能完成主从切换

#### 主库选举

- 过滤掉不健康的（下线或断线），没有回复过哨兵ping响应的从节点

- 选择`salve-priority`从节点优先级最高（redis.conf）的

- 选择复制偏移量最大，只复制最完整的从节点

  ![db-redis-sen-3](https://cdn.jsdelivr.net/gh/Rockly97/netNoteWork/img/db-redis-sen-3.jpg)

  

# Redis Cluster （分片技术）

在Reids3.0版本中推出这个技术， 主要解决读写分离时虽然slave节点扩展了主从的读并发能力，但是**写能力**和**存储能力**是无法进行扩展，就只能是master节点能够承载的上限。



# 缓存问题
在并发场景下，缓存出现的问题

## 缓存穿透

访问不存在数据库和缓存中的数据，进行大量请求不存在的key，导致压力过大



## 缓存击穿

在数据库中存在，缓存未存在或者过期后，大量请求访问导致系统宕机



## 缓存雪崩

缓存中的数据大批量过期，在这一时间段内面临大量的请求来进行数据请求



## 缓存污染

缓存污染问题说的是缓存中一些只会被访问一次或者几次的的数据，被访问完后，再也不会被访问到，但这部分数据依然留存在缓存中，消耗缓存空间

### 数据淘汰策略

Redis共支持八种淘汰策略，分别为`noeviction`、`volatile-random`、`volatile-ttl`、`volatile-lru`、`volatile-lfu`、`allkeys-lru`、`allkeys-random`和` allkeys-lfu` 策略。

分为3类型



## 缓存和数据库一致性












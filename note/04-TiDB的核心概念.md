# 1-存储引擎-RocksDB

## 写入

RocksDB于TiDB，类似InnoDB于MySQL。TiKV的数据没有直接写在磁盘上，内存与磁盘中间夹了一层RocksDB。对于TiKV来说数据写入RocksDB，具体的刷盘策略、机制由RocksDB负责。

![01](04-TiDB的核心概念.assets/01.png)

写入一条数据，先刷盘到WAL文件里，防止数据丢失**（类似MySQL Redo Log一阶段提交的功能？）**，然后写入内存里的MemTable，MemTable被写满后会复制成Immutable Memtable（有点类似Kafka的Batch机制），RocksDB将Immutable Memtable刷盘。MemTable和Immutable Memtable可以抽象理解为InnoDB的BufferPool，也因为Immutable的存在，MemTable写操作在刷盘时不会被阻塞，因为刷的是Immutable Memtable。

Immutable Memtable被flush pipeline流控着，如果Memtable的写入速度太快，超过Immutable Memtable的刷盘次数，Immutable Memtable会在flush pipeline堆积，当flush pipeline的Immutable Memtable≥5个时，Memtable的写入速度就会被放缓，同时发出日志告警。

## 存储

![02](04-TiDB的核心概念.assets/02.png)

在RocksDB里，磁盘里的数据有Level之分，从0开始递增有多个Level。每一个Level都有它的数据容量，当前Level的数据容量到达阈值后，会将当前Level的数据进行压缩，迁移存放到下一个Level。对于Level0来说，容量阈值是4个Immutable，对于Level1来说，容量阈值是256M，对于Level2来说，容量阈值是2.5GB......每一级Level都被上一级Level要高。

## 查询

![03](04-TiDB的核心概念.assets/03.png)

**TiDB的KV数据存储是有序的，排序依据是Key的二进制值**。查询过程中优先看Block Cache，再看MemTable，再看Immutable，实在没有就从磁盘获取，即使是从磁盘取，也是按照Level从低到高的排序查询。但这样会遍历多个Immutable，效率有点低，因此RocksDB为每一个Level维护了【当前Level的key范围】，遍历的时候只需判断要查找的key是否命中当前范围就好了，如果不命中则直接跳到下一个Level。除此以外，RocksDB还提供了一个布隆过滤器，协助判断数据是否命中Level。

# 2-数据同步

![04](04-TiDB的核心概念.assets/04.png)

回顾一下，Region作为存储单位，KV按照Key的二进制排序，有序存放在Region内，1个Region的默认容量是96MB，Region的容量超出144MB后，会进行Region的拆分，形成2个Region。

Region与Region之间基于Raft协议进行数据同步，通过TiDB Server操作写请求到TiKV，实际上TiKV收到的是一条一条的Log，一个写操作对应一个Log。Log也是KV格式，Key是${Region编号}_${Log的ID}，Value是log{实际操作数据}。Region收到Log后，不会立刻走RocksDB的存储逻辑，而是先走一个同步过程。

同步过程指的是Leader与Follower之间的同步，同步的数据单位是Log。**首先要明确，RocksDB的存储过程基于RocksDB KV，同步过程基于RocksDB Raft，每一台TiKV节点同时拥有Rocks KV与RocksDB Raft**。Log被Leader接收后，会经历以下步骤：

1. Propose：将Log标记为 “已收到”。
2. Append：将Log写入RocksDB Raft。
3. Replicate：将Log同步给Follower。
4. Replicate-Append：Follower将Log写入自己的RocksDB Raft，写入完成后向Leader响应Ack。
5. Committed：Leader收到半数以上的Ack（包含自身Append后的Ack），就会认为本次Log同步成功。
6. Apply：将Log写入RocksDB KV，同时通知Follower也将Log写入RocksDB KV。

之后，就是走RocksDB的存储逻辑了（Memtable → Immutable → Disk）。

# 3-MVCC

TiDB的MVCC是用来解决不可重复读冲突的，MySQL也不例外。假设有这么一个场景：

1. 事务A在时间戳T开始，读取第N行数据。
2. 事务B在时间戳T+1开始，修改第N行数据的值。
3. 事务A在时间戳T+2，读到第N行数据。

明明事务A从T开始读，却会读到事务B在T+1修改的新数据，也就是不可重复读。如果没有MVCC，需要事务A对行上锁（比如FOR UPDATE），防止其他事务修改，但这样效率太差了。对于MySQL来说，可以通过RR隔离级别避免，本质是通过ReadView + 遍历行记录的undo log链实现MVCC，从而读到正确的值。

**对于TiDB来说，MVCC基于TSO进行**（联动03-9的因果一致性事务）。TSO作为版本号添加到行记录Key的后缀，多个事务之间的写操作本质是行记录的追加。比如：

1. 事务A在时间戳T新增行记录N，此时Region存放[KeyN_T → value]
2. 事务B在时间戳T+1修改行记录N，此时Region存放[KeyN_T → value , KeyN_T+1 → value2]

这样不会因为数据量冗余，导致存储空间告警吗？回顾一下上面RocksDB的存储，数据从低Level到高Level会经过压缩，我猜压缩就是针对同Key不同TSO的记录取最新值，舍弃旧值（类似Kafka的消息压缩策略）。

因为KeyN作为Key的前缀是一致的，所以N记录代表的KV都会处于同一个Region。有了这个TSO的存在，事务A不会读到事务B修改的数据，因为事务A开始的TSO ＜ 事务B开始的时间戳。
# 整体架构

![01](01-TiDB的架构.assets/01.png)

![02](01-TiDB的架构.assets/02.png)

对于TiDB来说，最核心的是：Metadata、TiDB Cluster、Storage Cluster这3部分。TiSpark是用来做流处理的。关于TiDB的架构，有以下要点：

1. TiDB兼容MySQL协议，也就是说客户端可以使用MySQL的语句来操作TiDB Cluster。
2. Storage Cluster包含了许多TiKV节点、TiFlash节点，数据也是存储在这些节点上。
3. 顾名思义，对于TiKV来说，数据以K-V结构组成的键值对存储，有点类似Redis的结构。
4. TiDB作为一个分布式的数据库，数据在TiKV上存储，肯定不是普通的一份，它会采用跨TiKV分片冗余的方式存储。数据在TiKV上的分片单位叫**Region，它有Leader和Follower的身份，Leader负责写入，Follower负责读取和备份。我认为可以把Region理解为Kafka的Replica**。
5. 客户端的写请求、读请求到达TiDB Cluster以后，TiDB Cluster作为1个无状态的服务，数据实际存储在Storage Cluster上。TiDB Cluster需要和Storage Cluster进行交互（通过KV API和DistSQL API），完成数据的写入 与 读取。
6. PD Cluster它也有Leader、Follower之分，存储着整个TiDB集群的元数据，包含数据所在的TiKV位置。也就是说，当TiDB准备插入、查询数据时，会先经过PD Cluster获取数据对应的位置（Data Location），再进行下一步操作。有点类似Zookeeper或Kraft对于Kafka的作用。
7. TiDB和PD Cluster还会交互一种叫TSO的时间戳，因为TiDB本身支持CP的分布式事务，为了维护CP分布式事务，TiDB会进行多次数据行为。TSO正是为了保证多次数据行为的有序性而存在的。

# TiDB Server的作用

TiDB Cluster由多台TiDB Server组成，TiDB Server的架构图如下：

![03](01-TiDB的架构.assets/03.png)

1. TiDB Server最大的作用是，对客户端假装成一个MySQL数据库，接收来自客户端的MySQL协议请求，最终生成一个执行计划。通过Distributed Executor 和 Storage Cluster进行交互，**Distributed Executor可以并行地执行1个客户端请求的SQL语句（这就是Golang的好处了，用Goroutine实现高IO操作）**。
2. TiDB Server本身是无状态的，它不负责存储任何数据。即使某台Serer挂了，客户端可以通过负载均衡器，将请求转发到另一台TiDB Server上执行，也是可以的。
3. TiDB通过Protocol Layzer实现MySQL协议，通过Session Context校验和保存 连接 的会话信息。
4. 请求到达Parser后，会询问PD Cluster，确定这次目标数据的Data Location。并且将客户端传过来的SQL解析成语法树。
5. 语法树会被Logical Optimizer进行优化，改写一部分内容，改善SQL语句的执行效率，并且生成Logical Plan。Logical Plan被Physical Optimzier再次优化，**这次优化有很大不同，因为会参考前面执行结果生成的统计信息**，生成最终的Physical Plan（优化再优化的结果）。
6. Physical Plan会被两个执行器执行：Local Executor和Distributed Executor，前者是处理那些 影响TiDB Server本身的请求，后者是处理那些 影响Storage Cluster实际数据的请求。
7. Storage Cluster返回数据后，经过Distributed Executor、Local Executor、Session Context、Protocol Layer返回给客户端。

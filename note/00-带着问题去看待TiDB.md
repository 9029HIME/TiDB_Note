## 1-TiDB是一个天然的分布式数据库，号称能进行无缝扩容缩容，这是怎么做到的？

//TODO

## 2-TiDB会不会比MySQL更适合上Kubernetes？

//TODO

## 3-假设我写入一行数据，对于Region来说是发生了改变，那这行数据是立刻同步到其他Follower上，同步成功后才返回写入成功吗？（就类似MySQL的MGR集群那样），但我看目前的笔记（01-4），应该不是这样的，那TiDB如何保证写入一条数据的可靠性？

实际上，结合04-2可以知道，数据写入TiKV后就会基于RocksDB Raft走同步过程，同步完成后才会走RocksDB KV的存储过程。通过这样的方式保证写入一条数据的可靠性。

## 4-缓存表是保存在TiDB Server上的，其他TiDB Server对这张表在TiKV的数据进行修改后，如何保证缓存与TiKV的一致性呢？

//TODO

## 5-基于问题3的回答，如果1个TiDB事务执行了N条写语句，那这N条写语句也是先同步，再存储吗？如果是的话，最后我执行了回滚，如何保证Follower的数据回滚？

//TODO
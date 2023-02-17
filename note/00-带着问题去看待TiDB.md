## 1-TiDB是一个天然的分布式数据库，号称能进行无缝扩容缩容，这是怎么做到的？

//TODO

## 2-TiDB会不会比MySQL更适合上Kubernetes？

//TODO

## 3-假设我写入一行数据，对于Region来说是发生了改变，那这行数据是立刻同步到其他Follower上，同步成功后才返回写入成功吗？（就类似MySQL的MGR集群那样），但我看目前的笔记（01-4），应该不是这样的，那TiDB如何保证写入一条数据的可靠性？

//TODO

## 4-缓存表是保存在TiDB Server上的，其他TiDB Server对这张表在TiKV的数据进行修改后，如何保证缓存与TiKV的一致性呢？

//TODO
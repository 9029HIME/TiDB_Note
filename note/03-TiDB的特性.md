# 1-闪回表（顺带提出MVCC）

TiDB自身有垃圾回收机制（依赖Golang），TiDB的闪回表（Flashback Table）指的是GC之前被还原的已删除表。

假设有一张表ft01，它在2023-02-16 19:00:00被删除，但我后悔了，想恢复它。

我可以通过SELECT * FROM mysql.tidb WHERE variable_name = 'tikv_gc_safe_point'找到最近一次GC的时间。

如果tikv_gc_safe_point < 2023-02-16 19:00:00，那ft01可以被闪回，反之则不行。

如果ft01可以闪回，那可以通过 flashback table ft01 to new_table，将ft01里面**处于tikv_gc_safe_point之后的数据**闪回到新表new_table。

闪回表基于TiDB的MVCC实现，TiDB对于MVCC的实现是：当KV被执行写操作后，新的行记录值的Key会附带一个时间戳，时间戳的值是修改时刻，原来的Key还不会被删除。落实到图上，是这样的：

![01](03-TiDB的特性.assets/01.png)

**当然，实际没有这么简单，这得到后面了解TiDB的MVCC，才能发现本质。或许和MySQL的类似，也是基于链表呢？**
# 淘汰算法

| 算法 | 描述 |
| --- | --- |
| volatile-lru | 在设置了过期时间的key中，移除最近最少使用的key，如果数据最近被访问过，那么将来被访问的几率很大。 |
| volatile-lfu | 在设置了过期时间的key中，移除最近最不经常使用的key，在一段时间内使用次数最少，那么将来被使用的几率不大。 |
| volatile-random | 在设置了过期时间的key中，随机删除key。 |
| volatile-ttl | 在设置了过期时间的key中，优先删除更早过期时间的key。 |
| allkeys-lru | 在所有key中，使用 lru 算法进行删除。 |
| allkeys-lfu | 在所有key中，使用 lfu 算法进行删除。 |
| allkeys-random | 在所有key中，随机删除。 |
| noeviction | 内存不足时，写入数据报错。 |

**LRU：Least Recently Used（最近最少使用）**

重点是使用的频率，如果一个数据最近被访问过，那么将来被访问的几率很大。

**LFU：Least Frequently Used（最不经常使用）**

重点是使用的次数，如果一个数据在最近这段时间几乎没访问过，那么将来被访问的几率很小。

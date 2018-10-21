# redis数据淘汰策略

#### 背景 
先前已经遇到好几次因redis内存满了导致数据丢失的情况，引发了一些意想不到的事故；一张典型的redis内存溢出曲线图：
![](https://github.com/lvsz1/db/blob/master/redis/res/redis_memory1.png)

#### redis数据淘汰策略相关指令
查看redis最大内存配置:
```
config get maxmemory
```

redis 当前内存占用情况：
```
info 
查看memory相关信息，例如：
# Memory
used_memory:816152
used_memory_human:797.02K
used_memory_rss:2535424
used_memory_peak:816152
used_memory_peak_human:797.02K
used_memory_lua:36864
mem_fragmentation_ratio:3.11
mem_allocator:jemalloc-3.6.0

used_memory : 由 Redis 分配器分配的内存总量，以字节（byte）为单位
used_memory_human : 以人类可读的格式返回 Redis 分配的内存总量
used_memory_rss : 从操作系统的角度，返回 Redis 已分配的内存总量（俗称常驻集大小）。这个值和 top 、 ps 等命令的输出一致。
used_memory_peak : Redis 的内存消耗峰值（以字节为单位）
used_memory_peak_human : 以人类可读的格式返回 Redis 的内存消耗峰值
used_memory_lua : Lua 引擎所使用的内存大小（以字节为单位）
mem_fragmentation_ratio : used_memory_rss 和 used_memory 之间的比率
mem_allocator : 在编译时指定的， Redis 所使用的内存分配器。可以是 libc 、 jemalloc 或者 tcmalloc 。
```

查看redis数据淘汰策略：
```
config get maxmemory-policy
例如：
1) "maxmemory-policy"
2) "volatile-lru"
```

#### 数据淘汰策略类型
主要有如下6中策略：
```
volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用 的数据淘汰
volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数 据淘汰
volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据 淘汰
allkeys-lru：从数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰
allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰
no-enviction（驱逐）：禁止驱逐数据
```
redis.conf 配置：
```
# MAXMEMORY POLICY: how Redis will select what to remove when maxmemory
# is reached. You can select among five behaviors:
#
# volatile-lru -> remove the key with an expire set using an LRU algorithm
# allkeys-lru -> remove any key according to the LRU algorithm
# volatile-random -> remove a random key with an expire set
# allkeys-random -> remove a random key, any key
# volatile-ttl -> remove the key with the nearest expire time (minor TTL)
# noeviction -> don't expire at all, just return an error on write operations
#
# Note: with any of the above policies, Redis will return an error on write
#       operations, when there are no suitable keys for eviction.
#
#       At the date of writing these commands are: set setnx setex append
#       incr decr rpush lpush rpushx lpushx linsert lset rpoplpush sadd
#       sinter sinterstore sunion sunionstore sdiff sdiffstore zadd zincrby
#       zunionstore zinterstore hset hsetnx hmset hincrby incrby decrby
#       getset mset msetnx exec sort
#
# The default is:
#
# maxmemory-policy noeviction
```

### 源码分析
#### 版本
```
redis--3.0.7
```



#### 数据淘汰策略


#### 参考资料
<http://redisdoc.com/server/info.html>
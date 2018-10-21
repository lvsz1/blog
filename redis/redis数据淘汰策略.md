# redis数据淘汰策略

#### 背景 
先前已经遇到好几次因redis内存满了导致数据丢失的情况，引发了一些意想不到的事故；一张典型的redis内存溢出曲线图：


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




#### 参考资料
<http://redisdoc.com/server/info.html>
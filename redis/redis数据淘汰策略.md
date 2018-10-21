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

### 源码分析
#### 版本
```
redis--3.0.7
```


#### key过期策略
1、每次执行相关操作时，如果发现key过期，则清除  
在每次进行redis key相关操作时，会调用expireIfNeeded(),检测key是否过期，如果过期，则直接清除     

2、周期检测并清除过期key     
serverCron：每隔1000/server.hz ms执行一次，默认1s中执行10次      
其中，清除key的周期函数在 redis.c activeExpireCycle()函数中，大概函数如下：     
```
void activeExpireCycle(int type) {
	...
	//单次清除过期key的最长时间， 默认25%周期执行时间
    timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;

    ... 

	//遍历指定个数的dbs
    for (j = 0; j < dbs_per_call; j++) {
        int expired;
        redisDb *db = server.db+(current_db % server.dbnum);
        current_db++;

		...

        do {

            ...

            if (num > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP)
                num = ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP;

			// 默认从所有设定过期时间的keys中随机取20个，清除过期keys，并统计过期key个数
            while (num--) {
                dictEntry *de;
                long long ttl;

                if ((de = dictGetRandomKey(db->expires)) == NULL) break;
                ttl = dictGetSignedIntegerVal(de)-now;
                if (activeExpireCycleTryExpire(db,de,now)) expired++;
                if (ttl < 0) ttl = 0;
                ttl_sum += ttl;
                ttl_samples++;
            }

			... 

			// 每执行16次，则检测执行清理过期keys的时间是否大于timelimit，如果大于，则退出，防止执行时间过长影响redis响应等。
            iteration++;
            if ((iteration & 0xf) == 0) { /* check once every 16 iterations. */
                long long elapsed = ustime()-start;
                latencyAddSampleIfNeeded("expire-cycle",elapsed/1000);
                if (elapsed > timelimit) timelimit_exit = 1;
            }
            if (timelimit_exit) return;
            
            // 如果随机的keys中，过期key比例低于25%，则退出
        } while (expired > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4);
    }
}
```
由代码中可以看出：   
1、周期检测清除过期key的执行时间不会超过周期时间的25%，以每秒执行10cron为例，每次清除key的时间不会超过 25ms；
2、周期清除key时，在随机获取的keys中，如果过期keys占随机keys比例不超过25%时，则退出清除；


#### 参考资料
<http://redisdoc.com/server/info.html>
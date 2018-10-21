
# redis 键过期删除策略

#### 过期删除策略
过期阐述策略主要有三种：   
1、定时删除 
+ 含义：在设置key的过期时间的同时，为该key创建一个定时器，让定时器在key的过期时间来临时，对key进行删除
+ 优点：保证内存被尽快释放
+ 缺点：若过期key很多，删除这些key会占用很多的CPU时间，在CPU时间紧张的情况下，CPU不能把所有的时间用来做要紧的事儿，还需要去花时间删除这些key；定时器的创建耗时，若为每一个设置过期时间的key创建一个定时器（将会有大量的定时器产生），性能影响严重

2、惰性删除
+ 含义：key过期的时候不删除，每次从数据库获取key的时候去检查是否过期，若过期，则删除，返回null。
+ 优点：删除操作只发生在从数据库取出key的时候发生，而且只删除当前key，所以对CPU时间的占用是比较少的，而且此时的删除是已经到了非做不可的地步（如果此时还不删除的话，我们就会获取到了已经过期的key了）
+ 缺点：若大量的key在超出超时时间后，很久一段时间内，都没有被获取过，那么可能发生内存泄露（无用的垃圾占用了大量的内存）

3、定期删除
+ 含义：每隔一段时间执行一次删除过期key操作
+ 优点：通过限制删除操作的时长和频率，来减少删除操作对CPU时间的占用--处理"定时删除"的缺点；定期删除过期key--处理"惰性删除"的缺点
+ 缺点：在内存友好方面，不如"定时删除"；在CPU时间友好方面，不如"惰性删除"
+ 难点：合理设置删除操作的执行时长（每次删除执行多长时间）和执行频率（每隔多长时间做一次删除）（这个要根据服务器运行情况来定了）

#### redis 过期删除策略
redis过期删除策略采用的是： 惰性删除 + 定期删除的方式，具体操作方式如下：     

1、每次执行相关操作时，如果发现key过期，则清除        
在每次进行redis key相关操作时，会调用expireIfNeeded(),检测key是否过期，如果过期，则直接清除   
```
int expireIfNeeded(redisDb *db, robj *key) {
	//获取过期时间
    mstime_t when = getExpire(db,key);
    mstime_t now;
	//如果没有设置，则直接返回
    if (when < 0) return 0; /* No expire for this key */

	...

	//如果还没有超时，直接返回；否则，执行下边的删除操作
    if (now <= when) return 0;

    //过期删除key，并通知slave等；
    server.stat_expiredkeys++;
    propagateExpire(db,key);
    notifyKeyspaceEvent(REDIS_NOTIFY_EXPIRED,
        "expired",key,db->id);
    return dbDelete(db,key);
}
```      

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
<https://www.cnblogs.com/java-zhao/p/5205771.html>
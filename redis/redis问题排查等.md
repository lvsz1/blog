# redis 问题排查等

### 一、redis延迟时间排查
**1、查看网络延时命令**
```
redis-cli --latency -h 127.0.0.1 -p 6379 -a ***
```
输出时间的单位为ms。网络时延大小是否属于正常，需经验确定，如果内网中时延超过10ms，可能出问题啦~    

**2、slowlog**    
可以通过 slowlog 查看延时的命令

在配置文件中，与slowlog相关的参数设置： 
```
slowlog-log-slower-than 10000  #单位为微秒，如果执行时间超过该数值，会记录
slowlog-max-len 128  #慢日志队列长度，FIFO队列
```

可以通过指令获取该两个配置：
```
127.0.0.1:6379> config get slowlog-log-slower-than
1) "slowlog-log-slower-than"
2) "10000"
127.0.0.1:6379> config get slowlog-max-len
1) "slowlog-max-len"
2) "128"
```

获取慢日志：
```
127.0.0.1:6379> debug sleep 1   	#模拟耗时1s
OK
(1.00s)
127.0.0.1:6379> slowlog 			#查看保存的慢日志条数
(integer) 1
127.0.0.1:6379> slowlog get 10		#获取最近10条的慢日志
1) 1) (integer) 11					#日志编号
   2) (integer) 1540125881			#指令的执行时间 
   3) (integer) 1000112				#指令耗时，单位微秒
   4) 1) "debug"					#具体指令
      2) "sleep"
      3) "1"
```

**3、查看客户端连接数**   
指令查看客户端连接信息：
```
127.0.0.1:6379> info clients
# Clients
connected_clients:242
client_longest_output_list:0  	#执行命令所返回的结果会保存到output buffer，返回给客户端;如果该值比较大则异常
client_biggest_input_buf:0
blocked_clients:0
```
在redis.conf配置中，有关于client_longest_output_list，限制output buffer内存大小，超过限制则关闭client，防止client因output_buffer占用内存过高
```
client-output-buffer-limit normal 10mb 10mb 0
```

在配置文件中，与maxclients相关的参数设置：
```
maxclients 10000
```

查看最大连接数配置：
```
127.0.0.1:6379> config get maxclients
1) "maxclients"
2) "10000"
```

查看连接到redis的client：
```
redis-cli -h 127.0.0.1 -p 6379 -a *** client list 
输出：
id=211243895 addr=127.0.0.1:25554 fd=204 name= age=0 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=4 oll=0 omem=0 events=rw cmd=lpush
```


### redis 内存相关
** 1、查看内存是否溢出 **
```
查看内存使用情况：
127.0.0.1:6379> info memory
# Memory
used_memory:3068145008    		#由 Redis 分配器分配的内存总量，以字节（byte）为单位
used_memory_human:2.86G 		#以人类可读的格式返回 Redis 分配的内存总量
used_memory_rss:4003962880		#从操作系统的角度，返回 Redis 分配的内存总量（俗称常驻集大小）。这个值和 top 、 ps 等命令的输出一致
used_memory_peak:4376663256
used_memory_peak_human:4.08G 	#Redis 的内存消耗峰值（以字节为单位）
used_memory_lua:35840 			#Lua 引擎所使用的内存大小（以字节为单位）
mem_fragmentation_ratio:1.31	#used_memory_rss 和 used_memory 之间的比率
mem_allocator:jemalloc-3.6.0	#在编译时指定的， Redis 所使用的内存分配器

获取redis配置的最大内存
127.0.0.1:6379> config get maxmemory
1) "maxmemory"
2) "18000000000"

比较配置的 maxmemory 和 info输出的used_memory两个值，判断是否超出限制
```


### redis 主从相关
** 1、查看主从连接状态 ** 
```
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.2,port=6379,state=online,offset=9093221,lag=1
master_repl_offset:9093221
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:8044646
repl_backlog_histlen:1048576

可以查看lag的参数，如果是0或1则正常，否则处于异常状态
可以对比slave与master 的offset值，判断slave和master是否已经完全数据一致；如果slave的offset小于master_repl_offset，说明slave没有完全同步master的数据。 

如果想监控redis主从延时，可以通过slave 与 master 的offset字段来进行判断。
```


#### 参考资料
问题排查总结：<https://www.cnblogs.com/mushroom/p/4738170.html>     
因redis monitor引起的内存上升问题：<https://blog.csdn.net/liqfyiyi/article/details/50894004>
<https://github.com/DataDog/dd-agent/issues/98>     
client_longest_output_list参数问题：<https://github.com/DataDog/dd-agent/issues/98>    

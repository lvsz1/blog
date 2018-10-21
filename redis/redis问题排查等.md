# redis 问题排查等

#### 一、redis延迟时间排查
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





#### 参考资料
问题排查总结：<https://www.cnblogs.com/mushroom/p/4738170.html>     
因redis monitor引起的内存上升问题：<https://blog.csdn.net/liqfyiyi/article/details/50894004>
<https://github.com/DataDog/dd-agent/issues/98>     
client_longest_output_list参数问题：<https://github.com/DataDog/dd-agent/issues/98>    

# redis 主从复制

### 背景
先前遇到redis主从的一个现象：在一个脚本中，通过连接master设置了key-value,立马连接slave获取key，value可能不是最新值。很显然，redis主从复制（增量）是异步的，这就牵扯出一个问题：什么时候进行主从复制呢？流程是怎么样的呢？主从延时会多久呢？

### 抓包
首先抓包看一下 redis 主从设备到底通信传输了那些东西：   

主从配置：
```
master：192.168.0.1 6379 slave：192.168.0.2 6379
```

在不进行redis读写操作的情况下：  
每隔1s slave 向 master发送：
```
22:58:11.323667 IP 192.168.0.2.30178 > 192.168.0.1.6379: Flags [P.], seq 74:111, ack 1, win 14100, length 37
22:58:11.324863 IP 192.168.0.1.6379 > 192.168.0.2.30178: Flags [.], ack 111, win 14600, length 0
```
发送的内容如下图所示：发送REPLCONF指令（这是干什么用的呢？？？）
![](https://github.com/lvsz1/db/blob/master/redis/res/redis-slave-send.png)

每隔10s master 向 slave 发送：
```
22:58:17.999065 IP 192.168.0.1.6379 > 192.168.0.2.30178: Flags [P.], seq 1:15, ack 333, win 14600, length 14
22:58:17.999104 IP 192.168.0.2.30178 > 192.168.0.1.6379: Flags [.], ack 15, win 14100, length 0
```
发送的内容如下图所示：发送Ping心跳包，检测slave是否健康
![](https://github.com/lvsz1/db/blob/master/redis/res/redis-master-send.png)


在进行redis master写的情况下：   
在master上执行如下指令：
```
127.0.0.1:6379> set name bob
```
抓包内容如下：
```
23:26:19.854913 IP 192.168.0.1.6379 > 192.168.0.2.30178: Flags [P.], seq 1:33, ack 259, win 14600, length 32
23:26:19.854936 IP 192.168.0.2.30178 > 192.168.0.1.6379: Flags [.], ack 33, win 14100, length 0
```
很显然，master向slave发送了相关的同步指令，其指令如下：
![](https://github.com/lvsz1/db/blob/master/redis/res/redis-masrer-set-name.png)

### 源码分析

#### 版本
```
redis--3.0.7
```

#### 参数加载

在配置主从时，需要在redis.conf文件中slaveof参数，配置类似如下：
```
slaveof 192.168.0.2 6379
```
redis在初始化参数时，如果不提供配置文件，主从状态的参数配置如下
```
server.repl_state = REDIS_REPL_NONE; //默认为master，不进行主从配置
```
如果指定了配置文件，并配置了slaveof参数，参数初始化如下(配置文件解析过滤到空行及'#'开头的配置，解析比nginx简单的多)：
```
...
else if (!strcasecmp(argv[0],"slaveof") && argc == 3) {
    slaveof_linenum = linenum;
    server.masterhost = sdsnew(argv[1]);
    server.masterport = atoi(argv[2]);
    server.repl_state = REDIS_REPL_CONNECT;
}
...
```
 通过slaveof配置，拿到master的ip:addr，并将主从复制状态置为 REDIS_REPL_CONNECT 

其中，redis的主从复制状态有如下三种类型：
```
#define REDIS_REPL_NONE 0 /* No active replication */
#define REDIS_REPL_CONNECT 1 /* Must connect to master */
#define REDIS_REPL_CONNECTING 2 /* Connecting to master */
```

当然，还有一些主从的其它配置参数。

#### 涉及的cron
初始化时，会初始化一个时间事件，周期处理serverCron，初始化如下：
```
//主要采用select、poll、epoll设置超时，超时后执行（这里第一次超时时间是1ms，后边超时时间为 1000/server.hz，其中server.hz 默认为10，也就是serverCron默认是100ms执行一次）
if(aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
    redisPanic("Can't create the serverCron time event.");
    exit(1);
}
```

超时后，执行serverCron，并更新超时时间；（这里时间管理采用链表，找到最近的超时时间，并在wait时设置。如果在超时之前有事件过来，处理事件后悔检查当前时间是否已到达设定的超时时间，以确保超时事件的执行）
```
retval = te->timeProc(eventLoop, id, te->clientData); //获取新的超时时间
...
if (retval != AE_NOMORE) {
    aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
} else {
    aeDeleteTimeEvent(eventLoop, id);
}
```

在serverCron到底做了哪些与主从复制相关的东西呢？


TODO。。。
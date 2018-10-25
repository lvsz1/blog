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

超时后，执行serverCron，并更新超时时间；
（这里时间管理采用链表，找到最近的超时时间，并在wait时设置。
如果在超时之前有事件过来，处理事件后会检查当前时间是否已到达设定的超时时间，
以确保超时事件的执行）
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
每个1s执行 replicationCron（）函数，其中该函数进行如下操作：    
1、根据配置，进行超时检测（连接、读写超时、心跳超时检测），当然如果还没有连接master，还是要连接master的；  
2、每隔1s，slave向master发送 REPLCONF、ACK，告诉master I’m alive！在这个过程中，slave向master发送了复制offset，master接收到slave offset后与会与自身的offset进行对比，如果小于master offset， 则进行同步未传输的指令。  
3、每隔server.repl_ping_slave_period秒（默认为10秒），master向slaves发送 PING 心跳包；  
4、如果slave如果处于预同步阶段，master向slave发送"\n"(这个地方没有看懂？？？)；  
5、master检测超时时间，断开超时的的连接；  

从serverCron代码来看，抓包抓到的东西主要用于master、slave之间的保活：master ping， slave ack。  
REPLCONF 指令除了保活，还有另一个作用：该指令携带slave 复制偏移量，可以与master 复制偏移量进行对比，如果小于master offset，则master向slave同步缺失的指令，防止slave 与 master数据不一致。    

#### sync/psync可以看相关资料

#### 命令传播
在redis.c processCommand()中，调用call(), 先master处理，然后发送指令，同步给各个slave
```
void call(redisClient *c, int flags) {
	...

    //执行相关指令
    c->flags &= ~(REDIS_FORCE_AOF|REDIS_FORCE_REPL);
    redisOpArrayInit(&server.also_propagate);
    dirty = server.dirty;
    start = ustime();
    c->cmd->proc(c);

    //判断指令是否需要aof及同步给slave，如果需要，则进行命令传播
    if (flags & REDIS_CALL_PROPAGATE) {
        int flags = REDIS_PROPAGATE_NONE;

        if (c->flags & REDIS_FORCE_REPL) flags |= REDIS_PROPAGATE_REPL;
        if (c->flags & REDIS_FORCE_AOF) flags |= REDIS_PROPAGATE_AOF;
        if (dirty)
            flags |= (REDIS_PROPAGATE_REPL | REDIS_PROPAGATE_AOF);
        if (flags != REDIS_PROPAGATE_NONE)
            propagate(c->cmd,c->db->id,c->argv,c->argc,flags);
    }
}
```
propagate()调用replicationFeedSlaves()依次同步各个slave。    

**注意**      
 master在执行每个写指令的时候，先本地执行指令，然后同步的将指令写到client buffer中。但是，虽然同步写入buffer，真正给slave传输信息的时间是不确定，因为redis的写操作采用事件的机制，在每次的addReply时，将写事件注册到select/poll/epoll等（写事件中，回复client、命令传播给slave的先后顺序是不一定的），事件触发后将buffer信息flush到内核的tcp发送缓冲区，然后再是tcp传输。所以说，redis主从复制是异步的。



#### replconf 作用   
为了确保主从数据一致，redis slave每隔1s向master发送 replconf ack offset ；    
1、slave cron每秒向master发送replconf指令，携带自身维护的复制偏移量offset 
```
void replicationSendAck(void) {
    redisClient *c = server.master;

    if (c != NULL) {
        c->flags |= REDIS_MASTER_FORCE_REPLY;
        addReplyMultiBulkLen(c,3);
        addReplyBulkCString(c,"REPLCONF");
        addReplyBulkCString(c,"ACK");
        addReplyBulkLongLong(c,c->reploff);
        c->flags &= ~REDIS_MASTER_FORCE_REPLY;
    }
}
```
2、master接收到replconf指令后，执行replconfCommand() -> putSlaveOnline(),向select/poll/epoll等注册写事件，执行sendReplyToClient()，将 master_offset - slave_offset 的数据传输给slave，用于同步；


### 主从复制相关conf参数

**1、repl-backlog-size 复制积压缓存大小 **         
该参数主要用于指定master缓存指令的空间大小，目的是当slave与master断连后，slave根据自己维护的offset与master维护的offset进行对比，然后进行增量同步。缓冲区的大小最小为：slave于master的平均断连second * master每秒接收的write指令量。该值可以通过如下指令查看：
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
```     
**2、免持久化复制 ** 
```
repo-diskless-sync yes  # yes表示开启在复制时不用写磁盘模式（不用写磁盘生成dump.rdbw文件)；no表示关闭  
```
TODO。。。
#### 参考资料
<http://www.cnblogs.com/kismetv/p/9236731.html>
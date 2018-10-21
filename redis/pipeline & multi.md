#### pipeline 与 multi 区别 

### 背景 
以前听说过pipeline和multi，但不是很明白其中的原理，一直以为是一个东西；前段时间，在写go代码的时候，由于某个redis操作需要用到批量操作，才知道这两个操作还是有很大不同的，有不同的应用场景。

### 源码分析
#### 版本
```
redis--3.0.7
```

### pipeline 
#### client 操作
1、pipeline操作代码：
```
$redis->pipeline();
$redis->set('name', 'zhangsan');
$redis->set('age', 20);
$ret = $redis->exec();
var_dump($ret);
```
代码输出：
```
其中 $ret 为：
array(2) {
  [0]=>
  bool(true)
  [1]=>
  bool(true)
}
```

2、抓包，如图所示：
![](https://github.com/lvsz1/db/blob/master/redis/res/pipeline.png)
![](https://github.com/lvsz1/db/blob/master/redis/res/pipeline_response.png)
redis server 接收到的东西：
```
*3\r\n$3\r\nSET\r\n$4\r\nname\r\n$8\r\nzhangsan\r\n*3\r\n$3\r\nSET\r\n$3\r\nage\r\n$2\r\n20\r\n
```

3、php redis扩展代码没有看过，但看过一点go redis client库的代码，pipeline 实现过程类似如下所示：
```
a、在conn中维护一个缓存，类似如下：
var wb *bufio.Writer 
wb = bufio.NewWrite(conn)

b、pipeline操作时，根据redis协议封装指令，并将操作指令写到 wb 缓冲中，类似如下：
pipeCount = 0
wb.writeString(cmd1)
pipeCount ++
wb.writeString(cmd2)
pipeCount ++

c、pipeline exec时，会将缓存中的指令一并发送给 redis server，类似如下：
wb.Flush()

d、读取redis返回值,类似如下：
for i := 0; i < pipeCount; i++ {
	ret[i]= conn.readResponse()
}
```

通过以上分析，可以发现，pipeline为了节省client与server多条指令传输的RTT时间。client在pipeline时，将指令缓存起来，一并发送给server。其中，这个过程中，从单条tcp上传输的数据来看，并没有额外的指令传输，只是将指令合并发送。    

那么问题来了，redis server接受到批量指令时，如何操作呢？是否是原子的呢？？？

#### server操作 

redis server接受指令流程：
从tcp中读取client数据：
```
在 networking.c readQueryFromClient() 中，
readlen = REDIS_IOBUF_LEN; //定义为1024*16
... 
nread = read(fd, c->querybuf+qblen, readlen);
```
处理指令,在这里，redis-server会处理read到的所有指令，也就是说，如果传过来了两个指令，则这时先会处理所有指令后再去干别的事情：（单次tcp包中的指令序列是原子的，不管pipeline还是multi ？？？）
```
processInputBuffer(c);
其中，代码如下：

while(sdslen(c->querybuf)) {
    /* Return if clients are paused. */
    if (!(c->flags & REDIS_SLAVE) && clientsArePaused()) return;

    /* Immediately abort if the client is in the middle of something. */
    if (c->flags & REDIS_BLOCKED) return;

    /* REDIS_CLOSE_AFTER_REPLY closes the connection once the reply is
     * written to the client. Make sure to not let the reply grow after
     * this flag has been set (i.e. don't process more commands). */
    if (c->flags & REDIS_CLOSE_AFTER_REPLY) return;

    /* Determine request type when unknown. */
    if (!c->reqtype) {
        if (c->querybuf[0] == '*') {
            c->reqtype = REDIS_REQ_MULTIBULK;
        } else {
            c->reqtype = REDIS_REQ_INLINE;
        }
    }

    if (c->reqtype == REDIS_REQ_INLINE) {
        if (processInlineBuffer(c) != REDIS_OK) break;
    } else if (c->reqtype == REDIS_REQ_MULTIBULK) {
        if (processMultibulkBuffer(c) != REDIS_OK) break;
    } else {
        redisPanic("Unknown request type");
    }

    /* Multibulk processing could see a <= 0 length. */
    if (c->argc == 0) {
        resetClient(c);
    } else {
        /* Only reset the client when the command was executed. */
        if (processCommand(c) == REDIS_OK)
            resetClient(c);
    }
}
```

可以看上边的代码（很多地方确实看不懂~），得出一个不确定的结论（自己的理解）：     
不管是pipeline方式还是multi方式，只要单次的tcp包中传输了多个完整的指令，redis-server都会事务执行该tcp包中的所有指令；  
但即使将redis client 可以将多个指令缓存起来（比如用户空间缓存，然后flush到内核空间)，由于内核空间tcp缓冲区的数据不确定什么时候清空（tcp拆包、粘包等，可以看tcp相关协议）,不能确保redis-server单个tcp包获得所有的缓冲指令，因此pipeline方式不能确保指令是事务执行的；为了确保批量指令的事务执行，这时multi起作用了。   

所以，写pipeline时，单个pipe操作，也不要过多的指令~


### multi
#### client操作
1、client端代码
```
$redis->multi();
$redis->set('name', 'zhangsan');
$redis->set('age', 20);
$ret=$redis->exec();
var_dump($ret);
```
2、抓包
![](https://github.com/lvsz1/db/blob/master/redis/res/multi_request1.png)
在multi 和 exec之间 client与server单个发送redis指令 
![](https://github.com/lvsz1/db/blob/master/redis/res/multi_request2.png)
3、处理流程：一次发送一下指令
```
a、MULTI 
b、SET name zhangsan
c、SET age 20 
d、EXEC
```
注意，client并没有将multi操作指令进行用户空间缓存，直接flush到内核空间，然后发送，所以抓包后看看的是每个指令一个tcp包；


#### server

1、redis-server 接收到multi操作后，执行multiCommand()函数,置相关标志位:
```
void multiCommand(redisClient *c) {
    if (c->flags & REDIS_MULTI) {
        addReplyError(c,"MULTI calls can not be nested");
        return;
    }
    c->flags |= REDIS_MULTI;
    addReply(c,shared.ok);
}
```

2、接受指令；在后续接受指令的操作过程中，可以看networking.c readQueryFromClient() -> processInputBuffer() -> processCommand() 中：
```
/* Exec the command */
if (c->flags & REDIS_MULTI &&
    c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
    c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
{
    queueMultiCommand(c);
    addReply(c,shared.queued);
} else {
    call(c,REDIS_CALL_FULL);
    c->woff = server.master_repl_offset;
    if (listLength(server.ready_keys))
        handleClientsBlockedOnLists();
}
```
如果是multi操作，会调用queueMultiCommand()将指令在server端进行缓存，直到接受的EXEC时才进行执行。

3、接收到exec指令后，执行execCommand(),执行multi/.../exec之间接收到的指令
```
    for (j = 0; j < c->mstate.count; j++) {
        c->argc = c->mstate.commands[j].argc;
        c->argv = c->mstate.commands[j].argv;
        c->cmd = c->mstate.commands[j].cmd;

        /* Propagate a MULTI request once we encounter the first write op.
         * This way we'll deliver the MULTI/..../EXEC block as a whole and
         * both the AOF and the replication link will have the same consistency
         * and atomicity guarantees. */
        if (!must_propagate && !(c->cmd->flags & REDIS_CMD_READONLY)) {
            execCommandPropagateMulti(c);
            must_propagate = 1;
        }

        call(c,REDIS_CALL_FULL);

        /* Commands may alter argc/argv, restore mstate. */
        c->mstate.commands[j].argc = c->argc;
        c->mstate.commands[j].argv = c->argv;
        c->mstate.commands[j].cmd = c->cmd;
    }
```

### 结论
1、pipeline是客户端缓存，multi是服务端缓存；    
2、pipeline不能保证事务执行；multi保证事务执行（注意，这里的事务执行，并不是在PHP代码执行multi、exec时，redis server不处理其它client端操作；而是在redis server 接收到exec后，redis server事务执行multi、exec之间的指令，这个时候不会执行其它client端的操作；      
3、pipeline和multi针对的问题不同：pipeline主要针对解决批量指令多次tcp传输造成的网络传输效率低下问题（类似tcp的negle算法，多条指令批量传输）；multi主要针对批量指令原子性执行问题（每条指令仍单次传输，没有改善网络传输效率）；  

另外，至于pipeline 单包发送指令是否是事务执行，可以读读源码~

#### 测试
可以看看以下代码会输出什么~ 这个估计得需要研究一下php redis扩展源码啦~
```
$redis->multi();
$redis->pipeline();
$redis->set('name', 'zhangsan');
$redis->set('age', 20);
$ret=$redis->exec();
var_dump($ret);
var_dump($redis->get('name'));
```


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
![]()
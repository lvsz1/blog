#reids-cli 扫描大键

###背景
有时，由于前期业务redis键设置不合理（未设置过期时间或某个键过大），会造成redis内存接近用满状态或数据倾斜。因此这是需要找出redis中的大key以做处理。（前几天在工作中，采用redis-cli --bigkeys扫描时，发现qps剧增，why？？？）

###指令
redis-cli -h host -p port -a passport  --bigkeys


###源码分析

####版本
```
redis--3.0.7
```

####处理流程

1、命令解析，根据--bigkeys执行相关函数
```
parseOptions()中
...
else if (!strcmp(argv[i],"--bigkeys")) {
    config.bigkeys = 1;
} 
...
/* Find big keys */
if (config.bigkeys) {
    if (cliConnect(0) == REDIS_ERR) exit(1);
    findBigKeys();
}
...
```

2、调用DBSIZE获取redis中所有键数据（o（1）操作），用于计算扫描进度
```
...
redisCommand(context, "DBSIZE");
...
```

3、循环操作

```
do {
	 扫描键...
   } while(it != 0); //直到scan返回的第一个元素为0(scan 返回为0表示扫描结束)
```


计算扫描进度
```
pct = 100 * (double)sampled/total_keys; //进度计算
```

SCAN cursor操作，获取部分键（默认为10个）
```
redisCommand(context, "SCAN %llu", *it); // 获取keys
```

TYPE key操作，获取获得的部分键的类型（0：string 1：list 2：set 3：hash 4：zset）
```
/* Pipeline TYPE commands */
for(i=0;i<keys->elements;i++) {
    redisAppendCommand(context, "TYPE %s", keys->element[i]->str); //该函数会将命令缓存起来，pipeline模式
}
...
/* Retrieve types */
for(i=0;i<keys->elements;i++) {
    if(redisGetReply(context, (void**)&reply)!=REDIS_OK) { //第一次读reply为空时，会将context缓存的指令发送给server，然后阻塞直至查询返回
    	...
    }

    types[i] = toIntType(keys->element[i]->str, reply->str); //将key类型记录下来
    ...
}

```

获取key大小，根据type获取的不同类型，执行不同操作："STRLEN","LLEN","SCARD","HLEN","ZCARD"
```
char *sizecmds[] = {"STRLEN","LLEN","SCARD","HLEN","ZCARD"};
...
/* Pipeline size commands */

redisAppendCommand(context, "%s %s", sizecmds[types[i]],
    keys->element[i]->str); // 缓存指令
...
redisGetReply(context, (void**)&reply) // 批量发送指令，阻塞获取结果

sizes[i] = reply->integer; //保存键大小
```

大小比较操作
```
if(biggest[type]<sizes[i]) { //如果先前指定类型保存的大小小于本次扫描的某个键值，则更新
    printf(
       "[%05.2f%%] Biggest %-6s found so far '%s' with %llu %s\n",
       pct, typename[type], keys->element[i]->str, sizes[i],
       typeunit[type]); //扫描过程中输出当前最大

    /* Keep track of biggest key name for this type */
    maxkeys[type] = sdscpy(maxkeys[type], keys->element[i]->str); //保存当前最大key的name
    if(!maxkeys[type]) {
        fprintf(stderr, "Failed to allocate memory for key!\n");
        exit(1);
    }

    /* Keep track of the biggest size for this type */
    biggest[type] = sizes[i];  // 更新当前最大key的size
}
```

延时操作,延时时间通过参数 -i 指定，单位是秒，默认interval是0，也就是不延时！！！也就是执行时不指定 -i参数，qps会剧增的原因，一直执行。。。
```
/* Sleep if we've been directed to do so */
if(sampled && (sampled %100) == 0 && config.interval) {
    usleep(config.interval);
}
```

4、输出统计结果，每种类型仅输出最大的那个key。（但不过可以通过扫描过程中的输出信息获取中间较大的key）
```
/* Output the biggest keys we found, for types we did find */
//输出每种类型的最大key name以及 size
for(i=0;i<TYPE_NONE;i++) {
    if(sdslen(maxkeys[i])>0) {
        printf("Biggest %6s found '%s' has %llu %s\n", typename[i], maxkeys[i],
           biggest[i], typeunit[i]);
    }
}

printf("\n");
//输出 每种类型的总的数量、大小及占比（这里的占比是 key数量占比，而不是key大小占比）
for(i=0;i<TYPE_NONE;i++) {
    printf("%llu %ss with %llu %s (%05.2f%% of keys, avg size %.2f)\n",
       counts[i], typename[i], totalsize[i], typeunit[i],
       sampled ? 100 * (double)counts[i]/sampled : 0,

```

### 注意
由于执行redis-cli --bigkeys默认扫描过程不延时，尽管采用了scan方式扫描，但是默认每扫描10个key，需要 scan、type、size三次操作（3次网络传输），且扫描过程中不延时，qps上升是比较高的！注意！！！






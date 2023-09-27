
# Redis和Memcached的区别和共同点

- 这两个都是基于内存存储的。
- 最大的区别就是memcached重启完之前数据就没了，而redis有持久化机制，重启完数据还在。
- memcached不支持集群，而redis支持集群。
- memcached只支持k/v数据类型，redis还支持hash、list、set等。
- memcached是多线程、非阻塞IO复用的网络模型，而Redis是单线程的多路IO复用模型。
- memcached过期数据的删除策略只有惰性删除，而Redis有惰性删除与定期删除。
- Redis功能也强大多，支持发布订阅模型、Lua 脚本、事务、分布式锁等功能。

# redis的基本数据类型
### string

- 可以存储常用的k/v，也可以用于分布式锁，或者计数器。
- 底层数据结构是sds。

```shell
(1) 单值缓存
set k v
(2) 对象缓存
set user:1 json
mset user:1:name zhuge user:1:age 18
mget user:1:name user:1:age
(3) 分布式锁
setNx k v
set k v ex 10 nx
(4) 计数器或自增器
incr k
``` 

### hash

- 可以存储对象、也可以用于计数器。
- 方便数据整合存储。
- 相比string的操作，消耗内存与cpu更小，储存更节省空间。
- 底层存储结构是哈希表，数据少且短小时也能用压缩列表存储

```shell
(1) 对象缓存
hmset user:1 name zhuge age 18
(2) 购物车
添加商品：hset cart:user:1 sku1 1 
增加数量：hincrby cart:user:1 sku1 1
商品类型总数：hlen cart:user:1
删除商品：hdel cart:user:1 sku1
获取购物车所有商品：hgetall cart:user:1
```

### list

- 可以存储列表、栈、队列
- 底层存储结构是双向链表，数据少且短小时也能用压缩列表存储

```shell
(1) 存储key
lpush k v [v..]
rpush k v [v..]
(2) 拿key
lpop key 移除并返回
rpop key
lrange key start stop 返回指定区间内的元素，索引从0开始
blpop key [key..] timeout  左边弹出一个元素，若列表没有就阻塞等待
brpop key [key..] timeout  右边弹出一个元素，若列表没有就阻塞等待
(3) statck
栈=lpush+lpop=FIFO
(4) queue
队列=lpush+rpop
(5) blocking mq
阻塞队列=lpush+brpop
(6) 微博和微信公众号的消息列表  （这种只能用于粉丝数少的）
比如A用户关注了B、C
B发消息：lpush msg:A msgId1
C发消息：lpush msg:A msgId2
A查看消息：lrange msg:A 0 10
```

### set

- 可以存储集合、也可以用于抽奖
- 底层存储结构是哈希表，或者是整数集合

```shell
(1) 存储key
SADD key member [member..] 插入key，元素存在则忽略
(2) 删除元素
srem key member [member..] 
(3) 查询
smembers key 获取所有元素 
scard key 获取元素的个数
sismember key member 是否存在
(4) 随机拿出来
srandmember  key [count] 选出count个元素，元素不删除
spop key [count] 选出count 个元素，元素删除
(5) 多个set类型的key的运算操作
sinter key [key... ]  交集运算
sinterstore dest key [key..]  交接结果放到新集合dest中
sunion key [key... ]  并集运算
sunionstore dest key [key... ]  并集结果放到新集合dest中
sdiff key [key... ] 差集运算  这里差集的意思是第一个集合减去后面集合的并集，第一个集合剩多少就是多少
sdiffstore dest key [key... ]  差集结果放到新集合dest中
(5) 抽奖场景
参与抽奖：sadd key user1
查看抽奖的所有用户：smemebers key
抽取count名中奖者 : spop key [count] 选出count 个元素，元素删除
(6) 朋友圈点赞收藏标签
点赞 sadd like:message1 user1
取消点赞: srem like:message1 user1
检查用户是否点赞过   sismember like:message1 user1
获取点赞的用户列表 smembers like:message1
获取点赞的用户数 scard like:message1 
(7) 互相关注
 A和B共同关注的人：sinter aSet  bSet
我(A)关注的人(C、D)也关注他们: sismember C  X ; sismemeber D X; 
我可能认识的人：sdiff A C 
(8) 商品的多条件筛选
sinter 安卓set 8GBset 高清set 
```
 
### zet

- 可以用于有序集合、排行榜
- 底层存储结构是跳表，数据少且短小时也能用压缩列表存储

```shell
(1) 删除
zrem key [member..] 删除一个元素
(2) 查询
zscore key memeber 返回key中某个元素的分值
zrange key start stop [withscores] 正序获取key范围内的元素
zrevrange key start stop  [withscores] 逆序获取key范围内的元素
(4) 修改、插入
zadd key score member [[score member]..] 往key中加入带分值的元素
zincrby key 数值 member 为key里面的元素的分值++数值
(5) 集合操作
zunionstore newkey 后面key的数量 key [key..]  并集计算
zinterstore newkey 后面key的数量 key [key..]  交集计算
(6) 排行榜
点击一次新增加一次热度： zincrby new:id1 1 标题
今天热度排行前十：zrevrange new:天:20220202 0 9 withscores
七日热度排行榜： zuionstore new:新的key:最近天 7 new:天:20220201 new:天:20220202 一直到 new:天:20220207 
七日排行前十：zrevrange  new:新的key:最近天  0 9 withscores
```

### hyperLogLog (特殊类型)

- 用来基数统计的，有点类似set，但是这个类型占用的内存空间小很多。
- 缺点是只能告诉某个元素有没有存在过，不能把所有元素拿出来，统计基数总个数也只是估算的。（因为只会根据输入元素来计算基数，它并不存储元素本身）。
- 可以用于估算每日访问ip/用户数

```shell
(1) 添加元素，计算成基数存储下来
PFADD runoobkey "redis"
(2) 统计基数估算值
PFCOUNT runoobkey
(3) 合并两个HyperLogLog到新的HyperLogLog
PFMERGE destkey sourcekey [sourcekey ...]
```

### geo (特殊类型)

-  主要用于存储地理位置的经度、维度、地名。类似集合。
-  使用geohash来保存地理位置的坐标

```shell
(1) 添加一个地理位置到
GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
(2) 获取地理位置的坐标
GEOPOS Sicily 地名1 地名2 
(3) 计算两个位置之间的距离。
GEODIST Sicily Palermo Catania
(4) 根据用户给定的经纬度坐标来获取指定范围内的地名+地理位置集合。
GEORADIUS Sicily 15 37 100 km
(5) 用户给定的地名来获取指定范围内的地名集合。
GEORADIUSBYMEMBER Sicily Agrigento 100 km
(6) geohash 用于获取一个或多个位置元素的geohash值
GEOHASH Sicily 地名1 地名2 
```

### bitmap(特殊类型)

- bitmap实际上就是string类型，只是每位bit都是0和1，用来代表存储的元素存不存在。
- 布隆过滤器就是基于bitmap
- 可以用于用户行为统计

```shell
(1) 001用户新增对userId为11的标记
setbit beauty_girl_001 11 1
(2) 查询001用户对userId为11的标记
getbit beauty_girl_001 11
(3) 统计被被设置为 1 的位的数量
bitcount beauty_girl_001
(4) BITOP 命令支持 AND 、 OR 、 NOT 、 XOR 这四种操作中的任意一种参数
BITOP operation destkey key [key ...]
```

# redis的数据结构
### sds
### 双向链表
### 压缩列表
### 哈希表
### 整数集合
### 跳表
# 模糊查找固定前缀的key

- 如果用keys指令，数据量大的话，由于redis执行命令是单线程，所以会阻塞其他命令请求。
- 所以建议使用scan指令，支持用游标分批查询。但是有可能遇到 遍历出重复的key、新增的键没被遍历到、在迭代过程中key被修改 这些情况

```
(1) 模糊查询以keyPefix为前缀的key
keys keyPefix*
(2) 游标查询以keyPefix为前缀的key，批量扫描redis所有key中的1000个。（游标第一次是0，第二次根据第一次返回的游标数）
scan 游标 match keyPefix* count 1000
```

# redis过期数据的删除策略 

redis用的是惰性删除+定期删除策略

- 惰性删除：当读或写一个已经过期的key时，再删除key。可能会造成太多过期 key 没有被删除。
- 定期删除：每隔一段时间抽取一批key进行删除。

# redis内存淘汰机制

当前已用内存超过maxmemory限定时，redis会触发内存淘汰。
- (1)针对所有key的淘汰策略有：
  - allkeys-lru: 最近一次访问时间越前越先被删除
  - allkeys-lfu: 最近访问次数越少越先被删除
  - allkeys-random：随机删除
- (2)针对设置了过期时间的key的淘汰策略上面三种都有，多了一个：
  -  volatile-ttl：离过期时间越近的key越先被删除
```
volatile-lru：最近一次访问时间越前越先被删除
volatile-lfu：最近访问次数越少越先被删除
volatile-random：随机删除
```
- (3)还有禁止驱逐策略，是默认的淘汰策略:
  - no-eviction：当内存不足以容纳新写入数据时，新写入操作会报OOM错误。

# redis的持久化机制

redis的持久化机制包括rdb和aof两种方式，默认是用rdb持久化方式。数据恢复的时候，优先使用aof，因为数据比较完整。

- rdb
  - 可以通过save指令生成rdb快照，但是save是在主进程执行的，会阻塞客户端命令。所以redis默认是用bgsave指令生成rdb快照，在配置文件可以配置一定时间内有多少个key有改动就触发bgsave。
  - bgsave的流程就是：redis会在主进程fork一个子进程，由子进程执行持久化，主进程正常处理客户端指令。子进程可以共享主进程的所有内存数据。子进程会把触发持久化那一时刻redis中所有的数据以二进制形式保存到rdb文件里面。如果持久化过程中有修改指令，就会把要修改的数据所在页拷贝一份，子进程把拷贝的数据存到rdb文件，主进程就正常修改原来页的数据。
  - 但bgsave持久化时，有大量的修改操作，就会触发很多页的拷贝，占用的内存就会持续增长，导致内存不够用。
  - 另外，由于每次快照保存的是当时redis全量的数据，数据量大的话，会持久化好几分钟，一旦宕机，持久化过程中新增或修改的数据就没了，丢了好几分钟的数据。
  - 好处就是因为每次快照不同时刻的数据到rdb文件，可以利用rdb文件还原到不同时刻的版本。 而且由于是二进制存储，RDB在恢复大数据集时的速度比AOF的恢复速度要快，很适合于灾难恢复。

<div align=center><img src="https://user-images.githubusercontent.com/27798171/176376539-26cab0c3-3afa-4248-bcac-8b5a32df1af1.png"/></div>

- aof
  - 把修改的每一条指令以resp协议追加到aof文件里面。
  - 主进程处理一条修改指令时，会先修改内存的这条数据。如果你配置的是always写盘策略，那主进程就还得再把这条指令追加到磁盘的aof文件里面。如果你配置的是每秒一次（EverySecond）的写盘策略，那主进程就还得再把这条指令写到aof内存缓冲区，缓冲区满了或者是每隔一秒，就会有一个线程专门把内存缓冲区内积累的指令同步到aof文件。还有一种no写盘策略，和每秒一次写盘策略类似，只不过没有后台线程去执行写入磁盘的操作，而是由操作系统来决定，何时刷新到磁盘。
  - 一般都是用每秒一次这种写盘策略，可以让主线程处理客户端指令时响应快些，因为直接写到缓冲区内存肯定比写到磁盘快。最多也就丢失一秒的数据。
  - 如果redis已经跑了一年，那这个aof文件会非常庞大，恢复数据会特别慢，比如一个key如果incr了好几千次，没必要记录几千次，只需要记录最后的值就好，所以需要对aof进行重写。但是就算重写了，速度也会比rdb慢很多，所以可以用混合持久化。

- aof重写
  - 原理就是主进程fork一个子进程，子进程扫描内存的所有数据，转换为Redis的set操作指令，再以resp协议写到新的aof文件，在重写期间，如果有修改指令，会被存到aof重写缓冲区中（要注意的是，原来aof持久化里面，修改的指令也会照常写到aof缓存区，两个缓冲区不一样的，aof持久化那边也是照常执行的）。子进程重写完以后，会通知父进程，父进程会把aof缓冲区的内存追加到aof文件，并把新文件覆盖原来的aof文件。就算重写中途失败了，新的指令还是正常追加到原来的aof文件，不会导致数据丢失。
  - 默认是文件大小增长100%，且大于64M才触发重写。

- 混合持久化
  - 其实他本质上也是aof持久化，只不过是重写的时候，把重写这一时刻内存所有的数据转换成rdb格式写到一个新的aof文件里面，在这一时刻之后所有的新增修改指令就还是正常按照aof格式追加在文件后面，文件写成功了再覆盖原来的aof文件。数据恢复的时候，就可以先加载前面的rdb数据，再加载后面的aof数据，恢复效率高很多。
  
<div align=center><img src="https://user-images.githubusercontent.com/27798171/176377625-a7b598a0-1093-4644-bbba-ad1195f98746.png"/></div>

# redis做消息队列

- 普通消息队列：redis可以用list的lpush和rpop做消息队列，可以用brpop做阻塞队列。缺点是不能查历史记录的mq。

```shell
(1)发送一条消息 / 将一个信息插入到列表头部
lpush mymq message1
(2)获取一条消息 / 移除并返回列表的最后一个元素
rpop mymq
(3)移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
bpop mymq 1000
```

- 发布订阅模式：redis可以用发布订阅模式实现一个客户端发布订阅消息，多个客户端接收接收订阅消息。缺点是消息无法持久化，一旦出现网络问题，或者宕机，消息就没了。

```shell
接收消息的redis客户端：
(1) 订阅给一个频道的信息。
SUBSCRIBE mymq

发送消息的redis客户端
(2) 将信息发送到指定的频道。
PUBLISH mymq "Redis PUBLISH test"
```

-  延时队列：redis可以用zset，拿时间戳作为score，消息内容作为member调用zadd来生产消息，消费者用zrangebyscore指令获取N秒之前的数据轮询进行处理。

```shell
(1)添加一条消息, 添加时间为20220628111008 / 向有序集合添加一个或多个成员，或者更新已存在成员的分数
ZADD mymq 20220628111008 message1
(2)获取一分钟内的消息,当前时间是添加时间为20220628111100 / 通过分数返回有序集合指定区间内的成员
ZRANGEBYSCORE mymq 20220628111000 20220628111100 [WITHSCORES] [LIMIT]
```

- redis stream：
  - 消息可以持久化，可以访问任何时刻的数据。支持消费者独立消费或者消费组消费。
  - 独立消费就是比如用阻塞方式获取一条消息时，这条消息被一个客户端拿到了，另外一个客户端还用阻塞方式获取就拿不到了。
  - 而消费组消费，维护了每个消费组当前消费的位置，新来一条消息，每个消费组都能消费，但是一条消息只能被消费组里面的一个消费者消费。
  - 当消费者从消费组获取到消息的时候，会先把消息添加到自己消费组的pending列表，每个消费组都有维护一个pending列表，当消费者消费成功返回确认的时候，就会把这条消息从pending列表删除。
  - 如果消费者没有及时返回确认，就会导致这条消息占用2倍内存，比较浪费内存。

```shell
一、生产者：
(1) 发送一条消息到队列中，如果指定的队列不存在，则创建一个队列, id为*代表自增，返回值为id
XADD key ID field value [field value ...]
XADD mymq * name xiaohua  

(2) 删除一条消息，其实只是逻辑删除
XDEL key ID [ID ...]
XDEL mymq 1538561700640-0

二、消费者的独立消费（在队列没有变更的情况，命令有幂等性）：
(1) 获取消息列表，会自动过滤已经删除的消息。start为-代表最小值，end为+代表最大值
XRANGE key start end [COUNT count]
获取所有消息: xrange mymq - +
获取某个id之后的消息: xrange mymq 1538561700640-0 +

(2)以阻塞或非阻塞方式获取消息列表
XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] id [id ...]
从头部读取两条消息，非阻塞模式，读不到就返回空: XREAD COUNT 2 STREAMS mymq 0-0
读取1条大于某个id的消息，非阻塞模式，读不到就返回空: XREAD COUNT 1 STREAMS mymq 1538561700640-0
从尾部读取一条新消息，阻塞模式，一直阻塞到有消息: XREAD BLOCK 0 STREAMS mymq $

三、消费者的消费组消费（在队列没有变更的情况，命令没有幂等性，消费组游标会一直往前）：
(1) 创建消费者组
XGROUP [CREATE key groupname id-or-$] [SETID key groupname id-or-$] [DESTROY key groupname] [DELCONSUMER key groupname consumername]
从头开始消费：XGROUP CREATE mymq my-consumer-group-1 0-0  
从尾开始消费，只接受新消息：XGROUP CREATE mymq my-consumer-group-1 $

(2) 读取消费组中的消息，id为>代表只把没有分发给别人的消息发给我
XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] ID [ID ...]
获取一条消息，非阻塞模式：XREADGROUP GROUP my-consumer-group-1 随便消费者名字1号 COUNT 1 STREAMS mymq >
获取一条消息，阻塞模式：XREADGROUP GROUP my-consumer-group-1 随便消费者名字2号 BLOCK 0 STREAMS mymq >

(3) 确认消息
XACK mymq my-consumer-group-1 1605524648266-0  

(4) 获取消费组或消费者里面还没确认的消息。count最大为10
XPENDING key group [[IDLE min-idle-time] start end count [consumer]]
获取消费组还没确认的消息信息，返回还没确认的总数量、最小消息id、最大消息id、列表[消费者名字、对应的待确认数量]：XPENDING mymq  my-consumer-group-1
获取消费者还没确认的消息，返回列表[消息id、所属消费者、消息被消费后到现在的时间、消息被获取的次数]：XPENDING mymq  my-consumer-group-1 0 + 10 随便消费者名字1号

(5) 转移消息归属权。min-idle-time 表示最小空闲时间，只有后续指定ID的消息空闲时间大于指定的空闲时间，消息归属权转移指令才会生效。
XCLAIM key group consumer min-idle-time ID [ID …] [IDLE ms] [TIME ms-unix-time] [RET]
把某个id为XX的消息归到消费者2号下，XCLAIM mymq my-consumer-group-1  随便消费者名字2号 10000 1605524648266-0 

```
# 缓存雪崩、缓存穿透、缓存击穿

- 缓存穿透：
  - 描述：访问一个缓存和数据库都没有的数据，每次请求打到数据库。
  - 解决前提：请求参数校验要做好，比如校验住id传-1的情况
  - 解决办法一：空值存储这个key到缓存。但如果用户每次请求不同的key，会导致缓存很多垃圾数据。
  - 解决办法二：布隆过滤器。（展开描述参考[布隆过滤器](#bloomFilter)）
- 缓存雪崩：
  - 描述：大面积的缓存失效，大量请求直接打到DB
  - 解决办法一：设置不同的失效时间
- 缓存击穿：
  - 描述：一个Key非常热点，突然失效时，打崩了DB
  - 解决办法一：设置热点数据永远不过期或者活动期间不过期，或者加上互斥锁。

# 布隆过滤器
<a id="bloomFilter"/>

备注：一般都会在回答缓存穿透、或者海量数据去重、或者讲bitmap时引出来

- 布隆过滤器的底层就是bitmap，bitmap底层就是string，用bit位的0和1来代表传的值存不存在。
- 布隆过滤器的原理就是把你要找的值经过几个哈希函数得到几个哈希值，根据哈希值拿bitmap对应下标的值，如果都为1则代表存在。
- 布隆过滤器的缺点就是有概率把不存在的误判成存在，因为哈希冲突问题，可以通过一开始控制布隆过滤器的大小，来降低误判率。另外，如果判断成不存在，就一定不存在。而且存过的值不能删除，也是因为哈希冲突问题避免误删了别的值的。
- 布隆过滤器的优点就是由于是用bit存储一个数据，占用内存非常小，string最大512M，换算成bit有40多亿，能存储的数量非常大。

# redis的主从架构

- redis主从复制流程
  - 如果是第一次进行主从同步，从节点会发个psync同步命令，主节点就执行bgsave命令来生成最新的rdb文件，持久化期间，新进来的修改指令会缓存到内存里。持久化完以后，主节点就会把rdb文件发给从节点，从节点就会清空掉自己的所有数据并加载收到的rdb数据到内存里。然后从节点会通知主节点将持久化期间放内存的新数据发送给从节点。后续的修改新增命令，就是来一条就异步发送给从节点。
  - 如果master收到了多个从节点的psync同步命令，主节点只会生成一次rdb文件，把这一份rdb文件发给多个从节点。
  - 如果中途从节点挂了，主节点会在内存创建一个缓存队列，缓存最近一段时间的数据，由于从节点有维护已同步数据的偏移位置和主节点的进程id，所以从节点可以发送带偏移位置的psync同步命令给主节点，主节点收到了就将缓存里面的数据从偏移量开始之后发给从节点，如果找不到偏移位置，或者主节点重启过导致进程id变了就直接全量同步给从节点。
  - 主节点挂了，需要人为指定一个节点为主节点，再修改其他从节点的主节点，而且客户端还得修改master节点地址，所以在数据量少的情况下，最好用哨兵模式。

- 主从复制风暴问题：
  - 如果从节点太多，会导致主节点压力太多，要维持太多长连接，还得必传维持心跳，网络带宽压力也会比较大，所以可以考虑让主节点同步给部分从节点以后，让从节点与剩余从节点之间同步。

```
(1) 从节点的redis.conf里面配置主节点
replicaof 主节点ip 端口
```
<div align=center>
<img src="https://user-images.githubusercontent.com/27798171/176372466-b21026c8-6d98-4780-b7a3-47d83a30eb8b.png"/>
<img src="https://user-images.githubusercontent.com/27798171/176372527-c77691bb-d9ce-4f72-92c1-b2880975d26e.png"/>
<img src="https://user-images.githubusercontent.com/27798171/176372584-9e0eab65-75f9-4528-bb62-e74b27554aa6.png"/>
</div>

# redis的哨兵模式

- 哨兵模式描述：客户端只配置所有哨兵的地址，哨兵只配置主节点的地址，不过哨兵能拿到所有从节点信息。客户端一开始连接到哨兵，从哨兵拿到redis的主节点以后，后续就直接访问redis的主节点了。当超过一半的哨兵发现主节点挂了，(也不一定是一半，这个可以再redis.conf配置)，哨兵之间就会发起投票选举出一个哨兵leader，由它负责故障转移，重新选举出redis主节点，然后通过发布订阅模式让其他哨兵把自己监控的从节点切换成新主节点的从节点。另外，哨兵还会通知客户端新的主节点，这样客户端就可以动态切换主节点地址。等原来的主节点重新上线，哨兵会自动把它设置为从节点。
- 哨兵模式只有一个redis主节点对外提供服务，没法支持很高的并发，且单个主节点内存总是有限而且也不能设置太大，不然持久化文件太大，影响主从同步效率，恢复数据也会太久，所以想要支持大数据量和高并发最好用集群模式。

```
(1) 配置哨兵的sentinel.conf的主节点，后面的3代表多少个sentinel认为master失效就才算失效，一般等于sentinel总数/2 + 1
sentinel monitor 随便起个master名字 主节点ip 端口  2
```

<div align=center>
<img src="https://user-images.githubusercontent.com/27798171/176372993-65297293-3119-4f7b-b9e8-2fae60048d41.png"/>
</div>

# redis的集群模式

- 集群模式描述：redis集群是由多个主从节点群组成的，不需要哨兵也能完成节点移除和故障转移，利用哈希槽实现数据分片存储。
- reids把内存地址分成16384个哈希槽，每个节点负责一部分槽位，key通过crc16算法后再对16384取模来确定放哪个槽。每个节点都会存储所有节点的槽位信息，客户端来连接集群的时候，也会把槽位信息存储到客户端本地。如果集群节点有新增或缩减，两边槽位信息不一致的时候，比如客户端向一个节点发送的key的槽位不在这个节点，该节点发现后会向客户端发送一个携带正确节点的跳转地址，然后客户端就去往正确的节点发送指令，并纠正客户端本地的槽位信息缓存。
- 当集群里面的其他主节点ping某个主节时，超过半数以上通讯超时，就认为这个主节点挂了，并自动启用它的从节点。如果他没有从节点可以用了，默认配置是整个集群都不能再提供服务了，当然也可以配置成部分key可用。
- Redis集群节点间的通信机制: redis cluster节点间采取gossip协议进行通信。维护集群的元数据(集群节点信息，主从角色，节点数量，各节点共享的数据等)有两种方式：集中
式和gossip
  - 集中式：将集群元数据集中存储在某个节点上；优点：元数据的更新和读取，会立即更新到集中式的存储节点，其他节点可以读取的时候可以立即感知到。缺点：所有的元数据的更新压力全部集中在一个地方。（很多中间件都会借助zookeeper集中式存储元数据。）
  - gossip：每个节点都会向其他节点发送信息，包括自己的状态和其他节点的状态，每个节点都会存储集群的元数据。优点：更新请求会陆陆续续打到所有节点上去更新，有一定的延时，降低了压力。缺点：元数据更新有延时可能导致集群的一些操作会有一些滞后。
     - gossip协议包含多种消息，包括ping，pong，meet，fail等等。
     - meet：某个节点发送meet给新加入的节点，让新节点加入集群中，然后新节点就会开始与其他节点进行通信；
     - ping：每个节点都会频繁给其他节点发送ping，其中包含自己的状态还有自己维护的集群元数据，互相通过ping交换元数据(类似自己感知到的集群节点增加和移除，hash slot信息等)；其他节点接收到ping消息之后返回pong消息
     - pong: 对ping和meet消息的返回，包含自己的状态和其他信息，也可以用于信息广播和更新；
     - fail: 某个节点判断另一个节点fail之后，就发送fail给其他节点，通知其他节点，指定的节点宕机了。

### Redis集群选举原理分析
- 当有主节点故障时，1个或多个从节点会将该节点标记为不可用，并向集群中的其他节点广播该信息（FAILOVER_AUTH_REQUEST 信息，含有epoch、槽位信息等信息），请求其他主节点进行投票。其他主节点收到该信息后，会验证认证信息和槽位信息（是否与主节点最后一条心跳中携带的槽位数据一致），验证通过就返回从节点支持他成为新主节点的消息（FAILOVER_AUTH_ACK消息，对每一个epoch只发送一次ack），并在一定时间内不再给其他的从节点投票。从节点收到FAILOVER_AUTH_ACK消息之后，会统计收到了多少个主节点的支持，如果超过N/2+1个支持，则该从节点会当选为新的主节点。否则根据投票数、数据完整性、槽位信息、节点id大小等因素，选出最合适的节点作为主节点。
- 新主节点会广播Pong消息通知其他集群节点，告诉主节点故障并由自己接管了故障节点的槽信息。
- 从节点并不是在主节点一进入 FAIL 状态就马上尝试发起选举，而是有一定延迟，一定的延迟确保我们等待FAIL状态在集群中传播，slave如果立即尝试选举，其它masters或许尚未意识到FAIL状态，可能会拒绝投票
  - 延迟计算公式：DELAY = 500ms + random(0 ~ 500ms) + SLAVE_RANK * 1000ms。SLAVE_RANK表示此slave已经从master复制数据的总量的rank。Rank越小代表已复制的数据越新。这种方式下，持有最新数据的slave将会首先发起选举（理论上）
### 集群为什么master至少三个master节点，并且推荐节点数为奇数
- 因为只有当半数以上的主节点觉得某个主节点挂了，才会重新选举新的主节点。如果只有两个主节点，一个主节点挂了，剩下的一个主节点怎么投票都没办法超过1的投票量。
- 奇数个主节点相比偶数个节点在相同效果下可以节省一个节点，比如三个节点和四个节点相比，挂了一个主节点，三个节点的投票数可以超过1，四个节点的投票数可以超过2，都能选举。但是如果是挂了2个节点，三个节点的投票数没办法超过1，四个节点的投票数没办法超过2，都不能选举，那一样的效果，何不直接节省一个节点，直接奇数个节点就够了。

- 集群模式下批量操作的支持：对于mset,mget命令默认只支持操作所有key的都落在同一个槽。如果非要支持的话，可以在key前面加上{固定前缀}，这样计算槽位的时候，只会拿括号里面的值，从而确保不同的key落到同一个槽。
```
（1） 集群模式下，批量操作多个key，保证落到同个槽
mset {user1}:1:name xiaoming {user1}:1:age 18
```

![image](https://user-images.githubusercontent.com/27798171/177273509-bd88eedc-3168-4b78-9224-9bd6468937a6.png)

```shell
集群模式怎么搭建：

(1) redis.conf文件配置成集群模式
cluster-enabled yes (启动集群模式)
port 当前reids节点端口 （机器端口号设置）
pidfile /var/run/redis_当前reids节点端口.pid (把pid进程好写入pidfile配置的文件)
cluster-config-file nodes-当前reids节点端口.conf （集群节点信息文件）
dir /usr/local/redis-cluster/当前reids节点端口/ (指定数据文件存放位置)

(2)分别启动所有主从redis实例
/usr/local/redis-5.0.3/src/redis-server redis.conf

(3) 用redis-cli创建集群，这里的1代表为每个主节点分配一个从节点，三主三从
/usr/local/redis-5.0.3/src/redis-cli -a 你的密码 --cluster create  --cluster-replicas 1 节点1号:端口 节点2号:端口 节点3号:端口 节点4号:端口 节点5号:端口 节点6号:端口

(4) 查看集群
先随便找个节点进来，-c代表集群模式： /usr/local/reids-5.0.3/src/redis-cli -a 你的密码 -c -h 节点1号 -p 端口
查看集群信息，返回集群节点数和当前状态等等: cluster info
查看节点信息，返回主从关系，分配的槽号范围等等: cluster nodes

(5) 关闭集群需要逐个节点关闭
/usr/local/redis-5.0.3/src/redis-cli -a 你的密码 -c -h 节点1号 -p 端口 shutdown

```

```
集群模式怎么水平扩展：

(1) 写好新主从节点的redis.conf文件，和上面一样
(2) 启动新主从节点的redis实例，和上面一样
(3) 给集群新增一个主节点
/usr/local/redis‐5.0.3/src/redis‐cli ‐a user ‐‐cluster add‐node 新增主节点:端口 已经存在的一个主节点:端口
(4) 重新分配hash槽
/usr/local/redis‐5.0.3/src/redis‐cli ‐a user ‐‐cluster reshard 已经存在的一个主节点:端口
然后会询问分配多少个槽移动到新的节点里？600
然后会询问从哪些主节点分一些槽出来凑成600个槽？all（代表从所有已存在的主节点抽部分槽，凑成600个槽）
(5) 添加新从节点到集群中去，并配置为从节点
加到集群里：/usr/local/redis‐5.0.3/src/redis‐cli ‐a user ‐‐cluster add‐node 新增从节点:端口 已经存在的一个主节点:端口
用新从节点访问集群：/usr/local/redis‐5.0.3/src/redis‐cli ‐a user ‐c ‐h 新从节点ip ‐p 端口 
在会话里指定作为谁的从节点 > cluster replicate 主节点id 

(6) 删除一个从节点
/usr/local/redis‐5.0.3/src/redis‐cli ‐a user ‐‐cluster del‐node 要删的从节点ip:端口 要删的从节点id
(7) 删除主节点
先把槽分配给的主节点：/usr/local/redis‐5.0.3/src/redis‐cli ‐a user ‐‐cluster reshard 要删的主节点ip:端口
删除主节点：/usr/local/redis‐5.0.3/src/redis‐cli ‐a user ‐‐cluster del‐node 要删的主节点ip:端口 要删的主节点id

```

```java
(1) 使用jedisCluster来连接集群

public class JedisClusterTest {
    public static void main(String[] args) throws IOException {

        JedisPoolConfig config = new JedisPoolConfig();
        config.setMaxTotal(20);
        config.setMaxIdle(10);
        config.setMinIdle(5);

        Set<HostAndPort> jedisClusterNode = new HashSet<HostAndPort>();
        jedisClusterNode.add(new HostAndPort("192.168.0.61", 8001));
        jedisClusterNode.add(new HostAndPort("192.168.0.62", 8002));
        jedisClusterNode.add(new HostAndPort("192.168.0.63", 8003));
        jedisClusterNode.add(new HostAndPort("192.168.0.61", 8004));
        jedisClusterNode.add(new HostAndPort("192.168.0.62", 8005));
        jedisClusterNode.add(new HostAndPort("192.168.0.63", 8006));

        JedisCluster jedisCluster = null;
        try {
            //connectionTimeout：指的是连接一个url的连接等待时间
            //soTimeout：指的是连接上一个url，获取response的返回等待时间
            jedisCluster = new JedisCluster(jedisClusterNode, 6000, 5000, 10, "zhuge", config);
            System.out.println(jedisCluster.set("cluster", "zhuge"));
            System.out.println(jedisCluster.get("cluster"));
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (jedisCluster != null)
                jedisCluster.close();
        }
    }
}

运行效果如下：
OK
zhuge
```

# redis脑裂数据丢失问题
redis哨兵模式或集群模式没有过半机制就会有脑裂问题，脑裂问题就是某一个主节点因为网络问题或者是正在处理大量客户端请求比较繁忙导致心跳回复失败，于是触发了主从切换机制选了新的主节点。在切换过程中，原来的主节点还在和部分客户端通讯，导致两个主节点同时在处理请求。危害就是比如：setNX可能会拿到相同的锁；基于incr生成的全局唯一id，也可能出现重复。当原来的主节点的心跳恢复，发现已经有新的主节点了，就会清空本地数据，开始全量同步新master数据，导致在双主的过程中，向原来主节点请求的的数据没了。所以最好设置写数据的时候，要同步一半的从节点才算写成功。不过这样会影响集群的可用性和性能，一般也不会配置。
![image](https://user-images.githubusercontent.com/27798171/177271543-fe12cd34-b888-4d10-ac2a-bdae22f94f2f.png)

```
（1） 比如主从三个节点，主节点必须同步至少1个从节点
in-slaves-to-write 1
```

# redis的分布式锁
- redis的分布式锁描述：用setNx key value这个命令来申请加锁，加锁成功返回1，否则返回0。用del key来释放锁。【但如果程序处理异常或者挂了导致没释放锁，客户端就一直拿不到锁，导致死锁】。
```
127.0.0.1:6379> SETNX myLock 1
(integer) 1     // 返回1加锁成功，返回0加锁失败
127.0.0.1:6379> DEL myLock // 释放锁
(integer) 1
```
- 怎么避免死锁？可以给key设置超时时间。【但这样可能会导致A加的锁，A业务处理太久，锁过期了，被B抢到了锁，导致两个客户端都抢到了锁，等A处理完，又把B的锁给释放了。】
```
127.0.0.1:6379> SET myLock 1 EX 60 NX
OK
```
- 怎么避免锁被别人释放？可以在加锁的时候，value存唯一id，删除key的时候如果value是这个唯一id才可以删。【但查询和删除是两个指令，需要保证原子执行，可以用lua脚本处理】
- 怎么避免还没处理完锁就过期的问题？可以用看门狗，就是开启一个守护线程，定时去检测这个锁的失效时间，如果锁快要过期了，操作共享资源还未完成，那么就自动对锁进行「续期」，重新设置过期时间。redisson已经封装好了。
- 说一下redission的分布式锁怎么实现的？首先，redission会存储成hash类型到redis，key就是你要抢占的key，field为guid+当前线程的ID，value就是重入锁的次数。所以抢锁的时候，是通过执行一段包含是否不存在key然后hset和expire的lua脚本，抢不到锁就while循环尝试获取，抢到了锁就同时还会开启守护线程看门狗。redission其实支持可重入锁，抢锁的时候，执行的lua脚本里面还包含一个判断就是，如果已经存在这个key了，就把value+1，释放锁的时候就逐个-1，减到为0就删除锁。

![image](https://user-images.githubusercontent.com/27798171/177285636-35a1e1a6-6357-4f5d-8815-1ef2e80d718d.png)

![image](https://user-images.githubusercontent.com/27798171/177285678-4ca851de-7fe7-4b81-aec3-9f3ab037f4c6.png)

``` java
// 1.获取锁
RLock lock = redisson.getLock("myLock");

// 2.尝试加锁，且实现了锁续命，而且可重入，配置60s后过期，只尝试30s，没抢到内部会while循环尝试
lock.lock(30,60, TimeUnit.SECONDS);

try {
   //3. 执行业务代码
} finally {
   //4.解锁
   lock.unlock();
}

```
- 在哨兵或者集群模式下，如果主节点挂了，没来得及同步给从节点，导致多个客户端抢到了锁怎么办？可以用redlock，但这个一般不太会有人用，因为官方推荐说是得一个主库至少得部署5个实例，你得在每个实例上都申请加锁，如果大于等于3个实例加锁成功才算真的加锁成功，释放锁的时候，得向5个实例都发起请求。

# 缓存一致性问题


# redis为什么快？

- 基于内存实现。
  - 读写速度比从磁盘读取快，因为从磁盘读取还得寻道、旋转、磁盘读取、再加载到内存、再从内存读取。
- 单线程，在一定的请求量以内（预估百万级别），多线程不一定就快过单线程，因为单线程可以避免上下文切换。使用多路复用IO模型。
  - 啥是上下文切换？CPU会给每个线程分配时间片来执行，CPU为了执行多个线程，就需要不停的切换执行的线程，一个任务执行一个时间片后切换到下一个任务。在切换前会保存上一个任务的状态，以便下次切换回这个任务时，可以再加载这个任务的状态。所以任务从保存到再加载的过程就是一次上下文切换。
  - 啥是多路复用IO模型？参考[redis的线程模型](#redisIoModel)
- 简单的自定义resp协议。
  - 这个协议的实现性能仍然可以和二进制协议一样快。因为 Redis 协议将数据的长度放在数据正文之前，所以程序无须像JSON那样，为了寻找某个特殊字符而扫描整个payload， 也无须对发送至服务器的payload进行转义。程序可以在对协议文本中的各个字符进行处理的同时，查找换行字符，并计算出批量回复或多条批量回复的长度。
- C语言实现，效率高
- 数据结构简单，操作快

# redis的线程模型
<a id="redisIoModel"/>

- Redis内部使用文件事件处理器，这个文件事件处理器是单线程的，所以Redis才叫做单线程的模型。
- 文件事件处理器的结构包含 4 个部分:（1）多个socket （2）IO多路复用程序 (3)文件事件分派器。（4）事件处理器(连接应答处理器、命令请求处理器、命令回复处理器等)。
- 客户端的socket向redis的server socket请求连接或者发送指令，serverSocket就会产生对应的事件。IO多路复用程序负责监听多个server socket，会将socket产生的事件放入队列中排队，事件分派器每次从队列中取出一个事件，把该事件交给对应的事件处理器进行处理。
     
![image](https://user-images.githubusercontent.com/27798171/180151302-d139c814-0762-45fc-b088-00e2fb45a153.png)

# redis管道、事务、lua脚本
 
- 管道
  - 管道是redis客户端提供的技术，和服务端没关系。
  - 客户端会一次性发送多个命令到内核的缓冲区，由操作系统把所有命令发送给服务端，服务端处理完会把所有处理结果缓存起来，都处理完了再一次性把所有处理结果返回给客户端。
  - 这样就可以节省多次连接、发送、返回的时间，只需要一次就好。
  - pipeline不是原子性的，前面的命令中途就算有失败的，也不会影响后面的命令。
  - 不过pipeline要求执行的指令间没有因果关系。。

<div align=center><img src="https://user-images.githubusercontent.com/27798171/176371687-1cb89df1-94d7-41ad-8d82-21c79dcc5ea8.png"/></div>
<div align=center><img src="https://user-images.githubusercontent.com/27798171/176405003-33bde355-554b-4082-bd6f-c0d207ec2e30.png"/></div>

```shell
(1) redis客户端利用pipeline发送指令。首先使用 PING 命令检查 Redis 是否正常工作，然后又分别使用了 SET/GET/INCR 命令，以及 sleep 阻塞 2 秒，最后将这些命令一次性的提交给 Redis 服务器，Redis 服务器在阻塞了 2 秒之后，一次性输出了所有命令的响应信息。每个命令字符串必须以 \r\n 结尾。至于语句最后的 nc localhost 6379 是固定格式无需更改。

(echo -en "PING\r\n SET name xiaohua\r\n GET name\r\n INCR num\r\n INCR num\r\n INCR num\r\n"; sleep 2)|nc localhost 6379
```


```java
(2) jedis利用pipeline发送指令

//得到关联jedis客户端的pipeline
Pipeline pl = jedis.pipelined();

//pipeline通过关联的jedis客户端向服务器发送请求 
pl.incr("num");
pl.set("name","xiaohua");

//pipeline向服务端请求返回值
List<Object> results = p.syncAndReturn();
```

- redis事务
  - redis的事务不支持回滚，其中一个命令失败了，也不会影响其他命令。
  - 用mutil开启事务，后续发送的命令放到服务端队列里面，调用exec执行所有命令，执行到exec会阻塞其他命令，调用discard清空队列的命令。
  - 有个watch命令可以监听指定的key，如果你执行的事务里面要修改这个key，就算调用了exec整个事务也不会执行。
  - 可以保证一致性和隔离性，因为事务前后数据是保持一致的，另外redis毕竟是单线程处理命令的，所以事务之间是隔离的，但不能保证持久性和原子性，因为不管是rdb和aof持久化都有概率宕机的时候丢失数据，没有原子性是因为不支持出错回滚，但能保证命令都执行过或都不执行。

```shell
(1)执行事务
> MULTI
OK
> SET USER "Guide哥"
QUEUED
> GET USER
QUEUED
> EXEC
1) OK
2) "Guide哥"

(2)取消事务
> MULTI
OK
> SET USER "Guide哥"
QUEUED
> GET USER
QUEUED
> DISCARD
OK

(3)监听某个key
> WATCH USER
OK
> MULTI
> SET USER "Guide哥"
OK
> GET USER
Guide哥
> EXEC
ERR EXEC without MULTI

```

- lua脚本
  - redis可以用EVAL命令来执行lua脚本，lua脚本可以支持原子性地执行多条命令，而且可以写一些计算和判断逻辑。
  - 相比执行多条命令，lua脚本只需要一次发送和一次返回。

```
(1)执行lua脚本
EVAL script numkeys key [key ...] arg [arg ...]
EVAL "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
```


| 对比 | pipeline | redis事务 | lua脚本 |
| ---- | ---- | ---- | ---- |
| 命令缓冲方式 | 命令缓冲在客户端本地内存中 | 命令缓冲在服务端内存队列 | 命令缓冲在服务端内存队列 |
| 请求次数 | 只需要发一次到服务端 | 每个命令都需要发送到服务端 | 只需要发一次到服务端 |
| 原子性 | 否 | 否 | 是 |





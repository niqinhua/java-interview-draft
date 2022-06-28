
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

```
(1) 单值缓存
set k v
(2)对象缓存
set user:1 json
mset user:1:name zhuge user:1:age 18
mget user:1:name user:1:age
(3)分布式锁
setNx k v
set k v ex 10 nx
(4)计数器或自增器
incr k
``` 

### hash

- 可以存储对象、也可以用于计数器。
- 方便数据整合存储。
- 相比string的操作，消耗内存与cpu更小，储存更节省空间。
- 底层存储结构是哈希表，数据少且短小时也能用压缩列表存储

```
(1)对象缓存
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

```
(1) 存储key
lpush k v [v..]
rpush k v [v..]
(2)拿key
lpop key 移除并返回
rpop key
lrange key start stop 返回指定区间内的元素，索引从0开始
blpop key [key..] timeout  左边弹出一个元素，若列表没有就阻塞等待
brpop key [key..] timeout  右边弹出一个元素，若列表没有就阻塞等待
(3)statck
栈=lpush+lpop=FIFO
(4)queue
队列=lpush+rpop
(5)blocking mq
阻塞队列=lpush+brpop
(6)微博和微信公众号的消息列表  （这种只能用于粉丝数少的）
比如A用户关注了B、C
B发消息：lpush msg:A msgId1
C发消息：lpush msg:A msgId2
A查看消息：lrange msg:A 0 10
```

### set

- 可以存储集合、也可以用于抽奖
- 底层存储结构是哈希表，或者是整数集合

```
(1) 存储key
SADD key member [member..] 插入key，元素存在则忽略
(2)删除元素
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
(5)抽奖场景
参与抽奖：sadd key user1
查看抽奖的所有用户：smemebers key
抽取count名中奖者 : spop key [count] 选出count 个元素，元素删除
(6)朋友圈点赞收藏标签
点赞 sadd like:message1 user1
取消点赞: srem like:message1 user1
检查用户是否点赞过   sismember like:message1 user1
获取点赞的用户列表 smembers like:message1
获取点赞的用户数 scard like:message1 
(7)互相关注
 A和B共同关注的人：sinter aSet  bSet
我(A)关注的人(C、D)也关注他们: sismember C  X ; sismemeber D X; 
我可能认识的人：sdiff A C 
(8) 商品的多条件筛选
sinter 安卓set 8GBset 高清set 
```
 
### zet

- 可以用于有序集合、排行榜
- 底层存储结构是跳表，数据少且短小时也能用压缩列表存储

```
(1)删除
zrem key [member..] 删除一个元素
(2)查询
zscore key memeber 返回key中某个元素的分值
zrange key start stop [withscores] 正序获取key范围内的元素
zrevrange key start stop  [withscores] 逆序获取key范围内的元素
(4)修改、插入
zadd key score member [[score member]..] 往key中加入带分值的元素
zincrby key 数值 member 为key里面的元素的分值++数值
(5)集合操作
zunionstore newkey 后面key的数量 key [key..]  并集计算
zinterstore newkey 后面key的数量 key [key..]  交集计算
(6)排行榜
点击一次新增加一次热度： zincrby new:id1 1 标题
今天热度排行前十：zrevrange new:天:20220202 0 9 withscores
七日热度排行榜： zuionstore new:新的key:最近天 7 new:天:20220201 new:天:20220202 一直到 new:天:20220207 
七日排行前十：zrevrange  new:新的key:最近天  0 9 withscores
```

### hyperLogLog (特殊类型)

- 用来基数统计的，有点类似set，但是这个类型占用的内存空间小很多。
- 缺点是只能告诉某个元素有没有存在过，不能把所有元素拿出来，统计基数总个数也只是估算的。（因为只会根据输入元素来计算基数，它并不存储元素本身）。
- 可以用于估算每日访问ip/用户数

```
(1)添加元素，计算成基数存储下来
PFADD runoobkey "redis"
(2)统计基数估算值
PFCOUNT runoobkey
(3)合并两个HyperLogLog到新的HyperLogLog
PFMERGE destkey sourcekey [sourcekey ...]
```

### geo (特殊类型)

-  主要用于存储地理位置的经度、维度、地名。类似集合。
-  使用geohash来保存地理位置的坐标

```
(1)添加一个地理位置到
GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
(2)获取地理位置的坐标
GEOPOS Sicily 地名1 地名2 
(3)计算两个位置之间的距离。
GEODIST Sicily Palermo Catania
(4)根据用户给定的经纬度坐标来获取指定范围内的地名+地理位置集合。
GEORADIUS Sicily 15 37 100 km
(5)用户给定的地名来获取指定范围内的地名集合。
GEORADIUSBYMEMBER Sicily Agrigento 100 km
(6)geohash 用于获取一个或多个位置元素的geohash值
GEOHASH Sicily 地名1 地名2 
```

### bitmap(特殊类型)

- bitmap实际上就是string类型，只是每位bit都是0和1，用来代表存储的元素存不存在。
- 布隆过滤器就是基于bitmap
- 可以用于用户行为统计

```
(1)001用户新增对userId为11的标记
setbit beauty_girl_001 11 1
(2)查询001用户对userId为11的标记
getbit beauty_girl_001 11
(3)统计被被设置为 1 的位的数量
bitcount beauty_girl_001
(4)BITOP 命令支持 AND 、 OR 、 NOT 、 XOR 这四种操作中的任意一种参数
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
  - bgsave的流程就是：redis会在主进程fork一个子进程，由子进程执行持久化，主进程正常处理客户端指令。子进程可以共享主进程的所有内存数据。子进程会把触发持久化那一时刻redis中所有的数据以二进制形式保存到rdb文件里面。如果持久化过程中有修改指令，就会把要修改的数据所在页赋值一份，子进程把拷贝的数据存到rdb文件，主进程就正常修改原来页的数据。
  - 但bgsave持久化时，有大量的修改操作，就会触发很多页的拷贝，占用的内存就会持续增长，导致内存不够用。
  - 另外，由于每次快照保存的是当时redis全量的数据，数据量大的话，会持久化好几分钟，一旦宕机，持久化过程中新增或修改的数据就没了，丢了好几分钟的数据。
  - 好处就是因为每次快照不同时刻的数据到rdb文件，可以利用rdb文件还原到不同时刻的版本。 而且由于是二进制存储，RDB在恢复大数据集时的速度比AOF的恢复速度要快，很适合于灾难恢复。

- aof
  - 把修改的每一条指令以resp协议追加到aof文件里面。
  - 主进程处理一条修改指令时，会先修改内存的这条数据。如果你配置的是always写盘策略，那主进程就还得再把这条指令追加到磁盘的aof文件里面。如果你配置的是每秒一次（EverySecond）的写盘策略，那主进程就还得再把这条指令写到aof内存缓冲区，然后redis后台会有一个线程专门，每隔一秒，把内存缓冲区内积累的指令同步到aof文件。还有一种no写盘策略，和每秒一次写盘策略类似，只不过没有后台线程去执行写入磁盘的操作，而是由操作系统来决定，何时刷新到磁盘。
  - 一般都是用每秒一次这种写盘策略，可以让主线程处理客户端指令时响应快些，因为直接写到缓冲区内存肯定比写到磁盘快。最多也就丢失一秒的数据。
  - 如果redis已经跑了一年，那这个aof文件会非常庞大，恢复数据会特别慢，比如一个key如果incr了好几千次，没必要记录几千次，只需要记录最后的值就好，所以需要对aof进行重写。但是就算重写了，速度也会比rdb慢很多，所以可以用混合持久化。

- aof重写
  - 原理就是主进程fork一个子进程，子进程扫描内存的所有数据，转换为Redis的操作指令，再以resp协议写到新的的aof文件，文件写成功了再覆盖原来的aof文件。如果重写过程中有修改指令，也是利用写时复制处理。就算重写中途失败了，新的指令还是正常追加到原来的aof文件，不会导致数据丢失。
  - 默认是文件大小增长100%，且大于64M才触发重写。

- 混合持久化
  - 其实他本质上也是aof持久化，只不过是重写的时候，把重写这一时刻内存所有的数据转换成rdb格式写到一个新的aof文件里面，在这一时刻之后所有的新增修改指令就还是正常按照aof格式追加在文件后面，文件写成功了再覆盖原来的aof文件。数据恢复的时候，就可以先加载前面的rdb数据，再加载后面的aof数据，恢复效率高很多。
  
# redis做消息队列

- 普通消息队列：redis可以用list的lpush和rpop做消息队列，可以用brpop做阻塞队列。缺点是不能查历史记录的mq。
```
(1)发送一条消息 / 将一个信息插入到列表头部
lpush mymq message1
(2)获取一条消息 / 移除并返回列表的最后一个元素
rpop mymq
(3)移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
bpop mymq 1000
```

- 发布订阅模式：redis可以用发布订阅模式实现一个客户端发布订阅消息，多个客户端接收接收订阅消息。缺点是消息无法持久化，一旦出现网络问题，或者宕机，消息就没了。
```
接收消息的redis客户端：
(1) 订阅给一个频道的信息。
SUBSCRIBE mymq

发送消息的redis客户端
(2) 将信息发送到指定的频道。
PUBLISH mymq "Redis PUBLISH test"
```

-  延时队列：redis可以用zset，拿时间戳作为score，消息内容作为member调用zadd来生产消息，消费者用zrangebyscore指令获取N秒之前的数据轮询进行处理。
```
(1)添加一条消息, 添加时间为20220628111008 / 向有序集合添加一个或多个成员，或者更新已存在成员的分数
ZADD mymq 20220628111008 message1
(2)获取一分钟内的消息,当前时间是添加时间为20220628111100 / 通过分数返回有序集合指定区间内的成员
ZRANGEBYSCORE mymq 20220628111000 20220628111100 [WITHSCORES] [LIMIT]
```

- redis stream：

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
  - 如果中途从节点挂了，主节点会在内存创建一个缓存队列，缓存最近一段时间的数据，由于从节点有维护已同步数据的偏移位置和主节点的进程id，所以从节点可以发送待偏移位置的psync同步命令给主节点，主节点收到了就将缓存里面的数据从偏移量开始之后发给从节点，如果找不到偏移位置，或者主节点重启过导致进程id变了就直接全量同步给从节点。

- 主从复制风暴问题：
  - 如果从节点太多，会导致主节点压力太多，要维持太多长连接，还得必传维持心跳，网络带宽压力也会比较大，所以可以考虑让主节点同步给部分从节点以后，让从节点与剩余从节点之间同步。

# redis的哨兵模式

- 哨兵模式描述：
- 哨兵leader选举流程：

# redis的集群模式

- 集群模式描述：
- 
- 集群模式和哨兵模式比较：
- 集群选举原理：
- 集群脑裂数据丢失问题：
- 集群为什么master至少三个master节点，并且推荐节点数为奇数？

# redis的分布式锁
# 缓存一致性问题
# redis为什么快？
# redis的线程模型
# 管道
# redis事务



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
删除商品 	hdel cart:user:1 sku1
获取购物车所有商品 hgetall cart:user:1
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
blpop key [key..] timeout  弹出一个元素，若列表没有就阻塞等待
brpop key [key..] timeout
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

- 可以存储集合、抽奖
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

# redis的数据结构
### sds
### 双向链表
### 压缩列表
### 哈希表
### 整数集合
### 跳表
# 模糊查找key

# redis过期数据的删除策略
# redis内存淘汰机制
# redis的持久化机制
# 缓存雪崩、缓存穿透、缓存击穿
# redis的哨兵模式
# redis的集群模式
# redis的分布式锁
# 缓存一致性问题
# redis为什么快？
# redis的线程模型

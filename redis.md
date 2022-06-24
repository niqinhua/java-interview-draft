
# Redis和Memcached的区别和共同点
- 这两个都是基于内存存储的。
- 最大的区别就是memcached重启完之前数据就没了，而redis有持久化机制，重启完数据还在。
- memcached不支持集群，而redis支持集群。
- memcached只支持k/v数据类型，redis还支持hash、list、set等。
- memcached是多线程、非阻塞IO复用的网络模型，而Redis是单线程的多路IO复用模型。
- memcached过期数据的删除策略只有惰性删除，而Redis有惰性删除与定期删除。
- Redis功能也强大多，支持发布订阅模型、Lua 脚本、事务、分布式锁等功能。
# redis常见数据类型
### string
- 可以存储常用的k/v，也可以用于分布式锁，或者计数器。
- 底层数据结构是sds。
```
(1). 单值缓存
set k v
(2). 对象缓存
set user:1 json
mset user:1:name zhuge user:1:age 18
mget user:1:name user:1:age
(3). 分布式锁
setNx k v
set k v ex 10 nx
(4).计数器或自增器
incr k
``` 
### hash
- 可以存储对象、也可以用于计数器。
- 方便数据整合存储。
- 相比string的操作，消耗内存与cpu更小，储存更节省空间
```

```
### list
- 可以存储列表、栈、队列
### set
- 
### zet
### hyperLogLog
### geo
### bitMap
# redis的数据结构
### sds
### 双向链表
### 压缩列表
### 哈希表
### 整数集合
### 跳表
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

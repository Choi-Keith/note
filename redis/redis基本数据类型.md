# Redis基本数据类型
-----------------------------


## 1. 字符串

### 1.1 类型介绍
字符串可以存储的值可以是字符串、正述或浮点数，它是采用预分配冗余空间的方式来减少内存的频繁分配，内部为当前字符串实际分配的空间capacity，一般高于实际字符串长度Len。当字符串长度小于1M时，扩容都是加倍现有的空间，如果超过1M, 扩容时一次只会多扩1M的空间。需要注意的时字符串的最大长度为512M

### 1.2 应用场景
主要应用于缓存数据、提高查询性能。比如用户登录信息、电商中商品信息、计数器等

### 1.3 操作指令

```shell
set key value
get key
mset key1 value1 key2 value2 ...
mget key1 key2
incr num
decr num
incrby num 2
decrby num 2
del num

```

## 散列


### 2.1 类型介绍

一个hash有多个key



### 2.2应用场景
Hash也可以存储用户登录信息，与字符串不同的是，字符串需要将对象序列化之后才能保存，而Hash则可以将用户对象的每个字段单独存储，这样就能节省序列化和反序列化的时间


### 2.3 操作指令

```shell
hset field key1 value1 key2 value2
hget field key1 
hmget field key1 key2
hgetall field
hlen field
hincrby field
del field
hdel field key
```

## 列表

### 3.1 类型介绍
List的实现原理是一个双向链表，即可以支持反向查找和遍历，插入和删除操作非常快，时间复杂度为O(1), 但是索引定位很慢O(n)

### 3.2 使用场景
List可以实现热销榜、最新列表(比如最新评论)


### 3.3 操作指令


```shell
lpush key value1 value2 value3...
lpop key
lrange key start end

rpush key value1 value2 value3
rpop key
```

## 集合

### 4.1 类型介绍
set的实现原理是通过计算hash的方式去重的，这也是set能提供判断一个成员是否在集合内的原因


### 4.2 应用场景
set支持集合内的增删改查，并且支持多个集合间的交集、并集和差集操作。可以利用这些集合操作，解决程序开发过程中很多数据结合中的问题。比如计算网站独立IP、用户画像中的用户标签、共同好友等功能


### 4.3 操作指令

```shell
sadd key value1 value2
smembers key
srem key value
sinter key1 key2 # 取交集
sdiff key1 key2 # 取差集
sunion key1 key2 # 取并集

```


## 有序集合

### 5.1 类型介绍
sorted sets内部使用一个hash和一个skipList(跳跃表)来保证数据的存储和有序，hash里放的是成员和score的映射， 而skipList里存放的是所有的成员，排序依据是hash里的score,使用跳跃表可以提高查询速率。当sorted sets最后一个value被移除后，数据结构自动删除被缓存和回收。


### 5.2 应用场景
主要应用场景是根据某个权重进行排行，比如游戏积分的排行、设置优先级的任务队列、学生成绩等。


### 5.3 操作指令
```shell
zadd key score value score value ...
zrange key start end
zrangebyscore key start end
zrem key value
zcard key
zcount key start end
zrank key value
zrevrank key value
```
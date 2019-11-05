# Redis

[TOC]

## 简介

REmote DIctionary Server(Redis) 是一个由Salvatore Sanfilippo写的key-value存储系统。

Redis是一个开源的使用ANSI C语言编写、遵守BSD协议、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。

### 特点

1. 基于内存（速度快），也支持数据持久化
2. 非关系型数据库
3. 支持多种数据结构

## 数据类型

|        数据类型         |                          存储的值                           |
| :---------------------: | :---------------------------------------------------------: |
|      String 字符串      |           可以是字符串，整数，浮点数，统称为元素            |
|        List 列表        |                          元素列表                           |
|        Hash 散列        |                       key-value键值对                       |
|        Set 集合         |                       各不相同的元素                        |
| Sort Set(ZSet) 有序集合 | 带分数的sorce-value有序集合，其中sorce为浮点数，value为元素 |

### 字符串

最基本的数据类型，能存储任意形式的字符串，包括二进制数据  

1. 赋值和取值  
    `SET key value`  
    `GET value`  
2. 递增数字  
    `INCR key`  
    当需要操作的键不存在时默认键值为0，所以第一次递增后结果为1.当键值不是整数时Redis会提示错误。
3. 增加指定的整数  
    `INCRBY key increment`  
4. 减少指定的整数  
    `DECR key`  
    `DECRBY key decrement`
5. 增加指定浮点数  
    `INCRBYFLOAT key increment`
6. 向尾部追加值  
    `APPEND key value`
7. 获取字符串长度  
    `STRLEN key`
8. 同时获得/设置多个键值  
    `MGET key [key ...]`  
    `MSET key value [key value ...]`
9. 位操作  
    `GETBIT key offset`  
    `SETBIT key offset value`  
    `BITCOUNT key [start] [end]`  
    `BITOP operation  destkey key [key ...]`

### 散列

散列类型适合存储对象：使用对象类别和id构成键名，使用字段表示对象的属性，而字段值则存储属性值  

1. 赋值和取值  
    `HSET key field value`  
    `HGET key field`  
    `HMSET key field value [field value ...]`  
    `HMGET key field [field ...]`  
    `HGETALL key`  
2. 判断字段是否存在  
    `HEXISTS key field`
3. 当字段不存在时赋值  
    `HSETNX key field value`
4. 增加数字  
    `HINCRBY key field increment`
5. 删除字段  
    `HDEL key field [field ...]`
6. 只获取字段名和字段值  
    `HKEYS key`  
    `KVALS key`
7. 获得字段数量  
    `HLEN key`

### 列表

可以存储一个有序的字符串列表，内部是使用双向链表实现的。  

1. 向列表两端增加元素  
    `LPUSH key value [value ...]`  
    `RPUSH key value [value ...]`  
2. 从列表两端弹出元素  
    `LPOP key`  
    `RPOP key`  
3. 获取列表中元素个数  
    `LLEN key`
4. 获取列表片段  
    `LRANGE key start end`
5. 删除列表中指定的值（删除列表中前count个值为value的元素）  
    `LREM key count value`  
    1. 当count > 0时，从列表左边开始删除count个值为value的元素
    2. 当count < 0时，从列表右边开始删除count个值为value的元素
    3. 当count = 0时，删除所有值为value的元素
6. 获得/设置指定索引的元素值  
    `LINDEX key index`  
    `LSET key index value`
7. 只保留列表指定片段  
    `LTRIM key start end`
8. 向列表中插入元素  
    `LINSERT key BEFORE|AFTER pivot value`  
    在列表中从左到右查找值为pivot的元素，然后根据BEFORE还是AFTER决定插入到该元素前面还是后面
9. 将元素从一个列表转到另一个列表  
    `RPOPLPUSH source destination`  
    从source右边弹出一个元素，然后加入到destination列表的左边，并返回这个元素的值

### 集合

在集合中每个元素都是不相同的，且没有顺序  

1. 增加/删除元素  
    `SADD key member [member ...]`  
    `SREM key member [member ...]`
2. 获得集合中的所有元素  
    `SMEMBERS key`  
3. 判断元素是否在集合中  
    `SISMEMBER key member`
4. 集合间运算  
    并：`SDIFF key [key ...]`  
    交：`SINTER key [key ...]`  
    差：`SONION key [key ...]`  
5. 获得集合中元素个数
    `SCARD key`
6. 进行集合运算并将结果存储  
    `SDIFFSTORE destination key [key ...]`  
    `SINTERSTORE destination key [key ...]`  
    `SONIONSTORE destination key [key ...]`  
7. 随机获得集合中元素  
    `SRANDMEMBER key [count]`  
    1. 当count为正数时，会随机从集合中获得count个不重复的元素，如果count的值大于集合中的元素个数，会返回集合中的全部元素  
    2. 当count为负数时，会随机从集合中获得|count|个，这些元素有可能相同
8. 从集合中弹出一个元素  
    `SPOP key`  
    由于集合类型是无序的，所以会随机选择一个元素弹出

### 有序集合

有序列表是使用散列表和跳跃表实现的  

1. 增加元素  
    `ZADD key score member [score member ...]`  
    分数支持整型，双精度浮点型，正负无穷大±inf
2. 获取元素的分数  
    `ZSCORE key member`  
3. 获得排名在某个范围的元素列表  
    从小到大：`ZRANGE key start end [WITHSCORES]`  
    从大到小：`ZREVRANGE key start end [WITHSCORES]`  
    WITHSCORES 为是否返回分数
4. 获得指定分数范围的元素  
    `ZRANGESCORE key min max [WITHSCORES] [LIMIT offset count]`  
    `ZREVRANGESCORE key min max [WITHSCORES] [LIMIT offset count]`  
    min max 默认返回分数的范围，包含端点值，在分数钱加上"("符号可以返回不包含端点值的元素  
    offset 从指定范围的第几个元素开始返回  
    count 返回的元素个数  
5. 增加某个元素的分数  
    `ZINCRBY key increment member`  
6. 获得集合中元素的数量
    `ZCARD key`  
7. 获得指定分数范围内的元素个数
    `ZCOUNT key min max`  
8. 删除一个或多个元素  
    `ZREM key member [member ...]`
9. 按照排名范围删除元素  
    `ZREMRANGEBYRANK key start stop`
10. 按照分数范围删除元素  
    `ZREMRANGEBYSCORE key min max`  
11. 获得元素的排名  
    `ZRANK key member`  
    `ZREVRANK key member`  
12. 计算有序集合的交集

## 进阶

### 事务

redis中的事务是一组命令的集合，事务和命令一样都是redis的最小执行单元  

1. 事务命令

    ```redis
    MULTI
    命令1
    命令2
    ......
    EXEC
    ```

2. 错误处理
    1. 语法错误：命令不存在或命令参数个数不对。  
        有错误执行EXEC后会直接返回错误，不执行命令。
    2. 运行错误：在命令执行期间出现的错误。  
        会执行没有出现错误的命令，Redis的事务没有回滚功能，要开发者去恢复数据库状态。
3. WATCH命令  
    `WATCH`和`UNWATCH`
    WATCH命令可以监控一个或多个键，一旦其中一个键被修改或删除，之后的事务就不会执行。

### 过期时间  

- 设置一个键的过期时间：`EXPIRE key seconds`，时间单位为秒  
- 查看一个键还有多久被删除：`TTL key`  
  - 返回值为剩余时间
  - 被删除，返回值为-2
  - 未设置过期时间，返回-1
- 取消键的过期时间：`PERSIST key`或者用`SET`或`GETSET`重新为键赋值
- `PEXPIRE`：时间单位为毫秒
- 不常用，使用Unix时间作为过期时刻
  - `EXPIREAT`
  - `PEXPIREAT`

### 排序

`SORT`命令可以对列表类型、集合类型和有序集合类型键进行排序，并且可以完成与关系数据库中连接查询相类似的任务。`SORT key`  
`SORT`命令还可以通过`ALPHA`参数实现字典顺序排序非数字元素。`SORT key ALPHA`  
分页`SORT key [DESC] LIMIT offset count`
`BY`对每个使用元素的值替换参考键中的第一个"*"并获取其值，然后依据该值对元素排序  
`GET`参数不影响排序，作用是使`SORT`命令返回结果不再是元素自身的值，而是`GET`参数中指定的键值，`GET #`返回元素本身的值  
`STORE`保存排序结果

### 发布订阅

Redis 发布订阅(pub/sub)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。

Redis 客户端可以订阅任意数量的频道。

广播形式。

- 订阅一个或多个频道：`SUBSCRIBE channel [channel]`
- 发布消息：`PULISH channel message`
- 取消订阅：`UNSUBSCRIBE channel [channel]`

### 管道

Redis是一种基于客户端-服务端模型以及请求/响应协议的TCP服务。这意味着通常情况下一个请求会遵循以下步骤：

- 客户端向服务端发送一个查询请求，并监听Socket返回，通常是以阻塞模式，等待服务端响应。
- 服务端处理命令，并将结果返回给客户端。

Redis 管道技术可以在服务端未响应时，客户端可以继续向服务端发送请求，并最终一次性读取所有服务端的响应。

## 部署方式

### 单机模式

简单，但是将数据完全存储在单个redis中主要存在两个问题：数据备份和数据体量较大造成的性能降低。

### 主从复制模式

在主从复制中，数据库分为两类，主数据库(master)和从数据库(slave)。其中主从复制有如下特点：

1. 主数据库可以进行读写操作，默认配置下，master节点可以进行读和写，slave节点只能进行读操作，写操作被禁止，当读写操作导致数据变化时会自动将数据同步给从数据库
2. 从数据库一般都是只读的，并且接收主数据库同步过来的数据，不要修改配置让slave节点支持写操作，没有意义，原因一，写入的数据不会被同步到其他节点；原因二，当master节点修改同一条数据后，slave节点的数据会被覆盖掉
3. 一个master可以拥有多个slave，但是一个slave只能对应一个master
4. slave节点挂了不影响其他slave节点的读和master节点的读和写，重新启动后会将数据从master节点同步过来
5. master节点挂了以后，不影响slave节点的读，Redis将不再提供写服务，master节点启动后Redis将重新对外提供写服务。
6. master节点挂了以后，不会slave节点重新选一个master

#### 优点

- 备份数据，这样当一个节点损坏（指不可恢复的硬件损坏）时，数据因为有备份，可以方便恢复。
- 读写分离提高性能，所有客户端都访问一个节点肯定会影响Redis工作效率，有了主从以后，查询操作就可以通过查询从节点来完成

#### 缺点

- 每个客户端连接redis实例的时候都是指定了ip和端口号的，如果所连接的redis实例因为故障下线了，而主从模式也没有提供一定的手段通知客户端另外可连接的客户端地址，因而需要手动更改客户端配置重新连接
- 主从模式下，如果主节点由于故障下线了，那么从节点因为没有主节点而同步中断，因而需要人工进行故障转移工作
- 无法实现动态扩容

### Sentinel（哨兵）模式

主从模式中，当master节点挂了以后，slave节点不能主动选举一个master节点出来，那么就安排一个或多个sentinel来做这件事，当sentinel发现master节点挂了以后，sentinel就会从slave中重新选举一个master。

1. sentinel模式是建立在主从模式的基础上，如果只有一个Redis节点，sentinel就没有任何意义

2. 当master节点挂了以后，sentinel会在slave中选择一个做为master，并修改它们的配置文件，其他slave的配置文件也会被修改，比如slaveof属性会指向新的master

3. 当master节点重新启动后，它将不再是master而是做为slave接收新的master节点的同步数据

4. sentinel因为也是一个进程有挂掉的可能，所以sentinel也会启动多个形成一个sentinel集群

5. 当主从模式配置密码时，sentinel也会同步将配置信息修改到配置文件中

6. 一个sentinel或sentinel集群可以管理多个主从Redis

7. sentinel最好不要和Redis部署在同一台机器，不然Redis的服务器挂了以后，sentinel也挂了

8. sentinel监控的Redis集群都会定义一个master名字，这个名字代表Redis集群的master Redis。

9. 当使用sentinel模式的时候，客户端就不要直接连接Redis，而是连接sentinel的ip和port，由sentinel来提供具体的可提供服务的Redis实现，这样当master节点挂掉以后，sentinel就会感知并将新的master节点提供给使用者

#### 优点

- Master状态监测
- 如果Master异常，则会进行Master-slave转换，将其中一个Slave作为Master，将之前的Master作为Slave 
- Master-Slave切换后，master_redis.conf、slave_redis.conf和sentinel.conf的内容都会发生改变，即master_redis.conf中会多一行slaveof的配置，sentinel.conf的监控目标会随之调换

#### 缺点

- 如果是从节点下线了，sentinel是不会对其进行故障转移的，连接从节点的客户端也无法获取到新的可用从节点
- 无法实现动态扩容

### Cluster（集群）模式

#### 介绍

Redis 集群不支持那些需要同时处理多个键的 Redis 命令， 因为执行这些命令需要在多个 Redis 节点之间移动数据， 并且在高负载的情况下， 这些命令将降低 Redis 集群的性能， 并导致不可预测的行为。

Redis 集群**通过分区（partition）来提供一定程度的可用性**（availability）： 即使集群中有一部分节点失效或者无法进行通讯， 集群也可以继续处理命令请求。

Redis 集群提供了以下两个好处：

- 将数据自动切分（split）到多个节点的能力。
- 当集群中的一部分节点失效或者无法进行通讯时， 仍然可以继续处理命令请求的能力。

#### 数据分片

Redis 集群使用数据分片（sharding）而非一致性哈希（consistency hashing）来实现： 一个 Redis 集群包含 `16384` 个哈希槽（hash slot）， 数据库中的每个键都属于这 `16384` 个哈希槽的其中一个， 集群使用公式 `CRC16(key) % 16384` 来计算键 `key` 属于哪个槽， 其中 `CRC16(key)` 语句用于计算键 `key` 的 [CRC16 校验和](http://zh.wikipedia.org/wiki/循環冗餘校驗) 。

集群中的每个节点负责处理一部分哈希槽。 举个例子， 一个集群可以有三个哈希槽， 其中：

- 节点 A 负责处理 `0` 号至 `5500` 号哈希槽。
- 节点 B 负责处理 `5501` 号至 `11000` 号哈希槽。
- 节点 C 负责处理 `11001` 号至 `16384` 号哈希槽。

这种将哈希槽分布到不同节点的做法使得用户可以很容易地向集群中添加或者删除节点。 比如说：

- 如果用户将新节点 D 添加到集群中， 那么集群只需要将节点 A 、B 、 C 中的某些槽移动到节点 D 就可以了。
- 与此类似， 如果用户要从集群中移除节点 A ， 那么集群只需要将节点 A 中的所有哈希槽移动到节点 B 和节点 C ， 然后再移除空白（不包含任何哈希槽）的节点 A 就可以了。

因为将一个哈希槽从一个节点移动到另一个节点不会造成节点阻塞， 所以无论是添加新节点还是移除已存在节点， 又或者改变某个节点包含的哈希槽数量， 都不会造成集群下线。

**要让集群正常运作至少需要三个主节点**， 不过在刚开始试用集群功能时， 强烈建议使用六个节点： 其中三个为主节点， 而其余三个则是各个主节点的从节点。

## Redis做缓存

缓存是Redis最常见的应用场景，之所有这么使用，主要是因为Redis读写性能优异。而且，Redis内部是支持事务的，在使用时候能有效保证数据的一致性。 作为缓存使用时，一般有两种方式保存数据：

1. 读取前，先去读Redis，如果没有数据，读取数据库，将数据拉入Redis。
2. 插入数据时，同时写入Redis。

### 缓存雪崩

由于原有缓存失效，新缓存未到期间(例如：我们设置缓存时采用了相同的过期时间，在同一时刻出现大面积的缓存过期)，所有原本应该访问缓存的请求都去查询数据库了，而对数据库CPU和内存造成巨大压力，严重的会造成数据库宕机。从而形成一系列连锁反应，造成整个系统崩溃。

有一个简单方案就是将缓存失效时间分散开，比如我们可以在原有的失效时间基础上增加一个随机值，比如1-5分钟随机，这样每一个缓存的过期时间的重复率就会降低，就很难引发集体失效的事件。

### 缓存穿透

缓存穿透是指用户查询数据，在数据库没有，自然在缓存中也不会有。这样就导致用户查询的时候，在缓存中找不到，每次都要去数据库再查询一遍，然后返回空（相当于进行了两次无用的查询）。这样请求就绕过缓存直接查数据库，这也是经常提的缓存命中率问题。

有很多种方法可以有效地解决缓存穿透问题，最常见的则是采用布隆过滤器，将所有可能存在的数据哈希到一个足够大的bitmap中，一个一定不存在的数据会被这个bitmap拦截掉，从而避免了对底层存储系统的查询压力。

另外也有一个更为简单的方法，如果一个查询返回的数据为空（不管是数据不存在，还是系统故障），我们仍然把这个空结果进行缓存，但它的过期时间会很短，最长不超过五分钟。通过这个直接设置的默认值存放到缓存，这样第二次到缓冲中获取就有值了，而不会继续访问数据库。

### 热点key

1. 访问的数据是一个热点key
2. 构建缓存需要时间

如果这两个问题同时出现，就可能造成在缓存失效的瞬间，有大量线程来构建缓存，造成后端负载加大，甚至可能会让系统崩溃。

解决方法：

1. 加锁
2. 热点数据永远不过期

## Redis做分布式锁



## Redis为什么快

1. redis是基于内存的，内存的读写速度非常快。

2. 数据结构简单，对数据操作也简单，Redis中的数据结构是专门进行设计的。

3. redis是单线程的，省去了很多上下文切换线程的时间。

4. redis使用多路复用技术，可以处理并发的连接。

### 单线程

- 多线程处理可能涉及到锁 
- 多线程处理会涉及到线程切换而消耗CPU

### 多路复用

对于一次IO访问（以read举例），数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。所以说，当一个read操作发生时，它会经历两个阶段：

1. 等待数据准备
2. 将数据从内核拷贝到进程中

IO multiplexing就是我们说的`select`，`poll`，`epoll`，有些地方也称这种IO方式为`event driven IO`。`select/epoll`的好处就在于单个process就可以同时处理多个网络连接的IO。它的基本原理就是select，poll，epoll这个function会不断的轮询所负责的所有socket，当某个socket有数据到达了，就通知用户进程。

![I/O 多路复用（ IO multiplexing）](https://pic.iminho.me/wiki/uploads/blog/201810/attach_1562465e5c832c14.png)

当用户进程调用了select，那么整个进程会被block，而同时，kernel会“监视”所有select负责的socket，当任何一个socket中的数据准备好了，select就会返回。这个时候用户进程再调用read操作，将数据从kernel拷贝到用户进程。

所以，I/O 多路复用的特点是通过一种机制一个进程能同时等待多个文件描述符，而这些文件描述符（套接字描述符）其中的任意一个进入读就绪状态，select()函数就可以返回。
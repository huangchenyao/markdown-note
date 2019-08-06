# Redis
[TOC]

## 简介，安装和启动
key:value存储数据库，存储在内存中，所以速度快  
更多公司用于缓存和队列系统  
win运行服务端 `redis-server.exe redis.windows.conf`  
win运行客户端 `redis-cli.exe -h 127.0.0.1 -p 6379`  

## 数据类型

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

### 消息通知
1. 队列
2. 发布/订阅 `PUBLISH/SUBSCRIBE`

### 管道


### 节省空间
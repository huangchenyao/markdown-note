[TOC]

# Redis数据结构

> 摘录自 https://redisbook.readthedocs.io/en/latest/index.html

## 内部数据结构

在 Redis 的内部， 数据结构类型值由高效的数据结构和算法进行支持， 并且在 Redis 自身的构建当中， 也大量用到了这些数据结构。

### SDS（Simple Dynamic String，简单动态字符串）

#### sds的用途

##### 实现字符串对象

redis是k-v存储的，value可以是各种各样的类型（包括字符串），但key都是字符串，redis的key的数据结构就是用的sds。

##### 取代C语言char*

在 Redis 中绝大部分字符串都是使用sds来表示， 客户端传入服务器的协议内容、 aof 缓存、 返回给客户端的回复， 等等， 这些重要的内容都是由 sds 类型来保存的。

#### Redis的字符串

C语言字符串存在的问题：

- 获取长度低效，复杂度为O(n)
- 对字符串进行追加，需要重新分配内存
- 以\0表示字符串结尾

Redis的sds定义如下：

```c
typedef char *sds;

struct sdshdr {
    // buf 已占用长度
    int len;

    // buf 剩余可用长度
    int free;

    // 实际保存字符串数据的地方
    char buf[];
};
```

其中，类型 `sds` 是 `char *` 的别名（alias），而结构 `sdshdr` 则保存了 `len` 、 `free` 和 `buf` 三个属性。

##### 追加操作优化

Redis使用内存预分配的策略来优化字符串追加，这样可以加快追加操作的速度，并降低内存分配的次数，代价是多占用了一些内存，而且这些内存不会被主动释放。

在目前版本的 Redis 中， `SDS_MAX_PREALLOC` 的值为 `1024 * 1024` ， 也就是说， 当大小小于 `1MB` 的字符串执行追加操作时， `sdsMakeRoomFor` 就为它们分配多于所需大小一倍的空间； 当字符串的大小大于 `1MB` ， 那么 `sdsMakeRoomFor` 就为它们额外多分配 `1MB` 的空间。



### 双端链表

- Redis 实现了自己的双端链表结构。
- 双端链表主要有两个作用：
  - 作为 Redis 列表类型的底层实现之一；
  - 作为通用数据结构，被其他功能模块所使用；
- 双端链表及其节点的性能特性如下：
  - 节点带有前驱和后继指针，访问前驱节点和后继节点的复杂度为 O(1) ，并且对链表的迭代可以在从表头到表尾和从表尾到表头两个方向进行；
  - 链表带有指向表头和表尾的指针，因此对表头和表尾进行处理的复杂度为 O(1)；
  - 链表带有记录节点数量的属性，所以可以在 O(1) 复杂度内返回链表的节点数量（长度）；



### 字典

#### 应用

- 数据库的键空间

  redis整个数据库的k-v都是用的字典

- 用作Hash类型的底层实现之一

  hash类型底层实现之一是字典，另外是ziplist


#### 实现

redis使用哈希表作为字典的底层实现

##### 字典实现

```c
/*
 * 字典
 *
 * 每个字典使用两个哈希表，用于实现渐进式 rehash
 */
typedef struct dict {
    // 特定于类型的处理函数
    dictType *type;

    // 类型处理函数的私有数据
    void *privdata;

    // 哈希表（2 个）
    dictht ht[2];

    // 记录 rehash 进度的标志，值为 -1 表示 rehash 未进行
    int rehashidx;

    // 当前正在运作的安全迭代器数量
    int iterators;
} dict;
```

0 号哈希表`ht[0]`是字典主要使用的哈希表， 而 1 号哈希表`ht[1]`则只有在程序对 0 号哈希表进行 rehash 时才使用。

##### 哈希表实现

哈希表

```c
/*
 * 哈希表
 */
typedef struct dictht {
    // 哈希表节点指针数组（俗称桶，bucket）
    dictEntry **table;

    // 指针数组的大小
    unsigned long size;

    // 指针数组的长度掩码，用于计算索引值
    unsigned long sizemask;

    // 哈希表现有的节点数量
    unsigned long used;
} dictht;
```

哈希表节点

```c
/*
 * 哈希表节点
 */
typedef struct dictEntry {

    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 链往后继节点
    struct dictEntry *next;

} dictEntry;
```

示例：

![image-20200902011647001](/Users/huangchenyao/Documents/markdown-note/中间件/redis/Redis数据结构.assets/image-20200902011647001.png)

##### 哈希算法

redis使用2种哈希算法：

- MurmurHash2 32 bit 算法

  multiply and rotate，因为算法的核心就是不断的`x *= m; x = rotate_left(x,r);`

  非加密，redis内部使用，不要求安全性

- 基于 djb 算法实现的一个大小写无关散列算法

  ```c
  unsigned long
      hash(unsigned char *str)
      {
          unsigned long hash = 5381;
          int c;
  
          while (c = *str++)
              hash = ((hash << 5) + hash) + c; /* hash * 33 + c */
  
          return hash;
      }
  ```

#### 添加数据

- 如果0号哈希表未初始化，则先初始化
- 插入时发生碰撞，则使用链地址法处理碰撞
- 如果插入数据触发了rehash，则开始处理rehash
- 插入数据时如果已经在rehash，则插入到1号哈希表中，否则数据插入到0号哈希表

#### Rehash

##### 何时触发

- 自然rehash，负载因子大于1，且变量 `dict_can_resize` 为真
- 强制rehash，负载因子大`dict_force_resize_ratio` （默认为5）

> `dict_can_resize`为假？
>
> 当redis在进行持久化时，为了高效利用cow机制，程序会暂时将 `dict_can_resize` 设为假， 避免执行自然 rehash ， 从而减少程序对内存的触碰（touch）。

##### 过程

1. 创建一个比 `ht[0]->table` 更大的 `ht[1]->table` 
   1. 设置字典的 `rehashidx` 为 `0` ，标识着 rehash 的开始
   2. 为 `ht[1]->table` 分配空间，大小至少为 `ht[0]->used` 的两倍
2. 将 `ht[0]->table` 中的所有键值对迁移到 `ht[1]->table` ，渐进式
3. 将原有 `ht[0]` 的数据清空，并将 `ht[1]` 替换为新的 `ht[0]` 
   1. 释放 `ht[0]` 的空间
   2. 用 `ht[1]` 来代替 `ht[0]` ，使原来的 `ht[1]` 成为新的 `ht[0]` 
   3. 创建一个新的空哈希表，并将它设置为 `ht[1]` 
   4. 将字典的 `rehashidx` 属性设置为 `-1` ，标识 rehash 已停止

##### 渐进式

redis的rehash操作不是一次性完成的，会分散到多个步骤（增、删、改、查）去执行，避免集中式的计算。

在rehash过程中，查询、删除会同时在`ht[0]`和`ht[1]`进行，插入数据会往`ht[1]`插入。

##### 字典收缩

跟扩容操作一样，扩容的`ht[1]`要比`ht[0]`大，收缩的`ht[1]`要比`ht[0]`小



### 跳表

跳跃表以有序的方式在层次化的链表中保存元素， 效率和平衡树媲美 —— 查找、删除、添加等操作都可以在对数期望时间下完成， 并且比起平衡树来说， 跳跃表的实现要简单直观得多。

#### redis中的实现

1. 允许重复的 `score` 值：多个不同的 `member` 的 `score` 值可以相同。
2. 进行对比操作时，不仅要检查 `score` 值，还要检查 `member` ：当 `score` 值可以重复时，单靠 `score` 值无法判断一个元素的身份，所以需要连 `member` 域都一并检查才行。
3. 每个节点都带有一个高度为 1 层的后退指针，用于从表尾方向向表头方向迭代：当执行 [ZREVRANGE](http://redis.readthedocs.org/en/latest/sorted_set/zrevrange.html#zrevrange) 或 [ZREVRANGEBYSCORE](http://redis.readthedocs.org/en/latest/sorted_set/zrevrangebyscore.html#zrevrangebyscore) 这类以逆序处理有序集的命令时，就会用到这个属性。

 跳跃表在 Redis 的唯一作用， 就是实现有序集数据类型。



## 内存映射数据结构

### 整数集合

整数集合（intset）用于有序、无重复地保存多个整数值， 根据元素的值， 自动选择该用什么长度的整数类型来保存元素。

根据需要， intset 可以自动从 `int16_t` 升级到 `int32_t` 或 `int64_t`。

Intset 是集合键的底层实现之一，如果一个集合：

1. 只保存着整数元素；

2. 元素的数量不多；

那么 Redis 就会使用 intset 来保存集合元素。

#### 定义

```c
typedef struct intset {
    // 保存元素所使用的类型的长度
    uint32_t encoding;

    // 元素个数
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];
} intset;

#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```

#### 升级

当插入的数据比现有的长度要大时，需要升级集合以及转移数据

1. 对新元素进行检测，看保存这个新元素需要什么类型的编码；
2. 将集合 `encoding` 属性的值设置为新编码类型，并根据新编码类型，对整个 `contents` 数组进行内存重分配。
3. 调整 `contents` 数组内原有元素在内存中的排列方式，从旧编码调整为新编码。
4. 将新元素添加到集合中。

只能升级不能降级。



### 压缩列表

Ziplist 是由一系列特殊编码的内存块构成的列表， 一个 ziplist 可以包含多个节点（entry）， 每个节点可以保存一个长度受限的字符数组（不以 `\0` 结尾的 `char` 数组）或者整数， 包括：

- 字符数组
  - 长度小于等于 `63` （2^6^-1）字节的字符数组
  - 长度小于等于 `16383` （2^14^-1） 字节的字符数组
  - 长度小于等于 `4294967295` （2^32^-1）字节的字符数组
- 整数
  - `4` 位长，介于 `0` 至 `12` 之间的无符号整数
  - `1` 字节长，有符号整数
  - `3` 字节长，有符号整数
  - `int16_t` 类型整数
  - `int32_t` 类型整数
  - `int64_t` 类型整数

因为 ziplist 节约内存的性质， 哈希键、列表键和有序集合键初始化的底层实现皆采用 ziplist。

#### 结构

ziplist

![image-20200902211445780](/Users/huangchenyao/Documents/markdown-note/中间件/redis/Redis数据结构.assets/image-20200902211445780.png)

entry

![image-20200902211522760](/Users/huangchenyao/Documents/markdown-note/中间件/redis/Redis数据结构.assets/image-20200902211522760.png)

添加和删除 ziplist 节点有可能会引起连锁更新，因此，添加和删除操作的最坏复杂度为 O(N^2^)，不过，因为连锁更新的出现概率并不高，所以一般可以将添加和删除操作的复杂度视为 O(N)。



## 基础数据类型

### Redis对象处理

redis对象结构

```c
/*
 * Redis 对象
 */
typedef struct redisObject {
    // 类型
    unsigned type:4;

    // 对齐位
    unsigned notused:2;

    // 编码方式
    unsigned encoding:4;

    // LRU 时间（相对于 server.lruclock）
    unsigned lru:22;

    // 引用计数
    int refcount;

    // 指向对象的值
    void *ptr;
} robj;
```

type

```c
/*
 * 对象类型
 */
#define REDIS_STRING 0  // 字符串
#define REDIS_LIST 1    // 列表
#define REDIS_SET 2     // 集合
#define REDIS_ZSET 3    // 有序集
#define REDIS_HASH 4    // 哈希表
```

Encoding

```c
/*
 * 对象编码
 */
#define REDIS_ENCODING_RAW 0            // 编码为字符串
#define REDIS_ENCODING_INT 1            // 编码为整数
#define REDIS_ENCODING_HT 2             // 编码为哈希表
#define REDIS_ENCODING_ZIPMAP 3         // 编码为 zipmap
#define REDIS_ENCODING_LINKEDLIST 4     // 编码为双端链表
#define REDIS_ENCODING_ZIPLIST 5        // 编码为压缩列表
#define REDIS_ENCODING_INTSET 6         // 编码为整数集合
#define REDIS_ENCODING_SKIPLIST 7       // 编码为跳跃表
```

redis一种数据类型会有多种编码方式，数据类型对应的编码方式如下：

![image-20200903120331626](/Users/huangchenyao/Documents/markdown-note/中间件/redis/Redis数据结构.assets/image-20200903120331626.png)

Redis 使用自己实现的对象机制来实现类型判断、命令多态和基于引用计数的垃圾回收。

### 字符串

#### 编码选择

字符串类型分别使用 `REDIS_ENCODING_INT` 和 `REDIS_ENCODING_RAW` 两种编码：

- `REDIS_ENCODING_INT` 使用 `long` 类型来保存 `long` 类型值。
- `REDIS_ENCODING_RAW` 则使用 `sdshdr` 结构来保存 `sds` （也即是 `char*` )、 `long long` 、 `double` 和 `long double` 类型值。
- 新创建的字符串默认使用 `REDIS_ENCODING_RAW` 编码， 在将字符串作为键或者值保存进数据库时， 程序会尝试将字符串转为 `REDIS_ENCODING_INT` 编码。

### 哈希表

#### 编码选择

创建空白哈希表时， 程序默认使用 `REDIS_ENCODING_ZIPLIST` 编码， 当使用 `REDIS_ENCODING_ZIPLIST` 编码哈希表时， 程序通过将键和值一同推入压缩列表， 从而形成保存哈希表所需的键-值对结构。

#### 编码转换

当以下任何一个条件被满足时， 程序将编码从 `REDIS_ENCODING_ZIPLIST` 切换为 `REDIS_ENCODING_HT` ：

- 哈希表中某个键或某个值的长度大于 `server.hash_max_ziplist_value` （默认值为 `64` ）。
- 压缩列表中的节点数量大于 `server.hash_max_ziplist_entries` （默认值为 `512` ）。

### 列表

#### 编码选择

创建新列表时 Redis 默认使用 `REDIS_ENCODING_ZIPLIST` 编码。

#### 编码转换

当以下任意一个条件被满足时， 列表会被转换成 `REDIS_ENCODING_LINKEDLIST` 编码：

- 试图往列表新添加一个字符串值，且这个字符串的长度超过 `server.list_max_ziplist_value` （默认值为 `64` ）。
- `ziplist` 包含的节点超过 `server.list_max_ziplist_entries` （默认值为 `512` ）。

### 集合

#### 编码选择

第一个添加到集合的元素， 决定了创建集合时所使用的编码：

- 如果第一个元素可以表示为 `long long` 类型值（也即是，它是一个整数）， 那么集合的初始编码为 `REDIS_ENCODING_INTSET` 。
- 否则，集合的初始编码为 `REDIS_ENCODING_HT` 。

#### 编码转换

如果一个集合使用 `REDIS_ENCODING_INTSET` 编码， 那么当以下任何一个条件被满足时， 这个集合会被转换成 `REDIS_ENCODING_HT` 编码：

- `intset` 保存的整数值个数超过 `server.set_max_intset_entries` （默认值为 `512` ）。
- 试图往集合里添加一个新元素，并且这个元素不能被表示为 `long long` 类型（也即是，它不是一个整数）。

### 有序集

#### 编码选择

在通过 zadd 命令添加第一个元素到空 `key` 时， 程序通过检查输入的第一个元素来决定该创建什么编码的有序集。

如果第一个元素符合以下条件的话， 就创建一个 `REDIS_ENCODING_ZIPLIST` 编码的有序集：

- 服务器属性 `server.zset_max_ziplist_entries` 的值大于 `0` （默认为 `128` ）。
- 元素的 `member` 长度小于服务器属性 `server.zset_max_ziplist_value` 的值（默认为 `64` ）。

否则，程序就创建一个 `REDIS_ENCODING_SKIPLIST` 编码的有序集。

#### 编码转换

对于一个 `REDIS_ENCODING_ZIPLIST` 编码的有序集， 只要满足以下任一条件， 就将它转换为 `REDIS_ENCODING_SKIPLIST` 编码：

- `ziplist` 所保存的元素数量超过服务器属性 `server.zset_max_ziplist_entries` 的值（默认值为 `128` ）
- 新添加元素的 `member` 的长度大于服务器属性 `server.zset_max_ziplist_value` 的值（默认值为 `64` ）
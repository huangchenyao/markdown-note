[TOC]

# Redis数据结构

## 内部数据结构

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



## 内存映射数据结构



## 基础数据类型


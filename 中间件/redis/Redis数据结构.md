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

```C
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



### 双端链表



### 字典



### 跳表



## 内存映射数据结构



## 基础数据类型


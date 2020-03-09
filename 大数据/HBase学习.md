[TOC]

# HBase学习

## 基本概念

### RowKey

用来表示唯一一行记录的**主键**，HBase的数据是按照RowKey的**字典顺序**进行全局排序的，所有的查询都只能依赖于这一个排序维度。

### 稀疏矩阵

参考了Bigtable，HBase中一个表的数据是按照稀疏矩阵的方式组织的

**每一行中，列的组成都是灵活的，行与行之间并不需要遵循相同的列定义**

### Region

HBase中采用了"Range分区"，将Key的完整区间切割成一个个的"Key Range" ，每一个"Key Range"称之为一个Region。

也可以这么理解：将HBase中拥有数亿行的一个大表，**横向切割**成一个个"**子表**"，这一个个"**子表**"就是**Region**。

**Region是HBase中负载均衡的基本单元**，当一个Region增长到一定大小以后，会自动分裂成两个。

### Column Family

如果将Region看成是一个表的**横向切割**，那么，一个Region中的数据列的**纵向切割**，称之为一个**Column Family**。每一个列，都必须归属于一个Column Family，这个归属关系是在写数据时指定的，而不是建表时预先定义。

### KeyValue

每一行中的每一列数据，都被包装成独立的拥有**特定结构**的KeyValue，KeyValue中包含了丰富的自我描述信息，看的出来，KeyValue是支撑"稀疏矩阵"设计的一个关键点：一些Key相同的任意数量的独立KeyValue就可以构成一行数据。但这种设计带来的一个显而易见的缺点：**每一个KeyValue所携带的自我描述信息，会带来显著的数据膨胀**。



## Docker搭建HBase单机环境

### 创建hbase容器

```bash
docker run -d -h    \
-p 2181:2181 -p 8080:8080 -p 8085:8085 -p 9090:9090 -p 9095:9095    \
-p 16000:16000 -p 16010:16010 -p 16201:16201 -p 16301:16301   \
--name hbase6     \
harisekhon/hbase:1.3
```

**参数说明：**
`-d` 表示 后台运行
`-p` 表示 端口映射
`--name` 表示 容器别名，随机起名，要求唯一
`harisekhon/hbase` 是 image镜像

```bash
❯ docker run -d    \
-p 2181:2181 -p 8080:8080 -p 8085:8085 -p 9090:9090 -p 9095:9095    \
-p 16000:16000 -p 16010:16010 -p 16201:16201 -p 16301:16301   \
--name hbase6     \
harisekhon/hbase:1.3
9d15d9764d70d1ed99b979c7ba68e84a9bd03f1724fa052ed5e457ea9157d284

❯ docker ps -a
CONTAINER ID        IMAGE                  COMMAND                   CREATED              STATUS                        PORTS               NAMES
9d15d9764d70        harisekhon/hbase:1.3   "/bin/sh -c \"/entryp…"   About a minute ago   Exited (137) 52 seconds ago
```

### 进入hbase容器和hbase shell中

```bash
~
❯ docker container start hbase6
hbase6

~
❯ docker exec -it hbase6 /bin/bash
bash-4.4# hbase shell
2020-03-09 03:46:35,664 WARN  [main] util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 1.3.2, r1bedb5bfbb5a99067e7bc54718c3124f632b6e17, Mon Mar 19 18:47:19 UTC 2018

hbase(main):001:0>
```

### 访问hbase

http://localhost:16010/master-status


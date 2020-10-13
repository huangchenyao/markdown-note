[TOC]

# Mysql

## Docker运行

- 下载镜像

  `Docker pull mysql`

- 启动

  `docker run -itd --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql`




## 事务隔离级别与对应问题

| 隔离级别                     | 脏读 | 不可重复读 | 幻读 |
| ---------------------------- | ---- | ---------- | ---- |
| 读未提交（Read uncommitted） | Y    | Y          | Y    |
| 读已提交（Read committed）   | N    | Y          | Y    |
| 可重复读（Repeatable Read）  | N    | N          | Y    |
| 序列化（Serializable）       | N    | N          | N    |

### 脏读

所谓脏读是指一个事务中访问到了另外一个事务未提交的数据。

| TX1                                             | TX2                                              |
| ----------------------------------------------- | ------------------------------------------------ |
| begin                                           | begin                                            |
| select * from user where id=1; -- 查出来age为10 |                                                  |
|                                                 | update user set age=12 where id=1; --更新age为12 |
| select * from user where id=1; -- 查出来age为12 |                                                  |
| commit;                                         | commit;                                          |

### 不可重复读

一个事务读取同一条记录2次，得到的结果不一致。

与脏读不一样的地方，脏读是第二个事务未提交就能查到不一致数据，不可重复读是第二个事务提交了，查到不一致的数据。

| TX1                                             | TX2                                              |
| ----------------------------------------------- | ------------------------------------------------ |
| begin                                           | begin                                            |
| select * from user where id=1; -- 查出来age为10 |                                                  |
|                                                 | update user set age=12 where id=1; --更新age为12 |
|                                                 | commit;                                          |
| select * from user where id=1; -- 查出来age为12 |                                                  |
| commit;                                         |                                                  |

### 幻读

一个事务读取2次，得到的记录数不一样。

| TX1                                      | TX2                                                |
| ---------------------------------------- | -------------------------------------------------- |
| begin                                    | begin                                              |
| select count(*) from user; -- result = 1 |                                                    |
|                                          | insert into user (name, age) values ('name2', 10); |
|                                          | commit;                                            |
| select count(*) from user; -- result = 2 |                                                    |
| commit;                                  |                                                    |

### Mysql事务隔离级别

mysql查询与修改事务隔离级别语句：

`select @@global.transaction_isolation, @@transaction_isolation;`

`set [global | session] transaction isolation level [Read uncommitted | Read committed | Repeatable Read | Serializable];`

- Oracle默认事务隔离级别是：读已提交（Read committed）。测试中发现oracle会出现不可重复读的问题。

- Mysql的默认事务隔离级别是：可重复读（Repeatable Read）。测试中发现，在这个事务隔离级别下，不会出现不可重复读的问题，**并且也不会出现幻读的问题**。



## Mysql版本控制

### Mysql锁

#### 乐观锁

用数据版本（Version）记录机制实现，这是乐观锁最常用的一种实现方式。

`update user set age=xxx where id=1 and version=1.0`

#### 悲观锁

##### 共享锁(S)、排他锁(X)

- `select xxxx lock in share mode`，要设置`S`锁。
- `select xxx for update`，要设置`X`锁。

|      | S      | X      |
| ---- | ------ | ------ |
| S    | 兼容   | 不兼容 |
| X    | 不兼容 | 不兼容 |

InnoDB的行锁是通过给索引上的索引项加锁来实现的，只有通过索引条件进行数据检索，Innodb才使用行级锁。否则，将使用表锁（锁住索引的所有记录）。如果我们的删除/修改语句是没有命中索引的，则会变成表锁，锁住整个表，影响性能。

##### 意向共享锁(IS)、意向排他锁(IX)

- 意向共享锁。它预示着，事务正在或者有意向对表中的”某些行”加S锁。一个数据行在加共享锁之前必须先取得该表的IS锁。
- 意向排他锁。它预示着，事务正在或者有意向表中的“某些行”加X锁。一个数据行加排它锁之前必须先取得该表的IX锁。

意向锁仅仅是表明意向，它其实非常弱，意向锁之间可以相互并行，并不是排斥的。

|      | IS   | IX   |
| ---- | ---- | ---- |
| IS   | 兼容 | 兼容 |
| IX   | 兼容 | 兼容 |

但是，意向锁可以和表锁互斥。

|      | S      | X      |
| ---- | ------ | ------ |
| IS   | 兼容   | 不兼容 |
| IX   | 不兼容 | 不兼容 |

意向锁是InnoDB数据操作之前自动加的，不需要用户干预。

意向锁是表级锁。

这两个意向锁存在的意义是：

> 当事务想去进行锁表时，可以先判断意向锁是否存在，存在时则可快速的返回，告知该表不能启用表锁（也就是会锁住对应会话），提高了加锁的效率。

#### 行锁

行锁锁的是索引上的索引项，只有通过索引条件进行数据检索，Innodb才使用行级锁。否则，将使用表锁（锁住索引的所有记录）。

假设有这些数据：

![image-20201013204718850](/Users/huangchenyao/Documents/markdown-note/数据库/mysql.assets/image-20201013204718850.png)

##### 临键锁 next-key lock

当sql执行按照索引进行数据的检索时，查询条件为范围查找（between and < > 等等）并有数据命中，则SQL语句加上的锁为next-key lock。

锁住索引的记录区间加下一个记录区间，这个区间是左开右闭的。

```mysql
select * from user where age > 9 and age < 12 for update;
```

锁住(7, 11]与下一个区间(11, 14]，即(7, 14]。

##### 间隙锁 gap lock

当记录不存在时，临键锁退化成gap lock。

在上述检索条件下，如果没有命中记录，则退化成Gap锁，锁住数据不存在的区间（左开右开）。



##### 记录锁 record lock

唯一性索引条件为精准匹配，退化成record lock。

当SQL执行按照唯一性（Primary Key，Unique Key）索引进行数据的检索时，查询条件等值匹配且查询的数据存在，这是SQL语句上加的锁即为记录锁Record lock，锁住具体的索引项。



### MVCC Multi-Version Concurrency Control（多版本并发控制）

MVCC 是一种并发控制的方法，一般在数据库管理系统中，实现对数据库的并发访问；在编程语言中实现事务内存。

MVCC 提供了时点（point in time）一致性视图。MVCC 并发控制下的读事务一般使用**时间戳或者事务 ID**去标记当前读的数据库的状态（版本），读取这个版本的数据。读、写事务相互隔离，不需要加锁。**读写并存的时候，写操作会根据目前数据库的状态，创建一个新版本，并发的读则依旧访问旧版本的数据。**

一句话总结就是：MVCC(`Multiversion concurrency control`) 就是 同一份数据临时保留多版本的一种方式，进而实现并发控制。

#### Mysql中的实现

在MySQL中建表时，每个表都会有三列隐藏记录，其中和MVCC有关系的有两列

- 数据行的版本号 （DB_TRX_ID）
- 删除版本号 (DB_ROLL_PT)

#### 插入

```mysql
begin; -- 获取到的全局事务id是1
insert into user (name, age) values ('name1', 10);
insert into user (name, age) values ('name2', 10);
insert into user (name, age) values ('name3', 10);
commit;
```

|  id  | name  | age  | DB_TRX_ID | DB_ROLL_PT |
| :--: | :---: | :--: | :-------: | :--------: |
|  1   | name1 |  10  |     1     |    null    |
|  2   | name2 |  10  |     1     |    null    |
|  3   | name3 |  10  |     1     |    null    |

数据插入时，会把全局事务id记录到DB_TRX_ID中。

#### 删除

```mysql
begin；-- 获得全局事务ID = 3
delete user where id = 2;
commit;
```
执行完上述SQL之后数据并没有被真正删除，而是对删除版本号做改变。


|  id  | name  | age  | DB_TRX_ID | DB_ROLL_PT |
| :--: | :---: | :--: | :-------: | :--------: |
|  1   | name1 |  10  |     1     |    null    |
|  2   | name2 |  10  |     1     |     3      |
|  3   | name3 |  10  |     1     |    null    |

#### 修改

```mysql
begin; -- 获取全局系统事务ID=5
update user set name = 'aaa', age = 20 where id = 2;
commit;
```

修改数据的时候会先复制一条当前记录行数据，同时标记这条数据的数据行版本号为当前是事务版本号，最后把原来的数据行的删除版本号标记为当前是事务。

|  id  | name  | age  | DB_TRX_ID | DB_ROLL_PT |
| :--: | :---: | :--: | :-------: | :--------: |
|  1   | name1 |  10  |     1     |    null    |
|  2   | name2 |  10  |     1     |     3      |
|  3   | name3 |  10  |     1     |    null    |
|  2   |  aaa  |  20  |     3     |    Null    |

#### 查询

- 查找**数据行版本号早于当前事务版本号**的数据行记录

  也就是说，数据行的版本号要小于或等于当前是事务的系统版本号，这样也就确保了读取到的数据是当前事务开始前已经存在的数据，或者是自身事务改变过的数据

- 查找**删除版本号**要么为NULL，要么**大于当前事务版本号**的记录

  这样确保查询出来的数据行记录在事务开启之前没有被删除



### 快照读、当前读

普通的select是使用快照读的方式。

加锁的select，insert，update，delete是使用当前读的方式。

这也能解释为什么上面的幻读问题并没有出现在mysql中，但对于其他情况的幻读是存在的。


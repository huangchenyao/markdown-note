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

|      | S      | X      |
| ---- | ------ | ------ |
| S    | 兼容   | 不兼容 |
| X    | 不兼容 | 不兼容 |

读锁可以读，读锁不可写；写锁不可读也不可写。

##### 意向共享锁(IS)、意向排他锁(IX)

- 意向共享锁。它预示着，事务正在或者有意向对表中的”某些行”加S锁。`select xxxx lock in share mode`，要设置`IS`锁。
- 意向排他锁。它预示着，事务正在或者有意向表中的“某些行”加X锁。`select xxx for update`，要设置`IX`锁。

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

### 快照读、当前读

普通的select是使用快照读的方式。

加锁的select，insert，update，delete是使用当前读的方式。

这也能解释为什么上面的幻读问题并没有出现在mysql中，但对于其他情况的幻读是存在的。
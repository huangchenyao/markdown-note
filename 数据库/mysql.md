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

快照读
[TOC]
## 问题汇总
1. data studio连接到sample数据库
    - 账号为db2admin/80234833
    - 密码为Dx960331/pc密码
    - 选择用于JDBC和SQLJ的IBM数据服务器驱动程序
    - 主机填本机ip，不填localhost或者127.0.0.1（会连接不上报错）

2. Java连接到db2
    1. 在`/IBM/SQLLIB/java`下导入`db2jcc4.jar`，`db2jcc_licsence_cu.jar`这两个jar包
    2. 连接
        ```Java
        Class.forName("com.ibm.db2.jcc.DB2Driver").newInstance();
        String url = "jdbc:db2://host:post/database name";
        String usr = "xxx";
        String pwd = "xxx";
        Connection conn = DriverManager.getConnection(url, usr, pwd);
        
        conn.close();
        ```
    3. sql操作，用PreparedStatement执行参数化操作，避免SQL注入攻击
        ```Java
        PreparedStatement ps = conn.prepareStatement(sql);
        
        // 增
        PreparedStatement ps = conn.prepareStatement("insert into table(x,x,x) values(?,?,...)");
        ps.setInt/setString...(1/2/3..., xx);
        ps.excuteUpdate();
        
        // 删
        PreparedStatement ps = conn.prepareStatement("delete from table where xx=?...");
        ps.setInt/setString...(1/2/3..., xx);
        ps.excuteUpdate();
        
        // 改
        PreparedStatement ps = conn.prepareStatement("update table set xx=?... where xx=?...");
        ps.setInt/setString...(1/2/3..., xx);
        ps.excuteUpdate();
        
        // 查
        PreparedStatement ps = conn.prepareStatement("select ?... from table where column=?...");
        ps.setInt/setString...(1/2/3..., xx);
        
        ResultSet rs = ps.excuteQuery();
        while(rs.next()) {
            String xx = rs.getString(1/2/3...);
        }

        ```

3. 查看所用表和表字段（条件内容要大写）
    - `select * from sysibm.systables where type='T' and creator='SCHEMANAME'`
    - `select * from sysibm.columns where table_schema='SCHEMANAME' and table_name='TABLENAME'`

## SQL语句

### 表操作
- create table
- alter table
    - drop column
    - add column
- drop table

### 列操作
- select xx from table where xx
    - 别名
        - select xx as 别名 from table
        - select xx 别名 from table
        - select xx as "别 名" from table
    - case
        ```sql
        select xx
        case 
            when 条件1 then 表达式1
            when 条件2 then 表达式2
            ...
            else 表达式
        end
        [as]<新列名>
        from table
        where xx
        ```
    - join
        - 内连接（inner join on / join on）  
            ```
            select a.yy, b.zz from a inner join b on a.xx = b.xx;
            ```
            一般会写成
            ```sql
            select a.yy, b.zz from a, b, where a.xx = b.xx;
            ```
        - 全外连接（full outer join on）：左右表的元组全部选出来
            ```sql
            select a.yy, b.zz from a full join b on a.xx = b.xx;
            ```
        - 左外连接（left outer join on / left join on）：把左边的表的元组全部选出来
            ```sql
            select a.yy, b.zz from a left join b on a.xx = b.xx;
            ```
        - 右外连接（right outer join on / right join on）：把右边表的元组全部选出来
            ```sql
            select a.yy, b.zz from a right join b on a.xx = b.xx;
            ```
        - 交叉连接（cross join on）：自身连接，笛卡尔积
        - 星形连接
    - in
    - exists
- insert into table() values()
- update table set xx=value where xx=value
- delete from table where xx

## 完整性约束、索引与别名

### 默认值
1. null
2. 0
3. 空白
4. 零长度字符串
5. 日期
6. 时间或时间戳记
7. 用户定义的单值数据类型

### 约束
1. 非空约束
2. 唯一约束
3. 主键约束
4. 外键约束
5. 表检查约束

## SQL PL

### 组成部分
1. 数据定义语言DDL
    - create
    - alter
    - drop
2. 数据操纵语言DML
    - select
    - insert
    - update
    - merge*（db2特有）
3. 数据控制语言DCL
    - grant
    - revoke
    - deny

### 数据类型
大量内置类型+用户自定义类型
1. 单值数据类型  
    ```sql
    create type 名称 as 类型 with comparisons
    ```
    - with comparisons：比较运算符
2. 结构数据类型  
    ```sql
    create type 名称 [under 父类型] as (
        名称 类型,（内置类型和自定类型都可以）
        ...
    ) mode db2sql
    ```
    - mode db2sql必须
3. 数组数据类型  
    ```sql
    create type 名称 as 类型 array[数组最大值]
    ```

### 变量声明
```sql
declare 变量名 类型 [default null/值]
```

### 赋值
```sql
set var = ...
values into var
select into var
fetch into var
```

### 流程控制语句
1. 条件语句
    - if  
        ```
        if xxx
            then xxx
            else xxx
        end if
        ```
    - case
        ```
        case
            when xxx then xxx
            when xxx then xxx
            ...
            else xxx
        end
        ```
2. 循环语句
    - loop
        ```
        l1: loop
            xxx
            leave l1;
        end loop l1;
        ```
    - while
        ```
        while xxx
        do
            xxx
        end while;
        ```
    - repeat
        ```
        repeat
            xxx
            until xxx
        end repeat;
        ```
    - for
        ```
        for xxx as select ... from
        do
            xxx
        end for;
        ```

### 函数
1. 函数类型
    - 标量函数
    - 聚集函数
    - 表函数
    - 行函数
2. 内置函数
3. 用户自定义函数

### 存储过程
```
create procedure xxx
begin
    (in/out/inout var)
    xxx
end
```

### 触发器
- 前触发器：在更新、插入或删除操作前运行
- 后触发器：在更新、插入或删除操作后运行
- instead of触发器：与视图相关

## 一致性

### 数据库事务
1. 原子性：事务必须是原子工作，要么全部执行，要么全部不执行
2. 一致性：事务在完成时，必须使得所有数据都保持一致状态
3. 隔离性：并发事务所做的修改要隔离
4. 持久性：事务完成后，对系统的影响是永久性的

### 事务日志记录
日志文件：  
- 主日志文件
- 辅助日志文件  

日志记录模式：  
- 循环
- 归档

### 并发性控制
1. 丢失更新：应用程序A, B同时读写
2. 脏读：一个事务在提交操作结果之前，另一个事务可以看到该结果
3. 不可重复读：一个事务在提交结果之前，另一个事务可以修改和删除它
4. 幻象读：一个事务在提交查询结果之前，另一个事务可以更改该结果

### 锁
1. 基本概念
    1. 锁，软件机制
    2. 锁模式，最常用的两种是共享和排他
    3. 锁粒度
    4. 锁升级
    5. 暂挂
    6. 超时
    7. 死锁
2. 行级锁与表级锁的模式
    1. 表级锁
        1. S锁（共享，Share）：可以读，但是不可以修改
        2. X锁（互斥，独占，Exclusive）
        3. U锁（更新，Update）
        4. IN锁（无意图锁，Intent None）
        5. IS锁（意向共享，Intent Share）
        6. IX锁（意向互斥，Intent Exclusive）
        7. SIX锁（带意向互斥的共享，Share With Intent Exclusive）
        8. Z锁（超级互斥，Super Exclusive）
    2. 行级锁
        1. S锁
        2. U锁
        3. X锁
        4. W锁（弱互斥，Weak Exclusive）
        5. NS锁（下一键共享，Next Key Share）
        6. NW锁（下一键弱互斥，Next Key Weak Exclusive）

### 隔离级别
1. 可重复读（Repeatable Read，RR）
2. 读稳定性（Read Stability，RS）
3. 游标稳定性（Cursor Stability，CS）
4. 未提交读（Uncommitted Read，UR）
# Shedlock

shedlock是spring的定时任务锁，在分布式或者集群上可以保证同一时间同一个定时任务只执行1次。

可以使用MongoDB，jdbc，redis等作为锁的存储数据源。

## Jdbc配置

- name属性（锁的名称）必须指定，每次只能执行一个具有相同名字的任务。
- lockAtLeastFor属性，指定保留锁的最短时间。主要目的是在任务非常短的且节点之间存在时钟差异的情况下防止多个节点执行。这个属性是锁的持有时间。设置了多少就一定会持有多长时间，再此期间，下一次任务执行时，其他节点包括它本身是不会执行任务的。
- lockAtMostFor属性，设置锁的最大持有时间,为了解决如果持有锁的节点挂了,无法释放锁,其他节点无法进行下一次任务。

## 源码学习

1. Spring AOP拦截@SchedulerLock注解，获取待执行的task

2. 获取lock配置，若有配置则进入下一步带锁操作，没有就直接运行

   ```java
   if (!lockConfigOptional.isPresent()) {
     log.debug("No lock configuration for {}. Executing without lock.", task);
     task.run();
   } else {
     lockingTaskExecutor.executeWithLock(task, lockConfigOptional.get());
   }
   ```

3. 带锁执行task

   ```java
   lock = lockProvider.lock(lockConfig);
   if (lock.isPresent()) {
     try {
       log.debug("Locked {}", lockConfig.getName());
       task.run();
     } finally {
       log.debug("Unlocked {}", lockConfig.getName());
       lock.get().unlock();
     }
   } else {
     log.debug("Not executing {}. It's locked.", lockConfig.getName());
   }
   ```

4. jdbc中的lock()方法实际操作

   - insert时，因为name是主键，只有还不存在数据时会有1个定时能成功插入数据，其余的都会失败，然后走到update去
- update时，因为insert或者update已经更新了lock_until，所以where条件查不出数据，updatedRows会是0，只有1个定时任务能更新数据
  
   ```java
   boolean doLock(lockConfig) {
     if (insertRecord(lockConfig)) {
       return true;
     }
     return updateRecord(lockConfig);
   }
   
   boolean insertRecord(lockConfig) {
     sql = "insert into tableName (name, lock_until, locked_at, locked_by) values (?, ?, ?, ?)";
     try (
     	connection = dataSource.getConnection();
       statement = connection.prepareStatement(sql)
     ) {
       connection.setAutoCommit(true);
   		...
       int insertedRows = statement.executeUpdate();
       if (insertedRows > 0) {
         return true;
       }
     } catch (SQLException e) {
       handleInsertException(sql, e);
     }
     return false;
   }
   
   boolean updateRecord(lockConfig) {
     sql = "update tableName set lock_until = ?, locked_at = ?, locked_by = ? where name = ? and lock_until <= ?";
   	try (
     	connection = dataSource.getConnection();
       statement = connection.prepareStatement(sql)
     ) {
       connection.setAutoCommit(true);
   		...
       int updatedRows = statement.executeUpdate();
       return updatedRows > 0;
     } catch (SQLException e) {
       handleUpdateException(sql, e);
       return false;
     }
   }
   ```


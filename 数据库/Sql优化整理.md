[TOC]

# Sql优化整理

## limit语句

分页查询是最常用的场景之一，但也通常也是最容易出问题的地方。比如对于下面简单的语句，一般 DBA 想到的办法是在 type, name, create_time 字段上加组合索引。这样条件排序都能有效的利用到索引，性能迅速提升。

```sql
SELECT *
FROM			operation
WHERE 		type = 'xxx'
AND 			name = 'xxx'
ORDER BY 	create_time limit 1, 10;
```

好吧，可能90%以上的 DBA 解决该问题就到此为止。但当 LIMIT 子句变成 “LIMIT 1000000,10” 时，程序员仍然会抱怨：我只取10条记录为什么还是慢？

要知道数据库也并不知道第1000000条记录从什么地方开始，即使有索引也需要从头计算一次。出现这种性能问题，多数情形下是程序员偷懒了。

在前端数据浏览翻页，或者大数据分批导出等场景下，是可以将上一页的最大值当成参数作为查询条件的。SQL 重新设计如下：

```sql
SELECT *
FROM			operation
WHERE 		type = 'xxx'
AND 			name = 'xxx'
AND				create_time > 'xxx'
ORDER BY 	create_time limit 10;
```

在新设计下查询时间基本固定，不会随着数据量的增长而发生变化。

这个不仅是数据库可以用，es也可以使用这个技巧，这样就不会因为查询页数过大导致超时，内存溢出等问题。
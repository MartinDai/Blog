---
title: MySQL分页查询offset过大，Sql优化经验
date: 2019-02-20 21:05:00
categories: 
- MySQL
---

# 低性能版

```sql
SELECT
*
FROM table
where condition1 = 0
and condition2 = 0
and condition3 = -1
and condition4 = -1
order by id asc
LIMIT 2000 OFFSET 50000
```
当offset特别大时，这条语句的执行效率会明显减低，而且效率是随着offset的增大而降低的。
原因为：
MySQL并不是跳过`offset`行，而是取`offset+N`行，然后返回放弃前`offset`行，返回`N`行，当`offset`特别大，然后单条数据也很大的时候，每次查询需要获取的数据就越多，自然就会很慢。

# 优化版本

```sql
SELECT
*
FROM table
JOIN
(select id from table
where condition1 = 0
and condition2 = 0
and condition3 = -1
and condition4 = -1
order by id asc
LIMIT 2000 OFFSET 50000)
as tmp using(id)
```

或者

```sql
SELECT a.* FROM table a, 
(select id from table
where condition1 = 0
and condition2 = 0
and condition3 = -1
and condition4 = -1
order by id asc
LIMIT 2000 OFFSET 50000) b 
where a.id = b.id
```


先获取主键列表，再通过主键查询目标数据，即使offset很大，也是获取了很多的主键，而不是所有的字段数据，相对而言效率会提升很多。
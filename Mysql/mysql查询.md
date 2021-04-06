#  查询



## 1.分页

通过`limit`关键字

```mysql
select * from user limit 50 --查询前50条记录
select * from user limit 50,5 --查询从第50条记录开始的5条记录，
```

> 当offset特别大时，需要查找许多不必要的数据再抛弃，会影响性能，例如limit 10005,5 会先查找10000条，再抛弃前面的10000条数据

优化：

1.延迟关联

> 先通过索引（如主键索引）覆盖，再做一次关联查询所有的信息

```mysql
select * from stu inner join (select stu_id from stu order by age limit 100,5) using(stu_id);
```

2.从上一次查询的位置开始

> 如上次返回的是主键50~100的记录，那么下一页就可以从100这条记录开始查询

```mysql
select * from stu where stu_id>100 order by age limit 100,5
```

## 2.连接

分内连接，左连接，右连接

## 3，如何将一张表的数据部分更新到另外一张表



```mysql
update b set b.col = a.col from a,b where a.id=b.id

update b set b.col = a.col from b inner join a using(id)

update b set b.col = a.col from b left join a on b.id=a.id
```


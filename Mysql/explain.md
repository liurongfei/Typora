# explain

> explain是mysql中获取select语句执行计划的命令
>
> 通过explain命令，我们可以知道改select语句执行的相关信息

如下：

![image-20210324091354610](C:\Environment\Github\Typora\Mysql\image-20210324091354610.png)

## explain命令列说明

1. id



2. select_type

查询的类型，分为

* 简单查询：**simple**

* 复杂查询：**primary**、**subquery**、**derived**

3. table

要查询的表名

4. partitions



5. type

访问类型

All：全表查询

eq_ref:等职引用

index:使用索引查询

6. possible_keys

可能用到的索引

7. key

实际用到的索引

8. key_len

   索引长度

9. ref

表之间的引用

8. rows

   查询到结果行数

9. filtered

10. Extra

额外信息，一般是查询使用的方法

* Using index 查询时使用索引

* Using Where 查询时使用了where语句

  


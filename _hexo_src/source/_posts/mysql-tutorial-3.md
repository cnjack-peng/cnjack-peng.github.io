---
title: MySQL通识-SQL
date: 2017-12-14 09:17:52
tags: 数据库
categories: MySQL
---

## JOIN



### INNER JOIN
{% asset_img inner_join.gif INNER JOIN %}
INNER JOIN（内连接,或等值连接）：获取两个表中字段匹配关系的记录。默认为INNER JOIN。

### LEFT JOIN
{% asset_img left_join.gif LEFT JOIN %}
LEFT JOIN（左连接）：获取左表所有记录，即使右表没有对应匹配的记录。

### RIGHT JOIN
{% asset_img right_join.gif RIGHT JOIN %}
RIGHT JOIN（右连接）： 与 LEFT JOIN 相反，用于获取右表所有记录，即使左表没有对应匹配的记录。


## GROUP BY

分组依据为多条件组合成一个条件，当组合条件相同时为一组。

## MySql避免重复插入记录

提供三种在mysql中避免重复插入记录方法，主要是讲到了ignore,Replace,ON DUPLICATE KEY UPDATE三种方法。

### 使用ignore关键字

如果是用主键primary或者唯一索引unique区分了记录的唯一性,避免重复插入记录可以使用：
```sql
INSERT IGNORE INTO `table_name` (`email`, `phone`, `user_id`) 
VALUES ('test9@163.com', '99999', '9999');
```

这样当有重复记录就会忽略,执行后返回数字0。还有个应用就是复制表,避免重复记录：
```sql
INSERT IGNORE INTO `table_1` (`name`) SELECT `name` FROM `table_2`;
```

### 使用Replace

语法格式：
```sql
REPLACE INTO `table_name`(`col_name`, ...) VALUES (...);
REPLACE INTO `table_name` (`col_name`, ...) SELECT ...;
REPLACE INTO `table_name` SET `col_name`='value',
```

REPLACE的运行与INSERT很相像,但是如果旧记录与新记录有相同的值，则在新记录被插入之前，旧记录被删除，即：
1. 尝试把新行插入到表中
2. 当因为对于主键或唯一关键字出现重复关键字错误而造成插入失败时：从表中删除含有重复关键字值的冲突行,再次尝试把新行插入到表中。旧记录与新记录有相同的值的判断标准就是：表有一个PRIMARY KEY或UNIQUE索引，否则，使用一个REPLACE语句没有意义。该语句会与INSERT相同，因为没有索引被用于确定是否新行复制了其它的行。

**返回值：**
REPLACE语句会返回一个数，来指示受影响的行的数目。该数是被删除和被插入的行数的和。受影响的行数可以容易地确定是否REPLACE只添加了一行，或者是否REPLACE也替换了其它行：检查该数是否为1（添加）或更大（替换）。

示例:(phone字段为唯一索引)
```sql
REPLACE INTO `table_name` (`email`, `phone`, `user_id`) 
VALUES ('test569', '99999', '123');
```

另外,在 SQL Server 中可以这样处理：
```sql
if not exists (select phone from t where phone= '1')   
insert into t(phone, update_time) values('1', getdate()) 
else    
update t set update_time = getdate() where phone= '1'
```

### ON DUPLICATE KEY UPDATE

如‍上所写，你也可以在INSERT INTO…..后面加上 ON DUPLICATE KEY UPDATE方法来实现。如果您指定了ON DUPLICATE KEY UPDATE，并且插入行后会导致在一个UNIQUE索引或PRIMARY KEY中出现重复值，则执行旧行UPDATE。例如，如果列a被定义为UNIQUE，并且包含值1，则以下两个语句具有相同的效果：
```sql
INSERT INTO `table` (`a`, `b`, `c`) VALUES (1, 2, 3) 
ON DUPLICATE KEY UPDATE `c`=`c`+1; 
UPDATE `table` SET `c`=`c`+1 WHERE `a`=1;
```

如果行作为新记录被插入，则受影响行的值为1；如果原有的记录被更新，则受影响行的值为2。如果列b也是唯一列，则INSERT与此UPDATE语句相当：
```sql
UPDATE `table` SET `c`=`c`+1 WHERE `a`=1 OR `b`=2 LIMIT 1;
```

如果a=1 OR b=2与多个行向匹配，则只有一个行被更新。通常，您应该尽量避免对带有多个唯一关键字的表使用ON DUPLICATE KEY子句。

您可以在UPDATE子句中使用VALUES(col_name)函数从INSERT…UPDATE语句的INSERT部分引用列值。换句话说，如果没有发生重复关键字冲突，则UPDATE子句中的VALUES(col_name)可以引用被插入的col_name的值。本函数特别适用于多行插入。VALUES()函数只在INSERT…UPDATE语句中有意义，其它时候会返回NULL。
```sql
INSERT INTO `table` (`a`, `b`, `c`) VALUES (1, 2, 3), (4, 5, 6) 
ON DUPLICATE KEY UPDATE `c`=VALUES(`a`)+VALUES(`b`);
```

本语句与以下两个语句作用相同：
```sql
INSERT INTO `table` (`a`, `b`, `c`) VALUES (1, 2, 3) ON DUPLICATE KEY UPDATE `c`=3; 
INSERT INTO `table` (`a`, `b`, `c`) VALUES (4, 5, 6) ON DUPLICATE KEY UPDATE c=9;
```

注释：当您使用ON DUPLICATE KEY UPDATE时，DELAYED选项被忽略。

示例：
这个例子是我在实际项目中用到的：是将一个表的数据导入到另外一个表中，数据的重复性就得考虑(如下)，唯一索引为：email：
```sql
INSERT INTO `table_name1` (`title`, `first_name`, `last_name`, `email`, `phone`, 
`user_id`, `role_id`, `status`, `campaign_id`) 
    SELECT '', '', '', `table_name2`.`email`, `table_name2`.`phone`, NULL, NULL, 
'pending', 29 FROM `table_name2` 
    WHERE `table_name2`.`status` = 1 
ON DUPLICATE KEY UPDATE `table_name1`.`status`='pending'
```

再贴一个例子：
```sql
INSERT INTO `class` SELECT * FROM `class1` 
ON DUPLICATE KEY UPDATE `class`.`course`=`class1`.`course`
```

其它关键：DELAYED  做为快速插入，并不是很关心失效性，提高插入性能。
IGNORE  只关注主键对应记录是不存在，无则添加，有则忽略。

更多信息请看:  http://dev.mysql.com/doc/refman/5.1/zh/sql-syntax.html#insert

特别说明：在MYSQL中UNIQUE索引将会对null字段失效，也就是说(a字段上建立唯一索引)：
```sql
INSERT INTO `test` (`a`) VALUES (NULL);
```
是可以重复插入的（联合唯一索引也一样）。


## MySQL关联更新

假定我们有两张表，一张表为Product表存放产品信息，其中有产品价格列Price；另外一张表是ProductPrice表，我们要将ProductPrice表中的价格字段Price更新为Price表中价格字段的80%。在Mysql中我们有几种手段可以做到这一点。

### update table1 t1, table2 ts ...

```sql
UPDATE product p, productPrice pp
SET pp.price = pp.price * 0.8
WHERE p.productId = pp.productId
AND p.dateCreated < '2004-01-01'
```

### inner join更新

```sql
UPDATE product p
INNER JOIN productPrice pp
ON p.productId = pp.productId
SET pp.price = pp.price * 0.8
WHERE p.dateCreated < '2004-01-01'
```

### left outer join

我们也可以使用left outer join来做多表update，比方说如果ProductPrice表中没有产品价格记录的话，将Product表的isDeleted字段置为1，如下sql语句：
```sql
UPDATE product p
LEFT JOIN productPrice pp
ON p.productId = pp.productId
SET p.deleted = 1
WHERE pp.productId IS null
```

### 同时更新2张表

面的几个例子都是两张表之间做关联，但是只更新一张表中的记录，其实是可以同时更新两张表的，如下sql：
```sql
UPDATE product p
INNER JOIN productPrice pp
ON p.productId = pp.productId
SET pp.price = pp.price * 0.8,
p.dateUpdate = CURDATE()
WHERE p.dateCreated < '2004-01-01'
```

### 等价示例

对单表执行更新没有什么好说的，无非就是`update table_name set col1 = xx,col2 = yy where col = zz`，主要就是where条件的设置。有时候更新某个表可能会涉及到多张数据表，例如：
```sql
update table_1 set score = score + 5 where uid in (select uid from table_2 where sid = 10);
```

其实update也可以用到left join、inner join来进行关联，可能执行效率更高，把上面的sql替换成join的方式如下：
```sql
update table_1 t1 inner join table_2 t2 on t1.uid = t2.uid set score = score + 5 where t2.sid = 10;
```


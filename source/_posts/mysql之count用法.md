---
title: mysql之count用法
date: 2020-12-10 19:35:17
categories: 数据库
tags: 面试经典
---

这个常用的**COUNT**函数，却暗藏着很多玄机，尤其是在面试的时候，一不小心就会被虐。不信的话请尝试回答下以下问题：

> 1、COUNT有几种用法？
>
> 2、COUNT(字段名)和COUNT(*)的查询结果有什么不同？
>
> 3、COUNT(1)和COUNT(*)之间有什么不同？
>
> 4、COUNT(1)和COUNT(*)之间的效率哪个更高？
>
> 5、为什么《阿里巴巴Java开发手册》建议使用COUNT(*)
>
> 6、MySQL的MyISAM引擎对COUNT(*)做了哪些优化？
>
> 7、MySQL的InnoDB引擎对COUNT(*)做了哪些优化？
>
> 8、上面提到的MySQL对COUNT(*)做的优化，有一个关键的前提是什么？
>
> 9、SELECT COUNT(*) 的时候，加不加where条件有差别吗？
>
> 10、COUNT(*)、COUNT(1)和COUNT(字段名)的执行过程是怎样的？

### **COUNT(列名)、COUNT(常量)和COUNT(\*)之间的区别**

**`COUNT(常量)` 和 `COUNT(*)`表示的是直接查询符合条件的数据库表的行数。而`COUNT(列名)`表示的是查询符合条件的列的值不为NULL的行数。**

### **COUNT(\*)的优化**

前面提到了`COUNT(*)`是SQL92定义的标准统计行数的语法，所以MySQL数据库对他进行过很多优化。那么，具体都做过哪些事情呢？

这里的介绍要区分不同的执行引擎。MySQL中比较常用的执行引擎就是InnoDB和MyISAM。

MyISAM和InnoDB有很多区别，其中有一个关键的区别和我们接下来要介绍的`COUNT(*)`有关，那就是**MyISAM不支持事务，MyISAM中的锁是表级锁；****而InnoDB支持事务，并且支持行级锁。**

因为MyISAM的锁是表级锁，所以同一张表上面的操作需要串行进行，所以，**MyISAM做了一个简单的优化，那就是它可以把表的总行数单独记录下来，如果从一张表中使用COUNT(\*)进行查询的时候，可以直接返回这个记录下来的数值就可以了，当然，前提是不能有where条件。**

MyISAM之所以可以把表中的总行数记录下来供COUNT(*)查询使用，那是因为MyISAM数据库是表级锁，不会有并发的数据库行数修改，所以查询得到的行数是准确的。

但是，对于InnoDB来说，就不能做这种缓存操作了，因为InnoDB支持事务，其中大部分操作都是行级锁，所以可能表的行数可能会被并发修改，那么缓存记录下来的总行数就不准确了。

但是，InnoDB还是针对COUNT(*)语句做了些优化的。

在InnoDB中，使用COUNT(*)查询行数的时候，不可避免的要进行扫表了，那么，就可以在扫表过程中下功夫来优化效率了。

从MySQL 8.0.13开始，针对InnoDB的`SELECT COUNT(*) FROM tbl_name`语句，确实在扫表的过程中做了一些优化。前提是查询语句中不包含WHERE或GROUP BY等条件。

**我们知道，COUNT(\*)的目的只是为了统计总行数，所以，他根本不关心自己查到的具体值，所以，他如果能够在扫表的过程中，选择一个成本较低的索引进行的话，那就可以大大节省时间。**

我们知道，InnoDB中索引分为聚簇索引（主键索引）和非聚簇索引（非主键索引），聚簇索引的叶子节点中保存的是整行记录，而非聚簇索引的叶子节点中保存的是该行记录的主键的值。

所以，相比之下，非聚簇索引要比聚簇索引小很多，所以**MySQL会优先选择最小的非聚簇索引来扫表。****所以，当我们建表的时候，除了主键索引以外，创建一个非主键索引还是有必要的。**

至此，我们介绍完了MySQL数据库对于COUNT(*)的优化，这些优化的前提都是查询语句中不包含WHERE以及GROUP BY条件。

### **COUNT(\*)和COUNT(1)**

官方文档

> InnoDB handles SELECT COUNT(*) and SELECT COUNT(1) operations in the same way. There is no performance difference.

**所以，对于COUNT(1)和COUNT(\*)，MySQL的优化是完全一样的，根本不存在谁比谁快！**

那既然`COUNT(*)`和`COUNT(1)`一样，建议用哪个呢？

建议使用`COUNT(*)`！因为这个是SQL92定义的标准统计行数的语法

《阿里巴巴Java开发手册》中强制要求不让使用 `COUNT(列名)`或 `COUNT(常量)`来替代 `COUNT(*)`

![image-20201210194250149](C:\Users\Administrator.USER-20190627HM\AppData\Roaming\Typora\typora-user-images\image-20201210194250149.png)




# 建表语句

```sql
create table t1(id int primary key, a int, b int, index(a));

delimiter ;;
create procedure idata()
begin
  declare i int;

  set i=1;
  while(i<=1000)do
    insert into t1 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();
```



# union执行流程

执行下面语句，用到了 union，语义是取这两个子查询结果的并集(就是这两个集合加起来，重复的行只保留一行)

```sql
(select 1000 as f) union (select id from t1 order by id desc limit 2);
```



**explain 结果**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210713223410.png)

* 第二行的 `key=PRIMARY`，说明第二个子句用到了`索引id`
* 第三行的` Extra字段`，表示在对子查询的结果集做 union 时，使用了`临时表 (Using temporary)`



**执行流程**：

1. 创建一个`内存临时表`，这个临时表只有一个`整型字段 f`，并且 f 是`主键`字段
2. 执行第一个`子查询`，得到 1000 这个值，并存入临时表中
3. 执行第二个子查询：
   * 拿到第一行 id=1000，试图插入临时表中。但由于 1000 这个值已经存在于临时表了，违反了唯一性约束，所以插入失败，然后继续执行
   * 取到第二行 id=999，插入临时表成功
4. 从临时表中按行取出数据，返回结果，并删除临时表，结果中包含两行数据分别是 1000 和 999

这里的内存临时表起到了暂存数据的作用，而且计算过程还用上了`临时表主键 id 的唯一性约束`，实现了 union 的语义。



**执行流程图：**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210713223524.png)



如果把上面这个语句中的 union 改成 `union all `，就没有了“去重”的语义。



这样执行时就`依次执行子查询`，得到的结果直接作为结果集的一部分，发给客户端。因此也就`不需要临时表`了。

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210713223741.png)

第二行的 Extra 字段显示的是 `Using index`，表示只使用了`覆盖索引`，没有用临时表了



# group by执行流程

一个常见的使用临时表的例子是 group by

```sql
select id%10 as m, count(*) as c from t1 group by m;
```

这个语句是把表 t1 里的数据，按照` id%10 `进行分组统计，并按照 m 的结果排序后输出



**explain 结果**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210713224545.png)

* Using index，表示这个语句使用了`覆盖索引`，选择了索引 a，不需要回表
* Using temporary，表示使用了`临时表`
* Using filesort，表示需要`排序`



**执行流程**

1. 创建`内存临时表`，表里有两个字段 m 和 c，`主键是 m`
2. 扫描表 t1 的索引 a，依次取出叶子节点上的 id 值，计算` id%10 的结果`，记为 x
   * 如果临时表中没有主键为 x 的行，就插入一个记录 (x,1)
   * 如果表中有主键为 x 的行，就将 x 这一行的 c 值加 1
3. 遍历完成后，再根据字段 m 做排序，得到结果集返回给客户端



**执行流程图**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210713224743.png)

图中最后一步，对内存临时表的排序

`图 6 内存临时表排序流程`

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210713224905.png)

其中，`临时表的排序过程`就是图 6 中`虚线框内`的过程。



再看一下这条语句的执行结果：

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210713225136.png)

如果需是`并不需要对结果进行排序`，那可以在 SQL 语句末尾增加` order by null`，也就是改成

```sql
select id%10 as m, count(*) as c from t1 group by m order by null;
```

这样就`跳过了最后排序的阶段`，直接从`临时表`中取数据返回。返回的结果如图 8 所示

`图 8 group + order by null 的结果（内存临时表）`

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210713225228.png)

由于表 t1 中的 id 值是从 1 开始的，因此返回的结果集中第一行是 id=1；扫描到 id=10 的时候才插入 m=0 这一行，因此结果集里最后一行才是 m=0。



这个例子里由于临`时表只有 10 行`，内存可以放得下，因此全程只使用了内存临时表。

但是，内存临时表的大小是有限制的，参数` tmp_table_size `就是控制这个内存大小的，默认是 16M。



如果执行下面这个语句序列：

```sql
set tmp_table_size=1024;
select id%100 as m, count(*) as c from t1 group by m order by null limit 10;
```

把内存临时表的大小限制为最大 1024 字节，并把语句改成 `id % 100`，这样返回结果里有 100 行数据。但是，这时的内存临时表大小`不够存下这 100 行数据`，也就是说，执行过程中会发现内存临时表大小到达了上限（1024 字节）。



那么，这时候就会把内存临时表转成`磁盘临时表`，磁盘临时表默认使用的`引擎是 InnoDB`。 这时，返回的结果如图 9 所示。

`图 9 group + order by null 的结果（磁盘临时表）`

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210713225352.png)

如果这个表 t1 的数据量很大，很可能这个查询需要的磁盘临时表就会`占用大量的磁盘空间`。



# group by优化方法

## 方法1 索引

不论是使用内存临时表还是磁盘临时表，group by 逻辑都需要构造一个`带唯一索引的表`，执行代价都是比较高的。如果表的数据量比较大，上面这个 group by 语句执行起来就会很慢



要解决 group by 语句的优化问题，你可以先想一下这个问题：`执行 group by 语句为什么需要临时表`？



group by 的语义逻辑，是`统计不同的值出现的个数`。但是，由于`每一行的 id%100 的结果是无序的`，所以我们就需要有一个临时表，来记录并统计结果。



那么，如果扫描过程中可以`保证出现的数据是有序的`，是不是就简单了呢？



假设，现在有一个类似图 10 的这么一个数据结构，我们来看看 group by 可以怎么做。

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210713225926.png)

`图 10 group by 算法优化 - 有序输入`



可以看到，如果可以确保输入的数据是有序的，那么计算 group by 的时候，就只需要从左到右，顺序扫描，依次累加。也就是下面这个过程：

* 当碰到第一个 1 的时候，已经知道累积了` X 个 0`，结果集里的第一行就是 (0,X)
* 当碰到第一个 2 的时候，已经知道累积了` Y 个 1`，结果集里的第二行就是 (1,Y)

按照这个逻辑执行的话，扫描到整个输入的数据结束，就可以拿到 group by 的结果，`不需要临时表，也不需要再额外排序`。



你一定想到了，InnoDB 的索引，就可以满足这个输入有序的条件。

在 MySQL 5.7 版本支持了 `generated column` 机制，用来实现列数据的关联更新。

你可以用下面的方法创建一个列 z，然后在 z 列上创建一个索引（如果是 MySQL 5.6 及之前的版本，你也可以创建普通列和索引，来解决这个问题）。

```sql
alter table t1 add column z int generated always as(id % 100), add index(z);
```

这样，索引 z 上的数据就是类似图 10 这样有序的了。上面的 group by 语句就可以改成：

```sql
select z, count(*) as c from t1 group by z;
```



优化后的 group by 语句的 explain 结果，如下图所示：

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210713230041.png)

从 Extra 字段可以看到，这个语句的执行不再需要临时表，也不需要排序了。



## 方法2 直接排序

 如果碰上不适合创建索引的场景。那么这时的 group by 要怎么优化呢？



一个 group by 语句中需要放到临时表上的数据量特别大，却还是要按照“先放到内存临时表，插入一部分数据后，发现内存临时表不够用了再转成磁盘临时表”。



那么，MySQL 有没有让我们`直接走磁盘临时表`的方法呢？

> 答案是，有的。



在 group by 语句中加入` SQL_BIG_RESULT 这个提示（hint）`，就可以告诉优化器：这个语句涉及的数据量很大，请`直接用磁盘临时表`。



MySQL 的优化器一看，`磁盘临时表`是 `B+ 树存储`，存储效率不如数组来得高。所以，既然你告诉我数据量很大，那从磁盘空间考虑，还是`直接用数组来存`吧。



因此，下面这个语句的执行流程就是这样的：

```sql
select SQL_BIG_RESULT id%100 as m, count(*) as c from t1 group by m;
```

1. 初始化 `sort_buffer`，确定放入一个整型字段，记为 m
2. 扫描表 t1 的索引 a，依次取出里面的 id 值, 将 id%100 的值存入 sort_buffer 中
3. 扫描完成后，对 sort_buffer 的字段 m 做排序（如果 sort_buffer 内存不够用，就会利用磁盘临时文件辅助排序）
4. 排序完成后，就得到了一个有序数组。



根据`有序数组`，得到数组里面的不同值，以及每个值的出现次数。



下面两张图分别是执行流程图和执行 explain 命令得到的结果。

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210713230216.png)

图 12 使用 SQL_BIG_RESULT 的执行流程图



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210713230303.png)

图 13 使用 SQL_BIG_RESULT 的 explain 结果



从 Extra 字段可以看到，这个语句的执行没有再使用临时表，而是直接用了`排序算法`。



基于上面的 union、union all 和 group by 语句的执行过程的分析，我们来回答文章开头的问题：

**MySQL 什么时候会使用内部临时表？**

1. 如果语句执行过程可以一边读数据，一边直接得到结果，是不需要额外内存的，否则就需要额外的内存，来保存中间结果

2. join_buffer 是无序数组，sort_buffer 是有序数组，临时表是二维表结构；

3. 如果执行逻辑需要用到二维表特性，就会优先考虑使用临时表。比如我们的例子中，union 需要用到唯一索引约束， group by 还需要用到另外一个字段来存累积计数

   




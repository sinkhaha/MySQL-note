# 建表

```sql
-- 市民表
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`), // id为主键索引 
  KEY `city` (`city`) // city为普通索引
) ENGINE=InnoDB;
```



假设要查询城市是“杭州”的所有人名字，并且按照姓名排序返回前 1000 个人的姓名、年龄

```sql
select city,name,age from t where city='杭州' order by name limit 1000 ;
```

以下聊聊这个语句是怎么执行的，以及有什么参数会影响执行的行为



# 全字段排序

**city索引的示意图**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_city%E5%AD%97%E6%AE%B5%E7%B4%A2%E5%BC%95%E5%9B%BE)

如图，满足 `city='杭州’`条件的行，是从` ID_X 到 ID_(X+N) `的这些记录



因为 city加上了索引，所以explain执行如下

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210707221850.png)



**Extra字段**

1. Using filesort：表示需要排序，MySQL 会给每个线程分配一块`内存`用于排序，称为 `sort_buffer`



## 执行流程

这个语句执行流程如下 ：

1. `初始化sort_buffer`，确定要放入` name、city、age `这三个字段
2. 从索引 city 找到`第一个满足 city='杭州’`条件的`主键 id`，即图中的 ID_X
3. 到`主键 id 索引取出整行`，取 name、city、age 三个字段的值，`存入 sort_buffer 中`
4. 从索引 city 取下一个记录的主键 id
5. `重复步骤 3、4 `直到 city 的值不满足查询条件为止，对应的主键 id 也就是图中的 ID_Y
6. 对 sort_buffer 中的数据`按照字段 name 做快速排序`
7. 按照排序结果`取前 1000 行`返回给客户端

把这个排序过程，称为`全字段排序`



**执行流程的示意图如下**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_%E5%85%A8%E5%AD%97%E6%AE%B5%E6%8E%92%E5%BA%8F)

图中`“按 name 排序”`可能在内存中完成，也可能需要使用`外部排序`，这取决于`排序所需的内存和参数 sort_buffer_size`



## 外部排序

### 什么时候用外部排序

sort_buffer_size参数：就是MySQL 为排序开辟的内存（sort_buffer）的大小。



* 内存排序：要排序的数据量小于等于sort_buffer_size，排序就在内存中完成

* 外部排序：要排序的数据量大于sort_buffer_size，则要利用磁盘临时文件辅助排序，使用外部排序，外部排序一般使用`归并排序`算法。



### 怎么确定是否用了临时文件(归并排序)

可通过`查看 OPTIMIZER_TRACE表 的结果`，从结果中的 `number_of_tmp_files `中看`是否使用了临时文件`，执行以下语句

```sql
/* 打开optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a保存Innodb_rows_read的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b保存Innodb_rows_read的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算Innodb_rows_read差值 */
select @b-@a;
```

此时全排序的 OPTIMIZER_TRACE 部分结果如下

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_%E5%85%A8%E6%8E%92%E5%BA%8F%E6%98%AF%E5%90%A6%E4%BD%BF%E7%94%A8%E4%B8%B4%E6%97%B6%E6%96%87%E4%BB%B6)



**number_of_tmp_files**

表示排序过程中`使用的临时文件数`(此时为12个)

> 此时MySQL将需要排序的数据分成 12 份，每一份单独排序后存在这些临时文件中。然后把这 12 个有序文件再合并成一个有序的大文件



1. number_of_tmp_files为0

   > 表示排序可以直接在内存中完成
   >
   > 即sort_buffer_size大于需要排序的数据量的大小

2. number_of_tmp_files大于0

   > 表示需要放在临时文件中排序
   >
   > sort_buffer_size 越小，需要分成的份数越多，number_of_tmp_files 的值就越大



**sort_mode中packed_additional_fields的意思**

表示排序过程对字符串做了“紧凑”处理。即使 `name 字段的定义是 varchar(16)`，在排序过程中还是要按照实际长度来分配空间的。



**examined_row的意思**

最后一个查询语句 `select @b-@a `的返回结果是 4000，表示整个执行过程只扫描了 4000 行。



注意：为了避免对结论造成干扰，这里把 `internal_tmp_disk_storage_engine 设置成 MyISAM`。否则，`select @b-@a 的结果会显示为 4001`。

> 这是因为`查询 OPTIMIZER_TRACE 这个表时`，需要用到`临时表`，而` internal_tmp_disk_storage_engine 的默认值是 InnoDB`。如果使用的是 InnoDB 引擎的话，把数据从临时表取出来时，会让 Innodb_rows_read 的值加 1。



## 缺点

全字段排序对原表的数据读了一遍，剩下的操作都是在 `sort_buffer `和`临时文件`中执行的。



如果查询要返回的字段很多，即sort_buffer里要放的字段数太多，这样内存里能够同时放下的行数很少，要分成很多个临时文件，排序的性能会很差。



所以如果单行很大，这个方法效率不够好，可以使用`rowid排序`。



# rowid排序

## 什么时候启用

**max_length_for_sort_data参数：**

是专门控制`用于排序的行数据的长度`的一个参数，表示如果单行的长度超过这个值，MySQL就认为单行太大，要换成rowid排序算法。



**计算结果集city、name、age的总长度**

city、name、age 这三个字段的定义总长度是36

> 计算方式如下：
>
> int类型占用4个字节，varchar类型如果存的是中文，且是UTF8格式，那么1个中文占用3个字节，如果存的是英文字母，那么1个英文字母占用1个字节。此处是按英文字符计算，所以总长度为`16byte*1 + 16byte*1 + 4byte = 36`；如果varchar(16)里面存的全是中文，并且是utf8的话，应该是`16byte*3+16byte*3+4=100`



## 执行流程

现在把max_length_for_sort_data 设置为 16，再看看计算过程有什么改变

```sql
SET max_length_for_sort_data = 16;
```

新的rowid算法放入 `sort_buffer `的字段，只有要`排序的列（即 name 字段）和主键 id`

> 但排序的结果因为少了 city 和 age 字段的值，不能直接返回了



**整个执行流程如下：**

1. 初始化 sort_buffer，确定放入两个字段，即` name 和 id`

2. 从索引 city 找到`第一个满足 city='杭州’条件的主键 id`，也就是图中的 ID_X

3. 到主键 id 索引取出整行，取 `name、id 这两个字段`，存入 sort_buffer 中

4. 从索引 city 取下一个记录的主键 id

5. `重复步骤 3、4 直到不满足 city='杭州’条件为止`，也就是图中的 ID_Y

6. 对 sort_buffer 中的数据`按照字段 name 进行排序`

7. 遍历排序结果，取前 1000 行，并`按照 id 的值回到原表中取出 city、name 和 age 三个字段`返回给客户端


把这个过程称为 `rowid 排序`

> 注意：最后的`“结果集”`是一个逻辑概念，实际上 MySQL 服务端从排序后的 sort_buffer 中依次取出 id，然后到原表查到 city、name 和 age 这三个字段的结果，不需要在服务端再耗费内存存储结果，是直接返回给客户端的。
>



这个执行流程的示意图如下

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_rowid%E6%8E%92%E5%BA%8F)

跟`全字段排序`的区别是，rowid 排序多访问了一次表 t 的主键索引，就是最后一步的步骤 7



**问题**

这个时候执行 `select @b-@a`，结果会是多少呢？



首先，图中的 examined_rows 的值还是 4000，表示用于排序的数据是 4000 行。但是 select @b-@a 这个语句的值变成 5000 了。

因为这时候除了排序过程外，在排序完成后，还要根据 id 去原表取值。由于语句是 limit 1000，因此会`多读 1000 行`。



rowid 排序的 OPTIMIZER_TRACE 部分输出如下

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_rowid%E6%8E%92%E5%BA%8F%E7%9A%84OPTIMIZER_TRACE%E9%83%A8%E5%88%86%E8%BE%93%E5%87%BA)



从 OPTIMIZER_TRACE 的结果中，还能看到另外两个信息也变了

1. sort_mode 变成了 ，表示参与排序的只有 name 和 id 这两个字段
2. number_of_tmp_files 变成 10 了，是因为这时候参与排序的行数虽然仍然是 4000 行，但是每一行都变小了，因此需要排序的总数据量就变小了，需要的临时文件也相应地变少了



# 全字段排序 VS rowid 排序

## 区别

1. MySQL担心`排序内存太小`，会影响排序效率，才会采用` rowid 排序算法`，这样排序过程中一次可以排序更多行，但是`需要再回到原表去取数据`

2. MySQL认为内存足够大，会优先选择`全字段排序`，把需要的字段都放到 sort_buffer 中，这样排序后就会直接从内存里面返回查询结果了，不用再回到原表去取数据

> 这也就体现了 MySQL 的一个设计思想：如果内存够，就要多利用内存，尽量减少磁盘访问



对于 InnoDB 表，rowid 排序会要求`回表多造成磁盘读`，因此不会被优先选择



## **order by不一定有排序操作**

MySQL 做排序是一个`成本比较高`的操作，并不是所有的 order by 语句，都需要排序操作的。



从上面分析的执行过程，可以看到，MySQL 之所以需要生成临时表，并且在临时表上做排序操作，`其原因是原来的数据都是无序的`

> 如果能够保证从 city 这个索引上取出来的行，天然就是按照 name 递增排序的话，就可以不用再排序了



### 利用联合索引

可以在这个市民表上创建一个` city 和 name 的联合索引`：

```sql
alter table t add index city_user(city, name);
```



这个索引的示意图

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_city%E5%92%8Cname%E8%81%94%E5%90%88%E7%B4%A2%E5%BC%95)

整个查询过程的流程：

1. 从索引 (city,name) 找到`第一个满足 city='杭州’`条件的主键 id
2. 到主键 id 索引取出整行，取 `name、city、age `三个字段的值，作为结果集的一部分`直接返回`
3. 从索引 (city,name) 取下一个记录主键 id
4. `重复步骤 2、3`，直到查到第 1000 条记录，或者是`不满足 city='杭州’`条件时循环结束



![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/egg/3f590c3a14f9236f2d8e1e2cb9686692.jpg)

**这个查询过程不需要临时表，也不需要排序**



 explain 结果如下：

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_%E8%81%94%E5%90%88%E7%B4%A2%E5%BC%95explain%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92)

Extra 字段中`没有Using filesort` 了，也就是不需要排序了。而且由于` (city,name) 这个联合索引本身有序`，所以这个查询也不用把 4000 行全都读一遍，只要找到满足条件的前 1000 条记录就可以退出了。也就是说，在我们这个例子里，只需要扫描 1000 次。



### 利用覆盖索引

进一步优化，利用`覆盖索引`进行优化



覆盖索引：指索引上的信息足够满足查询请求，不需要再回到主键索引上去取数据



针对这个查询，可以创建一个 `city、name 和 age 的联合索引`进行进一步优化：

```sql
alter table t add index city_user_age(city, name, age);
```



对于` city 字段的值相同`的行来说，还是按照 name 字段的值`递增排`序的，此时的查询语句也就不再需要排序了。

这样执行流程就变成了：

1. 从索引 (city,name,age) 找到第一个满足` city='杭州’`条件的记录，取出其中的 city、name 和 age 这三个字段的值，作为结果集的一部分`直接返回`

2. 从`索引 (city,name,age) 取下一个记录`，同样取出这三个字段的值，作为结果集的一部分`直接返回`

3. 重复执行步骤 2，直到查到第 1000 条记录，或者是`不满足 city='杭州’`条件时循环结束

   

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_citynameage%E8%81%94%E5%90%88%E7%B4%A2%E5%BC%95)



explain 的结果

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_3%E4%B8%AA%E5%AD%97%E6%AE%B5%E8%81%94%E5%90%88%E7%B4%A2%E5%BC%95explain)



Extra 字段里面多了`“Using index”`，表示的就是使用了覆盖索引，性能上会快很多



# 获取随机消息-临时表

## 背景

一个英语学习 App 首页有一个随机显示单词的功能，也就是根据每个用户的级别有一个单词表，然后这个用户每次访问首页的时候，都会随机滚动显示三个单词。他们发现随着单词表变大，选单词这个逻辑变得越来越慢，甚至影响到了首页的打开速度。



SQL语句要如何写呢



对这个例子进行了简化：去掉每个级别的用户都有一个对应的单词表这个逻辑，直接就是从一个单词表中随机选出三个单词。



这个表的建表语句和初始数据的命令如下：

```sql
// 建表
CREATE TABLE `words` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `word` varchar(64) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

// 插入10000条记录
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=0;
  while i<10000 do
    insert into words(word) values(concat(char(97+(i div 1000)), char(97+(i % 1000 div 100)), char(97+(i % 100 div 10)), char(97+(i % 10))));
    set i=i+1;
  end while;
end;;
delimiter ;

call idata();
```



## 内存临时表

### 用rand()实现

首先，用 `order by rand() `来实现这个逻辑。

```sql
select word from words order by rand() limit 3;
```



**explain结果如下**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/egg/image-20210708210613068.png)



**Extra字段表示需要临时表，并且需要在临时表上排序**

1.  `Using temporary`，表示的是需要使用`临时表`

2.  `Using filesort`，表示的是需要`执行排序`操作



对于内存表，回表过程只是简单地根据数据行的位置，直接访问内存得到数据，根本不会导致多访问磁盘。

优化器没有了这一层顾虑，那么它会优先考虑的，就是`用于排序的行越小越好`了，所以，MySQL 这时就会选择` rowid 排序`。



order by rand() 使用了`内存临时表`，内存临时表排序的时候使用了` rowid 排序`方法。



### range()执行流程

1. 创建一个临时表

   > 这个临时表使用的是 `memory 引擎`，表里有两个字段，第一个字段是 double 类型，为了后面描述方便，记为字段 R，第二个字段是 varchar(64) 类型，记为字段 W。并且，这个表没有建索引

2. 从 words 表中，按主键顺序取出所有的 word 值。对于每一个 word 值，调用 rand() 函数生成一个大于 0 小于 1 的随机小数，并把这个随机小数和 word 分别存入临时表的 R 和 W 字段中，到此，`扫描行数是 10000`

3. 现在`临时表有 10000 行数据`了，接下来你要在这个`没有索引的内存临时表`上，按照字段 R 排序

4. 初始化 sort_buffer

   > sort_buffer 中有两个字段，一个是 double 类型，另一个是整型

5. 从内存临时表中一行一行地取出 R 值和位置信息，分别存入 sort_buffer 中的两个字段里。这个过程要对内存临时表做`全表扫描`，此时扫描行数增加 10000，变成了 20000。

6. 在 sort_buffer 中根据 R 的值进行排序

   > 注意，这个过程没有涉及到表操作，所以不会增加扫描行数

7. 排序完成后，取出前三个结果的位置信息，`依次到内存临时表中取出 word 值`，返回给客户端。这个过程中，访问了表的三行数据，总扫描行数变成了 20003



完整的排序执行流程图如下

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/17_%E9%9A%8F%E6%9C%BA%E6%8E%92%E5%BA%8F%E5%AE%8C%E6%95%B4%E6%B5%81%E7%A8%8B%E5%9B%BE1)



通过慢查询日志（slow log）来验证一下我们分析得到的扫描行数是否正确

```bash
# Query_time: 0.900376  Lock_time: 0.000347 Rows_sent: 3 Rows_examined: 20003
SET timestamp=1541402277;
select word from words order by rand() limit 3;
```

其中，Rows_examined：20003 就表示这个语句执行过程中扫描了 20003 行，也就验证了我们分析得出的结论。



图中的 pos 就是`位置信息`，这里的“位置信息”是个什么概念？在上一篇文章中，我们对 InnoDB 表排序的时候，明明用的还是 ID 字段。



### MySQL表是用什么来定位一行数据

innodb用`rowid`来唯一标识数据行

1. 对于有主键的innodb，rowid就是主键
2. 对于没有主键的innodb，rowid由`系统生成`一个长度为 6 字节的 rowid 来作为主键
3. MEMORY 引擎不是索引组织表。在这个例子里，可以认为它就是一个`数组`。因此，这个 rowid 其实就是`数组的下标`。 对于Memory引擎，rowid就是数组下标。



## 磁盘临时表

### tmp_table_size

tmp_table_size：限制了内存临时表的大小，默认值是 16M。



如果临时表大小超过了 tmp_table_size，那么内存临时表就会转成`磁盘临时表`。



磁盘临时表使用的`引擎默认是 InnoDB`，是由参数` internal_tmp_disk_storage_engine `控制的。

当使用磁盘临时表时，对应的就是一个`没有显式索引的 InnoDB 表的排序`过程。



### 复现使用磁盘临时表

把 tmp_table_size 设置成 1024，把 sort_buffer_size 设置成 32768， 把 max_length_for_sort_data 设置成 16

```sql
set tmp_table_size=1024;
set sort_buffer_size=32768;
set max_length_for_sort_data=16;
/* 打开 optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* 执行语句 */
select word from words order by rand() limit 3;

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G
```



查看 OPTIMIZER_TRACE表输出的部分信息

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210708212723.png)

因为max_length_for_sort_data 设置成 16，小于 word 字段的长度定义，所以看到 sort_mode 里面显示的是rowid排序，这个是符合预期的，参与排序的是随机值 R 字段和 rowid 字段组成的行。数据总行数是 10000，

超过了 sort_buffer_size 定义的 32768 字节



### 优先排序算法

**number_of_tmp_files 的值居然是 0，难道不需要用临时文件吗？**

这个 SQL 语句的排序`没有用到临时文件`，采用是 MySQL 5.6 版本引入的一个新的排序算法，即：`优先队列排序算法`。



**接下来，看看为什么没有使用临时文件的算法，也就是归并排序算法，而是采用了优先队列排序算法。**

我们现在的 SQL 语句，只需要`取 R 值最小的 3 个 rowid`。但是，如果使用归并排序算法的话，虽然最终也能得到前 3 个值，但是这个算法结束后，已经将 10000 行数据都排好序了。

也就是说，后面的 9997 行也是有序的了。但我们的查询并不需要这些数据是有序的。所以这浪费了非常多的计算量。



**而优先队列算法，就可以精确地只`得到三个最小值`，执行流程如下：**

1. 对于这 10000 个准备排序的 `(R,rowid)`，先取前三行，构造成一个最大堆

2. 取下一个行 (R’,rowid’)，跟当前堆里面最大的 R 比较，如果 R’小于 R，把这个 (R,rowid) 从堆中去掉，换成 (R’,rowid’)

3. 重复第 2 步，直到第 10000 个 (R’,rowid’) 完成比较

   

优先队列排序过程的示意图

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/17_%E4%BC%98%E5%85%88%E9%98%9F%E5%88%97%E6%8E%92%E5%BA%8F%E7%AE%97%E6%B3%95%E7%A4%BA%E4%BE%8B)



OPTIMIZER_TRACE 结果中，filesort_priority_queue_optimization 这个部分的` chosen=true`，就表示使用了优先队列排序算法，这个过程不需要临时文件，因此对应的 number_of_tmp_files 是 0。



这个流程结束后，我们构造的堆里面，就是`这个 10000 行里面 R 值最小的三行`。然后，依次把它们的 rowid 取出来，去`临时表`里面拿到 word 字段，这个过程就跟上一篇文章的` rowid 排序`的过程一样了。



看上面的SQL 查询语句：

```sql
select city,name,age from t where city='杭州' order by name limit 1000;
```

这里也用到了 limit，为什么没用优先队列排序算法呢？原因是，这条 SQL 语句是 limit 1000，如果使用优先队列算法的话，需要维护的堆的大小就是 1000 行的 (name,rowid)，超过了我设置的 sort_buffer_size 大小，所以只能使用归并排序算法。



总之，不论是使用哪种类型的临时表，order by rand() 这种写法都会让计算过程非常复杂，需要大量的扫描行数，因此排序过程的资源消耗也会很大。



## 随机排序方法

先把问题简化一下，如果只随机选择 1 个 word 值，可以怎么做呢？



### 方法一

**思路**

1. 取得这个表的主键 id 的`最大值 M` 和`最小值 N`
2. 用随机函数生成一个最大值到最小值之间的数 `X = (M-N)*rand() + N`
3. 取不小于 X 的第一个 ID 的行



**sql语句如下**

```sql
select max(id),min(id) into @M,@N from t ;
set @X= floor((@M-@N+1)*rand() + @N);
select * from t where id >= @X limit 1;
```

**优点**

1. 效率很高

   > `取 max(id) 和 min(id) 都是不需要扫描索引`的，第3步的select也可以用`索引快速定位`，可以认为就只扫描了 3 行

**缺点**

1. 不是真正的随机

   > 因为 ID如果不是连续的，ID 中间可能有空洞，因此选择不同行的概率不一样



### 方法二

**思路**

1. 取得整个表的行数，并记为 C
2. 取得 `Y = floor(C * rand())`。 floor 函数在这里的作用，就是取整数部分
3. 再用 `limit Y,1` 取得一行



**sql语句如下**

```sql
select count(*) into @C from t;

set @Y = floor(@C * rand());

set @sql = concat("select * from t limit ", @Y, ",1");

prepare stmt from @sql;

execute stmt;

DEALLOCATE prepare stmt;
```

由于 limit 后面的参数不能直接跟变量，所以我在上面的代码中使用了` prepare + execute` 的方法。你也可以把拼接 SQL 语句的方法写在应用程序中，会更简单些。



**优点**

1. 这个随机算法解决了前面方法1 里面明显的概率不均匀问题

**缺点**

2. MySQL 处理 `limit Y,1 `的做法就是按顺序一个一个地读出来，丢掉前 Y 个，然后把下一个记录作为返回结果，因此这一步需要扫描 Y+1 行。再加上，第一步扫描的 C 行，总共需要`扫描 C+Y+1 行`，执行代价比随机算法 1 的代价要高



按照方法 2 的思路，如果要随机取 3 个 word 值，流程如下

1. 取得整个表的行数，记为 C
2. 根据相同的随机方法得到 Y1、Y2、Y3
3. 再执行`三个 limit Y, 1 语句`得到三行数据



sql语句如下

```bash
select count(*) into @C from t;
set @Y1 = floor(@C * rand());
set @Y2 = floor(@C * rand());
set @Y3 = floor(@C * rand());
select * from t limit @Y1，1； //在应用代码里面取Y1、Y2、Y3值，拼出SQL后执行
select * from t limit @Y2，1；
select * from t limit @Y3，1；
```



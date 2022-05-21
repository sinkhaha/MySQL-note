# 建表

市民表的定义

```sql
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

因为` city字段加上了索引`，所以explain执行如下

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210707221850.png)



**Extra字段**

1. Using filesort：表示需要排序，MySQL 会给`每个线程`分配一块`内存`用于排序，称为 `sort_buffer`



## 执行流程

city索引的示意图

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_city%E5%AD%97%E6%AE%B5%E7%B4%A2%E5%BC%95%E5%9B%BE)

如图，满足 `city='杭州’`条件的行，是从` ID_X 到 ID_(X+N) `的这些记录



这个语句执行流程如下 ：

1. `初始化sort_buffer`，确定要放入` name、city、age `这三个字段
2. 从索引 city 找到`第一个满足 city='杭州’`条件的`主键 id`，即图中的 ID_X
3. 到`主键 id 索引取出整行`，取 name、city、age 三个字段的值，`存入 sort_buffer 中`
4. 从索引 city 取下一个记录的主键 id
5. `重复步骤 3、4 `直到 city 的值不满足查询条件为止，对应的主键 id 也就是图中的 ID_Y
6. 对 sort_buffer 中的数据`按照字段 name 做快速排序`
7. 按照排序结果`取前 1000 行`返回给客户端

把这个排序过程，称为`全字段排序`



执行流程的示意图如下

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_%E5%85%A8%E5%AD%97%E6%AE%B5%E6%8E%92%E5%BA%8F)



图中`“按 name 排序”`可能在内存中完成，也可能需要使用`外部排序`，这取决于`排序所需的内存和参数 sort_buffer_size`



## sort_buffer_size参数

就是MySQL 为排序开辟的内存（sort_buffer）的大小。



如果`要排序的数据量小于 sort_buffer_size`，排序就在内存中完成。但如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助排序。



## **怎么确定一个排序语句用了临时文件**

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



这个方法是通过`查看 OPTIMIZER_TRACE 的结果`来确认的，你可以从 `number_of_tmp_files `中看到`是否使用了临时文件`。

如下图`全排序的 OPTIMIZER_TRACE 部分结果`

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_%E5%85%A8%E6%8E%92%E5%BA%8F%E6%98%AF%E5%90%A6%E4%BD%BF%E7%94%A8%E4%B8%B4%E6%97%B6%E6%96%87%E4%BB%B6)



#### **number_of_tmp_files**

表示排序过程中`使用的临时文件数`(此时为12个)

> 为什么是12个文件，内存放不下时，就需要使用外部排序，外部排序一般使用`归并排序`算法。
>
> MySQL 将需要排序的数据分成 12 份，每一份单独排序后存在这些临时文件中。然后把这 12 个有序文件再合并成一个有序的大文件



1. number_of_tmp_files为0

   > 表示排序可以直接在内存中完成，即sort_buffer_size大于需要排序的数据量的大小

2. number_of_tmp_files大于0

   > 表示需要放在临时文件中排序
   >
   > sort_buffer_size 越小，需要分成的份数越多，number_of_tmp_files 的值就越大



**sort_mode中packed_additional_fields的意思**

表示排序过程对字符串做了“紧凑”处理。即使 `name 字段的定义是 varchar(16)`，在排序过程中还是要按照实际长度来分配空间的。



**examined_row的意思**

最后一个查询语句 `select @b-@a `的返回结果是 4000，表示整个执行过程只扫描了 4000 行。



这里需要注意的是，为了避免对结论造成干扰，我把 `internal_tmp_disk_storage_engine 设置成 MyISAM`。否则，`select @b-@a 的结果会显示为 4001`。



这是因为`查询 OPTIMIZER_TRACE 这个表时`，需要用到`临时表`，而` internal_tmp_disk_storage_engine 的默认值是 InnoDB`。如果使用的是 InnoDB 引擎的话，把数据从临时表取出来的时候，会让 Innodb_rows_read 的值加 1。



## 全字段排序的缺点

`全字段排序`对原表的数据读了一遍，剩下的操作都是在 `sort_buffer `和`临时文件`中执行的。



但这个算法有一个问题，就是`如果查询要返回的字段很多`的话，那么 sort_buffer 里面要放的字段数太多，这样内存里能够同时放下的行数很少，要分成很多个临时文件，排序的性能会很差。



所以如果`单行很大`，这个方法效率不够好，可以使用`rowid排序`



# rowid 排序

如果 MySQL 认为排序的`单行长度太大`会怎么做呢？



**max_length_for_sort_data的意思**

是 MySQL 中专门控制`用于排序的行数据的长度`的一个参数。

表示如果单行的长度超过这个值，MySQL 就认为单行太大，要换一个算法。



city、name、age 这三个字段的定义总长度是 36，现在把 max_length_for_sort_data 设置为 16，再看看计算过程有什么改变

> int类型占用4个字节，varchar类型如果存的是中文，且是UTF8格式，那么1个中文占用3个字节，如果存的是英文字母，那么1个英文字母占用1个字节。此处是按英文字符计算，所以总长度为`16byte*1 + 16byte*1 + 4byte = 36`；如果varchar(16)里面存的全是中文，并且是utf8的话，应该是`16byte*3+16byte*3+4=100`

```sql
SET max_length_for_sort_data = 16;
```



新的算法放入 `sort_buffer `的字段，`只有要排序的列（即 name 字段）和主键 id`

> 排序的结果就因为少了 city 和 age 字段的值，不能直接返回了



整个执行流程如下：

1. 初始化 sort_buffer，确定放入两个字段，即` name 和 id`

2. 从索引 city 找到`第一个满足 city='杭州’条件的主键 id`，也就是图中的 ID_X

3. 到主键 id 索引取出整行，取 `name、id 这两个字段`，存入 sort_buffer 中

4. 从索引 city 取下一个记录的主键 id

5. `重复步骤 3、4 直到不满足 city='杭州’条件为止`，也就是图中的 ID_Y

6. 对 sort_buffer 中的数据`按照字段 name 进行排序`

7. 遍历排序结果，取前 1000 行，并`按照 id 的值回到原表中取出 city、name 和 age 三个字段`返回给客户端

   

把它称为 `rowid 排序`



这个执行流程的示意图如下

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_rowid%E6%8E%92%E5%BA%8F)

跟`全字段排序`的区别是，rowid 排序多访问了一次表 t 的主键索引，就是步骤 7



需要说明的是，最后的`“结果集”`是一个逻辑概念，实际上 MySQL 服务端从排序后的 sort_buffer 中依次取出 id，然后到原表查到 city、name 和 age 这三个字段的结果，`不需要在服务端再耗费内存存储结果，是直接返回给客户端的`



**问题**

这个时候执行 `select @b-@a`，结果会是多少呢？



首先，图中的 examined_rows 的值还是 4000，表示用于排序的数据是 4000 行。但是 select @b-@a 这个语句的值变成 5000 了。

因为这时候除了排序过程外，在排序完成后，还要根据 id 去原表取值。由于语句是 limit 1000，因此会`多读 1000 行`。



rowid 排序的 OPTIMIZER_TRACE 部分输出如下

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/16_rowid%E6%8E%92%E5%BA%8F%E7%9A%84OPTIMIZER_TRACE%E9%83%A8%E5%88%86%E8%BE%93%E5%87%BA)



从 OPTIMIZER_TRACE 的结果中，你还能看到另外两个信息也变了

1. sort_mode 变成了 ，表示参与排序的只有 name 和 id 这两个字段
2. number_of_tmp_files 变成 10 了，是因为这时候参与排序的行数虽然仍然是 4000 行，但是每一行都变小了，因此需要排序的总数据量就变小了，需要的临时文件也相应地变少了



# 全字段排序 VS rowid 排序



## 区别

1. 如果 MySQL 实在是担心`排序内存太小`，会影响排序效率，才会采用` rowid 排序算法`，这样排序过程中一次可以排序更多行，但是`需要再回到原表去取数据`

2. 如果 MySQL 认为内存足够大，会优先选择`全字段排序`，把需要的字段都放到 sort_buffer 中，这样排序后就会直接从内存里面返回查询结果了，不用再回到原表去取数据

> 这也就体现了 MySQL 的一个设计思想：如果内存够，就要多利用内存，尽量减少磁盘访问



对于 InnoDB 表来说，rowid 排序会要求`回表多造成磁盘读`，因此不会被优先选择



## **order by不一定有排序操作**

MySQL 做排序是一个`成本比较高`的操作，并不是所有的 order by 语句，都需要排序操作的。



从上面分析的执行过程，可以看到，MySQL 之所以需要生成临时表，并且在临时表上做排序操作，`其原因是原来的数据都是无序的`

> 如果能够保证从 city 这个索引上取出来的行，天然就是按照 name 递增排序的话，就可以不用再排序了



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

Extra 字段中`没有 Using filesort` 了，也就是不需要排序了。而且由于` (city,name) 这个联合索引本身有序`，所以这个查询也不用把 4000 行全都读一遍，只要找到满足条件的前 1000 条记录就可以退出了。也就是说，在我们这个例子里，只需要扫描 1000 次。



**进一步优化**

利用`覆盖索引`进行优化



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



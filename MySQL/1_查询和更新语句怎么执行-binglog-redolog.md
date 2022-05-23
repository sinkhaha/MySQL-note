# 一、查询语句是怎么执行的

## MySQL基本架构示意图

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/MySQLStudy/mysql%E7%9A%84%E9%80%BB%E8%BE%91%E6%9E%B6%E6%9E%84%E5%9B%BE.png)




1. Server层

   > * 核心服务功能
   > * 所有内置函数(如日期、时间、数学和加密函数等)
   > * 所有跨存储引擎功能(如存储过程、触发器、视图)

2. 存储引擎层

   > 负责数据的存储和提取(插件式架构，有 InnoDB(MySQL5.5.5版本默认)、MyISAM、Memory 等多个存储引擎)



## 查看表存储引擎的3种方式

1. `show create table 表名`
2. `show table status from 库名 where name='表名'`
3. ``select * from information_schema.tables where table_schema='库名' and table_name='表名'`



## MySQL的5个组件

### 一、连接器

连接器负责跟客户端建立连接、获取权限、维持和管理连接

#### 连接过程

1. 输入MySQL连接命令
2. tcp握手建立连接 
3. 认证帐密 
4. 查权限(权限表)

>注意：对用户的权限做了修改后，只有再新建的连接才会使用新的权限设置，不会影响已经存在连接的权限

5. 连接完成

>如果此时没操作，此连接就处于空闲状态。wait_timeout参数控制空闲连接断开时间，默认8小时

```bash
// 查看连接命令，Command列为Sleep表示为空闲连接
show processlist;

// 输出如下 ,此时有一个空闲连接
+----+-----------------+-----------------+--------+---------+------+------------------------+------------------+
| Id | User            | Host            | db     | Command | Time | State                  | Info             |
+----+-----------------+-----------------+--------+---------+------+------------------------+------------------+
|  4 | event_scheduler | localhost       | NULL   | Daemon  | 1185 | Waiting on empty queue | NULL             |
|  8 | root            | localhost:49663 | NULL   | Sleep   | 1037 |                        | NULL             |
|  9 | root            | localhost       | mytest | Query   |    0 | starting               | show processlist |
+----+-----------------+-----------------+--------+---------+------+------------------------+------------------+
```

```bash
// 运行命令，查看wait_timeout控制空闲连接断开的时间参数
show variables like 'wait_timeout';

// 输出如下  28800秒即8小时
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| wait_timeout  | 28800 |
+---------------+-------+

```



#### 数据库的长连接和短连接

* 长连接客户端持续有请求，一直使用同一个连接

>缺点：内存占用过大，因为MySQL在执行过程中使用的内存的管理是在连接对象中的，这些资源会在断开连接时才释放，可能会导致MySQL异常重启

* 短连接：每次执行完很少的几次查询就断开连接，下次查询再重新建立一个

>缺点：建立连接的过程通常比较复杂，不适宜反复创建

**怎样解决长连接占用内存过大问题**

1. `定期断开`长连接 或者 程序里面`判断执行过一个占用内存的大查询后`断开连接，等要查询再重连
2. MySQL5.7及以后版本，可以在每次执行一个比较大的操作后，通过执行 `mysql_reset_connection` 来重新`初始化连接资源`，这个过程不需要重连和重新做权限验证



### 二、查询缓存(不建议用)

> MySQL8.0已经彻底删除了该功能

**流程**

1. 执行select语句
2. 先查缓存，因为执行完查询语句MySQl会`缓存查询过的语句`(key-value形式，key是语句的hash值，value是查询结果)
3. 缓存存在直接返回，不存在则执行后面的阶段



**查询缓存的缺点**

1. 对于更新比较频繁的数据库，命中率太低

2. 容易缓存失效频繁

   > 只要有对一个表的`更新`，这个表上所有的查询缓存都会被清空

3. 会额外耗费一次hash计算 

4. 耗费内存




**MySQL8.0之前的版本按需开启查询缓存**

将参数 `query_cache_type` 设置成 `DEMAND`，则默认都不使用查询缓存。

而对于要使用查询缓存的语句，可以用 `SQL_CACHE` 显式指定，如

```mysql
select SQL_CACHE * from T where ID=10；
```



### 三、分析器

如果没有命中查询缓存，就要开始真正执行语句了

#### 1. 词法分析

>识别语句的每个单词代表什么意思
>
>从输入的"select"识别出来这是一个查询语句，把字符串“T”识别成“表名 T”，把字符串“ID”识别成“列 ID”

#### 2. 语法分析

>判断输入的这个 SQL 语句是否满足 MySQL 语法，如果不正确，会报错`You have an error in your SQL syntax`

例如：

```mysql
// 从报错得知要关注的是紧接“use near”的内容
mysql> select * from t where ID=1;

ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'elect * from t where ID=1' at line 1
```



### 四、优化器

**优化器是在表里面有多个索引时，`决定使用哪个索引`**

或

**在一个语句有多表关联（join）时，决定各个表的`连接顺序`**



比如执行关联查询语句

```mysql
select * from t1 join t2 using(ID) where t1.c=10 and t2.d=20;
```

* 既可以先从表 t1 里面取出 c=10 的记录的 ID 值，再根据 ID 值关联到表 t2，再判断 t2 里面 d 的值是否等于 20
* 也可以先从表 t2 里面取出 d=20 的记录的 ID 值，再根据 ID 值关联到 t1，再判断 t1 里面 c 的值是否等于 10

这两种执行方法的逻辑结果是一样的，但是执行的效率会有不同，而优化器的作用就是决定选择使用哪一个方案



### 五、执行器

开始执行SQL语句

1. 先判断对该表是否有`查询权限`
2. 如果有权限则`打开表`继续执行

>打开表时，执行器就会根据表的引擎定义，去使用这个引擎提供的接口。



比如

```mysql
select * from T where ID=10;
```

假设表T中ID 字段`没有索引`，`执行器的执行流程`为：

1. 调用 `InnoDB 引擎相关接口`查表的第一行，判断 ID 值是不是 10，如果不是则跳过，如果是则将这行存到结果集中

2. 继续取“下一行”，重复相同的判断逻辑，直到取到这个表的最后一行

3. 执行器将上述`遍历`过程中`所有满足条件的行`组成的记录集作为结果集返回给客户端




数据库的`慢查询日志`有一个 `rows_examined` 的字段，表示这个语句执行过程中`扫描了多少行`。

这个值就是在执行器`每次调用引擎获取数据行`时累加的。

> 在有些场景下，执行器调用一次，在引擎内部则扫描了多行，因此引擎扫描行数跟 rows_examined 并不是完全相同的



# 二、更新语句是怎么执行的

## 基本流程

更新语句也和查询语句基本一样

1. 连接器连接

2. 把该表的所有查询缓存结果清空

3. 分析器分析

4. 优化器

5. 执行器找到要更新的数据，执行更新




## 两种日志

更新流程涉及到两个重要的`日志模块`

* redo log(重做日志)

* binlog(归档日志)




### 一、redo log和binlog区别

1. `redo log` 是` InnoDB 引擎`特有的

2. `binlog `是 MySQL 的`Server 层`实现的，所有引擎都可以使用

   

3. `redo log` 是`物理日志`，记录的是`“在某个数据页上做了什么修改”`

4. `binlog` 是`逻辑日志`，记录的是这个语句的原始逻辑，比如`“给 ID=2 这一行的 c 字段加 1”`

   

5. `redo log` 是`循环写的`，是`固定大小`的，所以空间会用完

6. `binlog` 是可以`追加写入的`

   > “追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志



7. binlog用于`数据库备份`，假设`备库`当前存的是`主库T1时间的数据`，那么要把`当前时间T-T1`之间的数据备份，则`取binlog从T1时间开始到T时间`的操作应用于备库即可

8. redo log用于`crash-safe保证`，假设主库突然crash停机了，重新开机后，`通过检测redo log状态与binlog记录`来决定是`回滚`还是`执行`以便保证数据完整性、以及确保binlog记录的事务与最终的数据一致（这样binlog用于数据库备份就是完整的）



### 二、redo log重做日志(InnoDB特有)

redo log 是 InnoDB 引擎`特有`的日志



**类比**

如果饭店老板把赊账的人记在`黑板`，redo log即黑板上记的名字



**要解决的问题**

MySQL更新数据时，如果每一次的更新操作都需要`写进磁盘`，然后磁盘也要`找到对应的那条记录`，然后再`更新`，整个过程 `IO成本`(随机IO)、`查找成本`都很高



**解决方法**

为了解决这个问题，MySQL 用了`先记录下更新的操作日志，事后再去写磁盘`的思路来提升更新效率，其实就是 `WAL 技术`



#### **WAL技术**

WAL 的全称是 `Write-Ahead Logging`，它的关键点就是`先写日志，再写磁盘`



**InnoDB引擎一条记录的更新流程**

1. 先查询页数据到buffer_pool

2. 更新buffer_pool的页数据

3. InnoDB引擎先记录redo log(顺序IO)

   > redo log也在磁盘，是一个固定的区域，是`顺序IO`

4. 更新内存

5. 此时更新就完成了

6. 后续系统比较空闲时，InnoDB会把redo log的记录刷到磁盘中



**redo log写满了怎么办**

InnoDB 的 redo log 是`固定大小`的，比如可以配置为`一组 4 个文件`，每个文件的大小是 1GB，那么总共就可以记录 4GB 的操作。

`当写满了，则会擦除掉一些记录，擦除记录之前会把记录更新到数据文件，即更新到磁盘`



 **crash-safe能力**

有了 `redo log`，`InnoDB` 就可以保证即使数据库发生`异常重启`，`之前提交的记录都不会丢失`，这个能力称为 `crash-safe`，MyISAM 没有 crash-safe 的能力




### 三、binlog归档日志

binlog 是 MySQL 的 Server 层实现的，`所有引擎都可以使用`



**binlog有两种模式**

* statement 格式：记录sql语句
* row格式：记录行的内容，`记两条`，更新前和更新后都有



### 四、执行update语句时的内部流程

比如执行语句：将 ID=2 这一行的值加 1, ID是主键

```mysql
update T set c=c+1 where ID=2;
```

1. 执行器先找引擎`取 ID=2 这一行`

   > ID 是主键，引擎直接`用树搜索`找到这一行。如果 ID=2 这一行所在的`数据页`本来就在`内存`中，就直接返回给`执行器`；否则，需要`先从磁盘读入内存`，然后再返回

2. 执行器拿到引擎给的`行数据`，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口`写入这行新数据`

3. 引擎将这行新数据`更新到内存中`，同时将这个`更新操作记录到 redo log` 里面，此时 redo log 处于` prepare 状态`。然后告知执行器执行完成了，`随时可以提交事务`

4. `执行器`生成这个操作的 `binlog`，并`把 binlog 写入磁盘`

5. 执行器调用引擎的`提交事务`接口，引擎把刚刚写入的 redo log 改成`提交（commit）状态`，更新完成



**这个过程涉及到了两阶段提交**

最后三步中，将 `redo log 的写入`拆成了两个步骤：prepare 和 commit



如图，图中`浅色框`表示是在 `InnoDB 内部`执行的，`深色框`表示是在`执行器`中执行的

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/MySQLStudy/update%E8%AF%AD%E5%8F%A5%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.png)



### 五、两阶段提交

两阶段提交是为了`让两份日志之间的逻辑一致`



**怎样让数据库恢复到半个月内任意一秒的状态？**

当需要恢复到指定的某一秒时，比如`某天下午两点`发现中午十二点有一次误删表，需要`找回数据`，那你可以这么做

1. 首先，找到最近的一次`全量备份`，如果你运气好，可能就是昨天晚上的一个备份，从这个备份恢复到`临时库`
2. 然后，`从最近一次备份的时间点开始`，`将备份的 binlog 依次取出来`，重放到中午误删表之前的那个时刻

这样你的临时库就跟误删之前的线上库一样了，然后你可以把表数据从临时库取出来，按需要恢复到线上库去



**为什么日志需要“两阶段提交”**

由于 redo log 和 binlog 是`两个独立的逻辑`，如果不用两阶段提交，要么就是先写完 redo log 再写 binlog，或者采用反过来的顺序，那么`数据库的状态`就有可能和`用它的日志恢复出来的库的状态`不一致



用前面的 update 语句来做例子，假设当前 ID=2 的行，字段 c 的值是 0，再假设执行 update 语句过程中在写完第一个日志后，第二个日志还没有写完期间发生了 crash，会出现什么情况呢？

1. 先写 redo log 后写 binlog（可能影响备份日志的binlog）

>假设在 redo log `写完`，binlog 还`没有写完`时，`MySQL 进程异常重启`。
>
>由于前面说过，redo log 写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行 c 的值是 1。但是由于 binlog 没写完就 crash 了，这时 binlog 里面就没有记录这个语句
>
>因此，之后`备份日志`的时候，存起来的 binlog 里面就没有这条语句。`如果需要用这个 binlog 来恢复临时库的话`，由于这个语句的 binlog 丢失，这个临时库就会少了这一次更新，恢复出来的这一行 c 的值就是 0，与原库的值不同

2. 先写 binlog 后写 redo log

>如果在 binlog 写完之后 crash，`由于 redo log 还没commit即没写，崩溃恢复以后这个事务无效`，所以这一行 c 的值是 0。但是 binlog 里面已经记录了`“把 c 从 0 改成 1”这个日志`。所以，在之后用 binlog 来恢复时就多了一个事务出来，恢复出来的这一行 c 的值就是 1，与原库的值不同



**平时恢复临时库的场景**

比如当需要`扩容`时，也就是需要再`多搭建一些备库来增加系统的读能力`时，

常见的做法是`用全量备份`加上`应用 binlog 来`实现的，如果redo log和binlog“不一致”就会导致你的线上出现`主从数据库不一致`的情况



# 三、小结

1. redo log 用于保证 crash-safe 能力

   `innodb_flush_log_at_trx_commit 这个参数设置成 1 时，表示`每次事务的 redo log 都直接持久化到磁盘`。这个参数我建议你设置成 1，这样可以`保证 MySQL 异常重启之后数据不丢失

2. `sync_binlog` 这个参数设置成 1 时，`表示每次事务的 binlog 都持久化到磁盘`。这个参数我也建议你设置成 1，这样保证 MySQL 异常重启之后 binlog 不丢失





# 四、binlog 的写入机制

## binlog 的写入逻辑

1. 事务执行过程中，先把日志写到 `binlog cache`
2. 事务提交时，执行器再把 `binlog cache` 的完整事务写到` binlog 文件`中，并清空binlog cache



一个事务的 binlog 是不能被拆开的，因此不论这个事务多大，也要确保一次性写入。这就涉及到了 binlog cache 的保存问题



## binlog cache

系统给 binlog cache 分配了一片内存，`每个线程一个binlog cache `，但是`共用同一份 binlog 文件`



### 参数 binlog_cache_size

用于控制`单个线程`内 binlog cache 所占内存的大小。如果超过了这个参数规定的大小，就要`暂存到磁盘`



**状态如图所示**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/23_%E5%9B%BE1binlog%E5%86%99%E7%9B%98%E7%8A%B6%E6%80%81)



1. 图中的 write，指的就是指把日志写入到文件系统的 page cache，并没有把数据持久化到磁盘，所以速度比较快
2. 图中的 fsync，才是将数据`持久化到磁盘`的操作。一般情况下，我们认为 fsync 才占磁盘的 IOPS



### sync_binlog参数

write 和 fsync 的时机是由参数 `sync_binlog` 控制的

1. sync_binlog=0 时，表示每次提交事务都只 write，不 fsync

2. sync_binlog=1 时，表示每次提交事务都会执行 fsync

3. sync_binlog=N(N>1) 时，表示每次提交事务都 write，但累积 N 个事务后才 fsync

   > 因此，在出现` IO瓶颈`的场景里，`将 sync_binlog 设置成一个比较大的值`，可以提升性能。



在实际的业务场景中，考虑到丢失日志量的可控性，一般不建议将这个参数设成 0，比较常见的是将其设置为 100~1000 中的某个数值。

但是，将 sync_binlog 设置为 N，对应的风险是：如果主机发生异常重启，会`丢失最近N个事务`的 binlog 日志



# 五、redo log 的写入机制

事务在执行过程中，生成的 redo log 是要先写到` redo log buffer`



## redo log buffer

1. redo log buffer 里面的内容，不是每次生成后都要直接持久化到磁盘

> 如果事务执行期间 MySQL 发生异常重启，那这部分日志就丢了。由于事务并没有提交，所以这时日志丢了也不会有损失

2. 事务还没提交时，redo log buffer 中的部分日志`有可能`被持久化到磁盘



## redo log 可能存在的3种状态

MySQL redo log 存储状态图

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210710152027.png)



**redo log 的三种状态分别是**

1. 存在 `redo log buffer` 中

   物理上是在 `MySQL 进程内存`中，就是图中的红色部分

2. 写到磁盘 (write)，但是没有持久化（fsync)

   物理上是在`文件系统的page cache` 里面，也就是图中的黄色部分

3. 持久化到磁盘

   对应的是` hard disk`，也就是图中的绿色部分

> 日志写到 redo log buffer 是很快的，wirte 到 page cache 也差不多，但是持久化到磁盘的速度就慢多了



## innodb_flush_log_at_trx_commit 写入策略

InnoDB 提供了` innodb_flush_log_at_trx_commit `参数控制 redo log 的写入策略，它有三种可能取值

1. 设置为 0 时，表示每次事务提交时都只是把 redo log 留在 `redo log buffer `中 
2. 设置为 1 时，表示每次事务提交时都将 redo log 直接`持久化到磁盘`
3. 设置为 2 时，表示每次事务提交时都只是把 redo log 写到` page cache`



## redo log怎么持久化到磁盘

### 后台线程定时持久化

InnoDB 有一个`后台线程`，每隔 1 秒，就会把` redo log buffer `中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘

> 注意，事务执行中间过程的 redo log 也是直接写在 redo log buffer 中的，这些 redo log 也会被后台线程一起持久化到磁盘。也就是说，`一个没有提交的事务的redo log，也是可能已经持久化到磁盘的`



### 其他2种写入到磁盘的场景

还有两种场景会让一个`没有提交的事务的 redo log `写入到磁盘中

1. redo log buffer 占用的空间即将达到` innodb_log_buffer_size 一半`时，后台线程会`主动写盘`

   > 注意，由于这个事务并没有提交，所以这个写盘动作只是 write，而没有调用 fsync，也就是只留在了文件系统的 page cache

2. `并行的事务提交`时，顺带将这个事务的 redo log buffer 持久化到磁盘

   > 假设一个事务 A 执行到一半，已经写了一些 redo log 到 buffer 中，这时有另外一个线程的事务 B 提交，如果 innodb_flush_log_at_trx_commit 设置的是 1，那么按照这个参数的逻辑，事务 B 要把 redo log buffer 里的日志全部持久化到磁盘。这时，就会带上事务 A 在 redo log buffer 里的日志一起持久化到磁盘



这里需要说明的是，两阶段提交时，时序上 redo log 先 prepare， 再写 binlog，最后再把 redo log commit。

如果把` innodb_flush_log_at_trx_commit 设置成 1`，那么 redo log 在 prepare 阶段就要持久化一次，因为有一个崩溃恢复逻辑是要依赖于 prepare 的 redo log，再加上 binlog 来恢复的。



每秒一次后台轮询刷盘，再加上崩溃恢复这个逻辑，InnoDB 就认为 redo log 在 commit 的时候就不需要 fsync 了，只会 write 到文件系统的 page cache 中就够了。



## MySQL 的`“双 1”配置`

指的就是 `sync_binlog `和` innodb_flush_log_at_trx_commi`t 都设置成 1。

也就是说，一个事务完整提交前，`需要等待两次刷盘`，一次是 redo log（prepare 阶段），一次是 binlog



这意味着我从 MySQL 看到的 TPS 是每秒两万的话，每秒就会写四万次磁盘。但是，我用工具测试出来，磁盘能力也就两万左右，怎么能实现两万的 TPS？

答：用到`组提交（group commit）`机制。



# 六、组提交机制（group commit）

### 日志逻辑序列号(log sequence number，LSN)

LSN 是`单调递增`的，用来对应 redo log 的一个个写入点。每次写入长度为 length 的 redo log， LSN 的值就会加上 length



LSN 也会写到 InnoDB 的数据页中，来确保数据页不会被多次执行重复的 redo log。



关于 LSN 和 redo log、checkpoint 的关系，会在后面的文章中详细展开。



如图 3 所示，是三个并发事务 (trx1, trx2, trx3) 在 prepare 阶段，都写完 redo log buffer，持久化到磁盘的过程，对应的 LSN 分别是 50、120 和 160。

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/23_%E5%9B%BE3redolog%E7%BB%84%E6%8F%90%E4%BA%A4)



**从图中可以看到**

1. trx1 是第一个到达的，会被选为这组的 leader
2. 等 trx1 要开始写盘的时候，这个组里面已经有了三个事务，这时候 LSN 也变成了 160
3. trx1 去写盘的时候，带的就是 LSN=160，因此等 trx1 返回时，所有 LSN 小于等于 160 的 redo log，都已经被持久化到磁盘
4. 这时候 trx2 和 trx3 就可以直接返回了



所以，一次组提交里面，组员越多，节约磁盘 IOPS 的效果越好。但如果只有单线程压测，那就只能老老实实地一个事务对应一次持久化操作了。



在并发更新场景下，第一个事务写完 redo log buffer 以后，接下来这个 fsync 越晚调用，组员可能越多，节约 IOPS 的效果就越好。



为了让一次 fsync 带的组员更多，MySQL 有一个很有趣的优化：`拖时间`。



图4：两阶段提交

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210710153633.png)

图中，我把“写 binlog”当成一个动作。但实际上，写 binlog 是分成两步的：

1. 先把 binlog 从 binlog cache 中写到磁盘上的 binlog 文件
2. 调用 fsync 持久化。



MySQL 为了让组提交的效果更好，把 redo log 做 fsync 的时间拖到了步骤 1 之后。也就是说，上面的图变成了这样，图5

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/23_%E5%9B%BE5%E4%B8%A4%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4%E7%BB%86%E5%8C%96)

这么一来，binlog 也可以组提交了。在执行图 5 中第 4 步把 binlog fsync 到磁盘时，如果有多个事务的 binlog 已经写完了，也是一起持久化的，这样也可以减少 IOPS 的消耗。



不过通常情况下第 3 步执行得会很快，所以 binlog 的 write 和 fsync 间的间隔时间短，导致能集合到一起持久化的 binlog 比较少，因此 binlog 的组提交的效果通常不如 redo log 的效果那么好。



如果你想提升 binlog 组提交的效果，可以通过设置 binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count 来实现。

1. binlog_group_commit_sync_delay 参数，表示延迟多少微秒后才调用 fsync;
2. binlog_group_commit_sync_no_delay_count 参数，表示累积多少次以后才调用 fsync。



这两个条件是或的关系，也就是说只要有一个满足条件就会调用 fsync。

所以，当 binlog_group_commit_sync_delay 设置为 0 的时候，binlog_group_commit_sync_no_delay_count 也无效了。



**之前有同学在评论区问到，WAL 机制是减少磁盘写，可是每次提交事务都要写 redo log 和 binlog，这磁盘读写次数也没变少？**

现在就能理解了，WAL 机制主要得益于两个方面：

1. redo log 和 binlog 都是顺序写，磁盘的顺序写比随机写速度要快；
2. 组提交机制，可以大幅度降低磁盘的 IOPS 消耗。



**分析到这里，再来回答这个问题：如果你的 MySQL 现在出现了性能瓶颈，而且瓶颈在 IO 上，可以通过哪些方法来提升性能呢？**



针对这个问题，可以考虑以下三种方法：

1. 设置 binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count 参数，减少 binlog 的写盘次数。这个方法是基于“额外的故意等待”来实现的，因此可能会增加语句的响应时间，但没有丢失数据的风险。
2. 将 sync_binlog 设置为大于 1 的值（比较常见是 100~1000）。这样做的风险是，主机掉电时会丢 binlog 日志。
3. 将 innodb_flush_log_at_trx_commit 设置为 2。这样做的风险是，主机掉电的时候会丢数据。



**不建议把 innodb_flush_log_at_trx_commit 设置成 0。**

因为把这个参数设置成 0，表示 redo log 只保存在内存中，这样的话 MySQL 本身异常重启也会丢数据，风险太大。而 redo log 写到文件系统的 page cache 的速度也是很快的，所以将这个参数设置成 2 跟设置成 0 其实性能差不多，但这样做 MySQL 异常重启时就不会丢数据了，相比之下风险会更小。



# 七、问题

## 问题 1

执行一个 update 语句以后，我再去执行 hexdump 命令直接查看 ibd 文件内容，为什么没有看到数据有改变呢？



回答：这可能是因为 WAL 机制的原因。update 语句执行完成后，InnoDB 只保证写完了 redo log、内存，可能还没来得及将数据写到磁盘。



## 问题 2

为什么 binlog cache 是每个线程自己维护的，而 redo log buffer 是全局共用的？



回答：MySQL 这么设计的主要原因是，binlog 是不能“被打断的”。一个事务的 binlog 必须连续写，因此要整个事务完成后，再一起写到文件里。



而 redo log 并没有这个要求，中间有生成的日志可以写到 redo log buffer 中。redo log buffer 中的内容还能“搭便车”，其他事务提交的时候可以被一起写到磁盘中。



## 问题 3

事务执行期间，还没到提交阶段，如果发生 crash 的话，redo log 肯定丢了，这会不会导致主备不一致呢？



回答：不会。因为这时候 binlog 也还在 binlog cache 里，没发给备库。crash 以后 redo log 和 binlog 都没有了，从业务角度看这个事务也没有提交，所以数据是一致的。



## 问题 4

如果 binlog 写完盘以后发生 crash，这时候还没给客户端答复就重启了。等客户端再重连进来，发现事务已经提交成功了，这是不是 bug？



回答：不是。



你可以设想一下更极端的情况，整个事务都提交成功了，redo log commit 完成了，备库也收到 binlog 并执行了。但是主库和客户端网络断开了，导致事务成功的包返回不回去，这时候客户端也会收到“网络断开”的异常。这种也只能算是事务成功的，不能认为是 bug。



实际上数据库的 crash-safe 保证的是：

1. 如果客户端收到事务成功的消息，事务就一定持久化了；
2. 如果客户端收到事务失败（比如主键冲突、回滚等）的消息，事务就一定失败了；
3. 如果客户端收到“执行异常”的消息，应用需要重连后通过查询当前状态来继续后续的逻辑。此时数据库只需要保证内部（数据和日志之间，主库和备库之间）一致就可以了。


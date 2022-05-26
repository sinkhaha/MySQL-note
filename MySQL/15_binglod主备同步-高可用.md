## 

# 主备的基本原理

## 主备切换流程

M-S架构

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/24_MySQL%20%E4%B8%BB%E5%A4%87%E5%88%87%E6%8D%A2%E6%B5%81%E7%A8%8B)



状态 1 中：客户端的`读写`都直接访问节点 A，而节点 B 是 A 的`备库`

> 节点 B 和 A 的数据是相同的，B节点没有直接被访问

状态 2：主备切换后的状态，这时客户端`读写`访问的都是节点 B，而节点 A 是 B 的备库



**建议把备库节点（节点B）设置成`只读（readonly）模式`。因为**

1. 有时一些运营类的查询语句会被放到备库上去查，设置为只读可以`防止误操作`
2. 防止切换逻辑有 bug，比如切换过程中`出现双写`，造成主备不一致
3. 可以用 readonly 状态，来判断节点的角色



**把备库设置成只读了，还怎么跟主库保持同步更新呢？**

因为 readonly 设置对超级 (super) 权限用户是`无效的`，而用于`同步更新的线程`，就拥有超级权限。



## 主备数据同步

图中是`一个 update语句在节点 A 执行，然后同步到节点 B` 的完整流程图

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/24_%E4%B8%BB%E5%A4%87%E6%B5%81%E7%A8%8B%E5%9B%BE)

1. 主库接收到客户端的更新请求后，执行内部事务的更新逻辑，同时写 binlog

2. 备库 B 跟主库 A 之间维持了一个`长连接`。主库 A 内部有一个线程，专门用于服务备库 B 的这个长连接




**一个事务日志同步的完整过程**

1. 在备库 B 上通过` change master` 命令，设置主库 A 的 IP、端口、用户名、密码，以及要从哪个位置开始请求 binlog，这个位置包含`文件名`和`日志偏移量`
2. 在备库 B 上执行 `start slave` 命令，这时候备库会启动两个线程，就是图中的` io_thread 和 sql_thread`。其中 io_thread 负责与主库建立连接
3. 主库 A 校验完用户名、密码后，`开始按照备库 B 传过来的位置`，从本地读取 binlog，发给 B
4. 备库 B 拿到 binlog 后，写到`本地文件`，称为`中转日志（relay log）`
5. sql_thread 读取中转日志，解析出日志里的命令，并执行

> 后来由于多线程复制方案的引入，sql_thread 演化成为了多个线程



# binlog的3种格式对比

1. statement
2. row
3. mixed，其实它就是前两种格式的混合



## 初始化表

创建了一个表，并初始化几行数据

```sql
 CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `t_modified`(`t_modified`)
) ENGINE=InnoDB;

insert into t values(1,1,'2018-11-13');
insert into t values(2,2,'2018-11-12');
insert into t values(3,3,'2018-11-11');
insert into t values(4,4,'2018-11-10');
insert into t values(5,5,'2018-11-09');
```



## 例子

**举例说明 delete删除一行数据的binlog 是怎么记录的**

注意，下面这个语句包含注释，如果你用 MySQL客户端来做这个实验的话，要记得加 -c 参数，否则客户端会自动去掉注释。

```sql
delete from t /*comment*/  where a>=4 and t_modified<='2018-11-10' limit 1;
```



### statement格式(不推荐)

当 binlog_format=statement 时，binlog 里面记录的就是` SQL语句的原文`

```sql
-- 查看 binlog 中的内容
show binlog events in 'master.000001';
```

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210710160727.png)

**binlog内容解析**

1. 第一行 `SET @@SESSION.GTID_NEXT='ANONYMOUS'`先忽略（GTID技术）

2. 第二行是一个` BEGIN`，跟第四行的` commit `对应，表示中间是一个事务

3. 第三行就是真正执行的语句了。可以看到，binlog记录了 SQL语句的原文

   > 在真实执行的 delete 命令之前，还有一个`“use ‘test’”`命令。这条命令是 MySQL 根据当前要操作的表所在的数据库，自行添加的。这样做可以保证日志传到备库去执行时，不论当前的工作线程在哪个库里，都能够正确地更新到 test 库的表 t

4. 最后一行是一个 COMMIT。你可以看到里面写着 xid=61



**注意：上面的delete语句是不安全的**

来看一下delete 执行 warnings图

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/image-20210710160847229.png)

可以看到，运行这条 delete 命令产生了一个 warning，`原因是当前 binlog 设置的是 statement 格式`，并且语句中有 limit，所以这个命令可能是` unsafe` 不安全的。



**因为delete带 limit，很可能会出现主备数据不一致的情况：**

1. 如果 delete 语句使用的是索引 a，那么会根据索引 a 找到第一个满足条件的行，也就是说删除的是 a=4 这一行
2. 但如果使用的是索引 t_modified，那么删除的就是 t_modified='2018-11-09’也就是 a=5 这一行



**由于 statement 格式下，记录到 binlog 里的是语句原文，因此可能会出现这样一种情况：**

在主库执行这条 SQL 语句时，用的是索引 a；而在备库执行这条 SQL 语句的时候，却使用了索引 t_modified。因此，MySQL 认为这样写是有风险的。



### row格式(推荐)

把 binlog 的格式改为 binlog_format=‘row’， 一样执行以上的delete语句。



row 格式 binlog 示例图，图5

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/image-20210710160937579.png)

可以看到，与statement 格式的 binlog 相比，前后的 BEGIN 和 COMMIT 是一样的。但row 格式的 binlog 里没有了 SQL 语句的原文，而是替换成了两个 event：`Table_map 和 Delete_rows`

1. Table_map event：用于说明接下来要操作的表是 test 库的表 t
2. Delete_rows event：用于定义删除的行为



其实，通过图中是看不到详细信息的，还需要借助 mysqlbinlog 工具，用下面这个命令解析和查看 binlog 中的内容。因为图中的信息显示，这个事务的 binlog 是从 8900 这个位置开始的，所以可以用 start-position 参数来指定从这个位置的日志开始解析。

```sql
mysqlbinlog  -vv data/master.000001 --start-position=8900;
```

row 格式 binlog 示例的详细信息图，图6

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/image-20210710161110602%E7%9A%84%E5%89%AF%E6%9C%AC.png)

**从这个图中，可以看到以下几个信息：**

1. server id 1，表示这个事务是在 server_id=1 的这个库上执行的。

2. 每个 event 都有 CRC32 的值，这是因为我把参数 binlog_checksum 设置成了 CRC32。

3. Table_map event 跟在图 5 中看到的相同，显示了接下来要打开的表，map 到数字 226。

   > 现在我们这条 SQL 语句只操作了一张表，如果要操作多张表呢？每个表都有一个对应的 Table_map event、都会 map 到一个单独的数字，用于区分对不同表的操作。

4. 我们在 mysqlbinlog 的命令中，使用了 -vv 参数是为了把内容都解析出来，所以从结果里面可以看到各个字段的值（比如，@1=4、 @2=4 这些值）。

5. binlog_row_image 的默认配置是 FULL，因此 Delete_event 里面，包含了删掉的行的所有字段的值。如果把 binlog_row_image 设置为 MINIMAL，则只会记录必要的信息，在这个例子里，就是只会记录 id=4 这个信息。

6. 最后的 Xid event，用于表示事务被正确地提交了。



当 binlog_format 使用 row 格式时，binlog 里面记录了真实删除行的主键 id，这样 binlog 传到备库去的时候，就肯定会删除 id=4 的行，不会有statement格式那样主备走到不同的索引导致删除不同行的问题。



### mixed格式的binlog

**为什么会有 mixed 这种 binlog 格式的存在场景？**

mixed 格式可以利用 statment 格式的优点，同时又避免了数据不一致的风险。



1. 因为有些 statement 格式的 binlog 可能会导致主备不一致，所以要使用 row 格式

2. 但 row 格式的缺点是，`很占空间`

   > 比如你用一个 delete 语句删掉 10 万行数据，用 statement 的话就是一个 SQL 语句被记录到 binlog 中，占用几十个字节的空间。但如果用 row 格式的 binlog，就要把这 10 万条记录都写到 binlog 中。这样做，不仅会占用更大的空间，同时写 binlog 也要耗费 IO 资源，影响执行速度。

3. 所以，MySQL 就取了个折中方案，也就是有了 mixed 格式的 binlog

   > mixed格式的意思是，MySQL 自己会判断这条 SQL 语句是否可能引起主备不一致，如果有可能，就用 row 格式，否则就用 statement 格式。
   >
   > 
   >
   > 比如上面的delete语句例子，设置为 mixed 后，就会记录为 row 格式；而如果执行的语句去掉 limit 1，就会记录为 statement 格式。



**当然，现在越来越多的场景要求把 MySQL 的 binlog 格式设置成 row。这么做的理由有很多，又一个可以直接看出来的好处：恢复数据。**



## row格式下数据恢复问题

接下来，分别从 delete、insert 和 update 这三种 SQL 语句的角度，来看看row格式下数据恢复的问题。



**通过图 6 可以看出来**

1. 执行delete语句时，row 格式的 binlog 也会把被删掉的行的整行信息保存起来。

   > 所以，如果误执行完一条 delete 语句，可以直接把 binlog 中记录的 delete 语句转成 insert，把被错删的数据插入回去就可以恢复了。

2. 执行 insert 语句，row 格式的binlog 里会记录所有的字段信息。这些信息可以用来精确定位刚刚被插入的那一行

   > 所以，如果误执行了insert语句，直接把 insert 语句转成 delete 语句，删除掉这被误插入的一行数据就可以了。

3. 执行update语句，在row格式的binlog里会记录修改前整行的数据和修改后的整行数据

   > 所以，如果误执行了 update 语句，只需要把这个 event 前后的两行信息对调一下，再去数据库里面执行，就能恢复这个更新操作了。

MariaDB 的`Flashback工具`就是基于上面介绍的原理来回滚数据的。



**binlog恢复数据的问题**

虽然 mixed 格式的 binlog 现在已经用得不多了，但这里我还是要再借用一下 mixed 格式来说明一个问题，来看一下这条 SQL 语句：

```sql
mysql> insert into t values(10,10, now());
```



如果把 binlog 格式设置为 mixed，你觉得 MySQL 会把它记录为 row 格式还是 statement 格式呢？

看一下这条语句执行的效果，mixed 格式和 now()图

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/image-20210710161348070.png)

可以看到，MySQL 用的居然是 statement 格式。你一定会奇怪，如果这个 binlog 过了 1 分钟才传给备库的话，那主备的数据不就不一致了吗？



接下来，再用 mysqlbinlog 工具来看看：TIMESTAMP 命令图![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/image-20210710161409515.png)

从图中的结果可以看到，原来 binlog 在记录 event 时，多记了一条命令：SET TIMESTAMP=1546103491。它用 SET TIMESTAMP 命令约定了接下来的 now() 函数的返回时间。



因此，不论这个 binlog 是 1 分钟之后被备库执行，还是 3 天后用来恢复这个库的备份，这个 insert 语句插入的行，值都是固定的。也就是说，通过这条 SET TIMESTAMP 命令，MySQL 就确保了主备数据的一致性。



有人在重放 binlog 数据时，是这么做的：用 mysqlbinlog 解析出日志，然后把里面的 statement 语句直接拷贝出来执行。

> 这个方法是有风险的。因为有些语句的执行结果是依赖于上下文命令的，直接执行的结果很可能是错误的。



所以，用 binlog 来恢复数据的标准做法是，用 mysqlbinlog 工具解析出来，然后把解析结果整个发给 MySQL 执行。类似下面的命令：mysqlbinlog master.000001 --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;

```sql

mysqlbinlog master.000001  --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;
```

这个命令的意思是，将 master.000001 文件里面从第 2738 字节到第 2973 字节中间这段内容解析出来，放到 MySQL 去执行。



# 双主循环复制问题

binlog 的特性确保了在备库执行相同的 binlog，可以得到与主库相同的状态。因此，我们可以认为正常情况下M-S架构下的主备的数据是一致的。



实际生产上使用比较多的是双 M 结构，也就是图 9 所示的主备切换流程。



图 9 MySQL 主备切换流程 -- 双 M 结构

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210710161518.png)

双 M 结构和 M-S 结构，其实区别只是多了一条线，即：节点 A 和 B 之间总是互为主备关系。这样在切换的时就不用再修改主备关系。



但是，双 M 结构还有一个问题需要解决。



业务逻辑在节点 A 上更新了一条语句，然后再把生成的 binlog 发给节点 B，节点 B 执行完这条更新语句后也会生成 binlog。（我建议你把参数 log_slave_updates 设置为 on，表示备库执行 relay log 后生成 binlog）。



那么，如果节点 A 同时是节点 B 的备库，相当于又把节点 B 新生成的 binlog 拿过来执行了一次，然后节点 A 和 B 间，会不断地循环执行这个更新语句，也就是循环复制了。这个要怎么解决呢？



从上面的图 6 中可以看到，MySQL 在 binlog 中记录了这个命令第一次执行时所在实例的 server id。因此，我们可以用下面的逻辑，来解决两个节点间的循环复制的问题：

1. 规定两个库的 server id 必须不同，如果相同，则它们之间不能设定为主备关系；
2. 一个备库接到 binlog 并在重放的过程中，生成与原 binlog 的 server id 相同的新的 binlog；
3. 每个库在收到从自己的主库发过来的日志后，先判断 server id，如果跟自己的相同，表示这个日志是自己生成的，就直接丢弃这个日志。



按照这个逻辑，如果我们设置了双 M 结构，日志的执行流就会变成这样：

1. 从节点 A 更新的事务，binlog 里面记的都是 A 的 server id；
2. 传到节点 B 执行一次以后，节点 B 生成的 binlog 的 server id 也是 A 的 server id；
3. 再传回给节点 A，A 判断到这个 server id 与自己的相同，就不会再处理这个日志。所以，死循环在这里就断掉了



# 主备延迟

与数据同步有关的时间点主要包括以下三个：

1. 主库 A 执行完成一个事务，写入 binlog，把这个时刻记为 T1
2. 之后传给备库 B，把备库 B 接收完这个 binlog 的时刻记为 T2
3. 备库 B 执行完成这个事务，把这个时刻记为 T3



## 什么是主备延迟

就是同一个事务，在`备库执行完成的时间` 和 `主库执行完成的时间` 之间的差值，也就是 `T3-T1`



## 查看延迟了多少秒

在备库上执行` show slave status `命令，返回结果会显示 `seconds_behind_master`，用于表示当前备库延迟了多少秒，单位是秒



**seconds_behind_master 的计算方法**

1. 每个事务的 binlog 里面都有一个时间字段，用于`记录主库上写入的时间`
2. 备库取出当前正在执行的事务的时间字段的值，计算它与当前系统时间的差值，得到 seconds_behind_master



**如果主备库机器的系统时间设置不一致，会不会导致主备延迟的值不准？**

不会

> 因为，备库连接到主库时，会通过执行 `SELECT UNIX_TIMESTAMP() 函数`来获得当前主库的系统时间。如果这时候发现主库的系统时间与自己不一致，备库在执行 `seconds_behind_master `计算的时候会自动扣掉这个差值



在网络正常时，日志从主库传给备库所需的时间是很短的，即 `T2-T1 `的值是非常小的。

也就是说，网络正常情况下，主备延迟的`主要来源`是备库接收完 binlog 和执行完这个事务之间的时间差。



**主备延迟最直接的表现**

`备库消费中转日志（relay log）的速度`，比主库生产 binlog 的速度要慢



# 主备延迟的来源

### 第1种：机器性能

**有些部署条件下，备库所在机器的性能要比主库所在的机器性能差**

> 因为主备可能发生切换，备库随时可能变成主库，所以主备库选用相同规格的机器，并且做对称部署，是现在比较常见的情况



### 第2种：备库压力大

**即备库的压力大**

一般的想法是，主库既然提供了写能力，那么备库可以提供一些读能力。或者一些运营后台需要的分析语句，不能影响正常业务，所以只能在备库上跑。结果就是，`备库上的查询耗费了大量的 CPU 资源`，影响了同步速度，造成主备延迟。



这种情况，一般可以这么处理：

1. 一主多从

   > 除了备库外，可以多接几个从库，让这些从库来分担读的压力

2. 通过 binlog 输出到外部系统，比如 Hadoop 这类系统，让外部系统提供统计类查询的能力



### 第3种：大事物

**即大事务**

因为`主库上必须等事务执行完成才会写入 binlog`，再传给备库。所以，如果一个主库上的语句执行 10 分钟，那这个事务很可能就会导致从库延迟 10 分钟。



**典型的大事务场景**

1. 一次性地用 delete 语句删除太多数据
2. 另一种典型的大事务场景，就是大表 DDL

> 这个场景处理方案就是，计划内的 DDL，建议使用 gh-ost 方案



### 第4种：备库的并行复制能力

备库的并行复制能力



# 高可用

由于主备延迟的存在，所以在主备切换时，就相应的有不同的策略



## 可靠性优先策略

在双 M 结构下，从状态 1 到状态 2 切换的详细过程是这样的：

1. 判断备库 B 现在的 seconds_behind_master，如果小于某个值（比如 5 秒）继续下一步，否则持续重试这一步
2. 把主库 A 改成`只读状态`，即把 readonly 设置为 true
3. 判断备库 B 的 seconds_behind_master 的值，直到这个值变成 `0 `为止
4. 把备库 B 改成可读写状态，也就是把 readonly 设置为 false
5. 把`业务请求`切到备库 B



这个切换流程，一般是由专门的` HA 系统`来完成的，暂时称之为`可靠性优先流程`

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/25_MySQL%E5%8F%AF%E9%9D%A0%E6%80%A7%E4%BC%98%E5%85%88%E4%B8%BB%E5%A4%87%E5%88%87%E6%8D%A2%E6%B5%81%E7%A8%8B)

备注：图中的 SBM，是 seconds_behind_master 参数的简写



这个切换流程中是有`不可用时间`的

> 因为在步骤 2 之后，主库 A 和备库 B 都处于 readonly 状态，也就是说这时系统处于不可写状态，直到步骤 5 完成后才能恢复



在这个不可用状态中，`比较耗费时间的是步骤 3`，可能需要耗费好几秒的时间。这也是为什么需要在步骤 1 先做判断，确保 seconds_behind_master 的值足够小。



试想如果一开始主备延迟就长达 30 分钟，而不先做判断直接切换的话，系统的不可用时间就会长达 30 分钟，这种情况一般业务都是不可接受的。



当然，系统的不可用时间，是由这个数据可靠性优先的策略决定的。你也可以选择可用性优先的策略，来把这个不可用时间几乎降为 0。



## 可用性优先策略

如果我强行把步骤 4、5 调整到最开始执行，也就是说不等主备数据同步，`直接把连接切到备库 B`，并且让备库 B 可以读写，那么系统几乎就没有不可用时间了。



我们把这个切换流程，暂时称作`可用性优先流程`。这个切换流程的代价，就是可能出现数据不一致的情况。



接下来分享一个`可用性优先流程`产生数据不一致的例子。假设有一个表 t

```sql
CREATE TABLE `t` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `c` int(11) unsigned DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

insert into t(c) values(1),(2),(3);
```

这个表定义了一个`自增主键 id`，初始化数据后，主库和备库上都是 3 行数据。



接下来，业务人员要继续在表 t 上执行两条插入语句的命令，依次是：

```sql
insert into t(c) values(4);
insert into t(c) values(5);
```



假设，`现在主库上其他的数据表有大量的更新`，导致主备延迟达到 5 秒。在插入一条 c=4 的语句后，发起了主备切换。



图 3 是可用性优先策略，且` binlog_format=mixed `时的切换流程和数据结果

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/25_%E5%8F%AF%E7%94%A8%E6%80%A7%E4%BC%98%E5%85%88%E7%AD%96%E7%95%A5%E4%B8%94%20binlog_format=mixed)



**分析下这个切换流程：**

1. 步骤 2 中，主库 A 执行完 insert 语句，插入了一行数据（4,4），之后开始进行主备切换
2. 步骤 3 中，由于主备之间有 5 秒的延迟，所以备库 B 还没来得及应用“插入 c=4”这个中转日志，就开始接收客户端“插入 c=5”的命令
3. 步骤 4 中，备库 B 插入了一行数据（4,5），并且把这个 binlog 发给主库 A
4. 步骤 5 中，备库 B 执行“插入 c=4”这个中转日志，插入了一行数据（5,4）；而直接在备库 B 执行的“插入 c=5”这个语句，传到主库 A，就插入了一行新数据（5,5）



最后的结果就是，`主库 A 和备库 B 上出现了两行不一致的数据`。可以看到，这个数据不一致，是由可用性优先流程导致的。



**那么，如果我还是用可用性优先策略，但设置 binlog_format=row，情况又会怎样呢？**

因为 row 格式在记录 binlog 的时候，会记录新插入的行的所有字段值，所以最后只会有一行不一致。而且，两边的主备同步的应用线程会报错` duplicate key error `并停止。也就是说，这种情况下，备库 B 的 (5,4) 和主库 A 的 (5,5) 这两行数据，都不会被对方执行。



图 4 中我画出了详细过程，你可以自己再分析一下。

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/25_%E5%9B%BE4%E5%8F%AF%E7%94%A8%E6%80%A7%E4%BC%98%E5%85%88%E7%AD%96%E7%95%A5%E4%B8%94%20binlog_format=row)



从上面的分析中，你可以看到一些结论：

1. 使用 row 格式的 binlog 时，数据不一致的问题更容易被发现。而使用 mixed 或者 statement 格式的 binlog 时，数据很可能悄悄地就不一致了。如果你过了很久才发现数据不一致的问题，很可能这时的数据不一致已经不可查，或者连带造成了更多的数据逻辑不一致。
2. 主备切换的可用性优先策略会导致数据不一致。因此，大多数情况下，我都建议你使用可靠性优先策略。毕竟对数据服务来说的话，数据的可靠性一般还是要优于可用性的。



**但事无绝对，有没有哪种情况数据的可用性优先级更高呢？**

答案是，有的。



我曾经碰到过这样的一个场景：

* 有一个库的作用是记录操作日志。这时候，如果数据不一致可以通过 binlog 来修补，而这个短暂的不一致也不会引发业务问题。
* 同时，业务系统依赖于这个日志写入逻辑，如果这个库不可写，会导致线上的业务操作无法执行。



这时候，你可能就需要选择先强行切换，事后再补数据的策略。



当然，事后复盘的时候，我们想到了一个改进措施就是，让业务逻辑不要依赖于这类日志的写入。也就是说，日志写入这个逻辑模块应该可以降级，比如写到本地文件，或者写到另外一个临时库里面。



这样的话，这种场景就又可以使用可靠性优先策略了。



接下来我们再看看，按照可靠性优先的思路，异常切换会是什么效果？



假设，主库 A 和备库 B 间的主备延迟是 30 分钟，这时候主库 A 掉电了，HA 系统要切换 B 作为主库。我们在主动切换的时候，可以等到主备延迟小于 5 秒的时候再启动切换，但这时候已经别无选择了。

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/25_%E5%9B%BE5%E5%8F%AF%E9%9D%A0%E6%80%A7%E4%BC%98%E5%85%88%E7%AD%96%E7%95%A5%E4%B8%BB%E5%BA%93%E4%B8%8D%E5%8F%AF%E7%94%A8)

采用可靠性优先策略的话，你就必须得等到备库 B 的 seconds_behind_master=0 之后，才能切换。但现在的情况比刚刚更严重，并不是系统只读、不可写的问题了，而是系统处于完全不可用的状态。因为，主库 A 掉电后，我们的连接还没有切到备库 B。



**那能不能直接切换到备库 B，但是保持 B 只读呢？**

这样也不行。



因为，这段时间内，中转日志还没有应用完成，如果直接发起主备切换，客户端查询看不到之前执行完成的事务，会认为有“数据丢失”。



虽然随着中转日志的继续应用，这些数据会恢复回来，但是对于一些业务来说，查询到“暂时丢失数据的状态”也是不能被接受的。



在满足数据可靠性的前提下，MySQL 高可用系统的可用性，是依赖于主备延迟的。延迟的时间越小，在主库故障的时候，服务恢复需要的时间就越短，可用性就越高。


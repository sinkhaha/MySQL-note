 

# MySQL两个 kill 命令

1. 一个是 `kill query + 线程 id`

   > 表示终止这个线程中正在执行的语句

2. 一个是 `kill connection + 线程 id`，这里 connection 可缺省

   > 表示断开这个线程的连接，当然如果这个线程有语句正在执行，也是要`先停止正在执行的语句`的



# 背景

有时使用了 kill 命令，`却没能断开这个连接`。再执行 show processlist 命令，看到这条语句的 Command 列显示的是` Killed`



1. 其实大多数情况下，`kill query/connection `命令是有效的

   > 比如，执行一个查询的过程中，发现执行时间太久，要放弃继续查询，这时我们就可以用 kill query 命令，终止这条查询语句

2. 还有一种情况是，语句处于`锁等待`的时候，直接使用 kill 命令也是有效的

   > 如下session C 执行 kill query 以后，session B 几乎同时就提示了语句被中断

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210711212945.png)



# 收到 kill 以后线程做什么

kill 并不是`马上停止`的意思，而是告诉执行线程说，这条语句已经不需要继续执行了，可以开始“执行停止的逻辑了”

> 其实，这跟 Linux 的 kill 命令类似，kill -N pid 并不是让进程直接停止，而是给进程发一个信号，然后进程处理这个信号，进入终止逻辑。只是对于 MySQL 的 kill 命令来说，不需要传信号量参数，就只有“停止”这个命令



实现上，当用户执行` kill query thread_id_B `时，MySQL 里处理 kill 命令的线程做了两件事：

1. 把 session B 的运行状态改成` THD::KILL_QUERY`(将变量 killed 赋值为` THD::KILL_QUERY`)

   > 此时线程 B 并不知道这个状态变化，还是会继续等待（如果没有收到信号，会一直等待）

2. 给 session B 的执行线程发算送一个信号

   > 发一个信号的目的，就是让 session B `退出等待`，线程继续执行时会判断线程状态，进而处理这个 THD::KILL_QUERY 状态



# kill 不掉的例子

首先，执行 `set global innodb_thread_concurrency=2`，将 InnoDB 的并发线程上限数设置为 2



然后，执行下面的序列：

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210723210512.png)

1. sesssion C 执行的时候被堵住了
2. 但是 session D 执行的 `kill query C 命令却没什么效果`
3. 直到 session E 执行了` kill connection 命令`，才断开了 session C 的连接，提示“Lost connection to MySQL server during query”
4. 但是这时候，如果在 session E 中执行 show processlist，你就能看到下面这个图

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210711214401.png)

这时`id=12 `这个线程的 Commnad 列显示的是` Killed`。也就是说，客户端虽然断开了连接，但实际上服务端上这条语句还在执行过程中。



**为什么session D在执行 `kill query 命令`时，为什么session C的查询语句不退出呢？**

1. 在实现上，`等行锁时`，使用的是`pthread_cond_timedwait `函数，这个等待状态可以`被唤醒`。但是，在这个例子里，12 号线程的等待逻辑是这样的：每 10 毫秒判断一下是否可以进入 InnoDB 执行，如果不行，就调用 nanosleep 函数进入 sleep 状态。

2. 也就是说，虽然 12 号线程的状态已经被`设置成了 KILL_QUERY`，但是在这个等待进入 InnoDB 的循环过程中，`并没有去判断线程的状态`，因此根本不会进入终止逻辑阶段



**而当 session E 执行 `kill connection `命令时，是样的**

1. 把 12 号线程状态设置为 `KILL_CONNECTION`
2. 关掉 12 号线程的网络连接。因为有这个操作，所以你会看到，这时候 session C 收到了断开连接的提示



**show processlist 时， Command 列为什么显示为 killed 呢**

因为在执行 show processlist 的时候，有一个特别的逻辑：

```
如果一个线程的状态是KILL_CONNECTION，就把Command列显示成Killed。
```

所以其实，`即使是客户端退出了`，这个线程的状态仍然是在`等待中`。那这个线程什么时候会退出呢？

答案是，`只有等到满足进入 InnoDB 的条件后`，session C 的查询语句`继续执行`，然后才`有可能判断到线程状态`已经变成了 KILL_QUERY 或者 KILL_CONNECTION，再进入终止逻辑阶段。



**小结**

这个例子是 kill 无效的第一类情况，即：线程没有执行到判断线程状态的逻辑

> 跟这种情况相同的，还有由于 IO 压力过大，读写 IO 的函数一直无法返回，导致不能及时判断线程的状态

另一类情况是，终止逻辑耗时较长

> 这时候，从 show processlist 结果上看也是 Command=Killed，需要等到终止逻辑完成，语句才算真正完成。这类情况，比较常见的场景有以下几种：
>
> 1. 超大事务执行期间被 kill。这时候，回滚操作需要对事务执行期间生成的所有新数据版本做回收操作，耗时很长
> 2. 大查询回滚。如果查询过程中生成了比较大的临时文件，加上此时文件系统压力大，删除临时文件可能需要等待 IO 资源，导致耗时较长
> 3. DDL 命令执行到最后阶段，如果被 kill，需要删除中间过程的临时文件，也可能受 IO 资源影响耗时较久



# **另外两个关于客户端的误解**

**第一个误解是：如果库里面的表特别多，连接就会很慢**



## 客户端慢

有些线上的库，会包含很多表。这时每次用客户端连接都会卡在下面这个界面上，其实不是连接慢，也不是服务端慢，而是客户端慢

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210711214900.png)



当使用`默认参数连接`时，MySQL 客户端会提供一个`本地库名和表名补全`的功能（输入库名或者表名前缀，可以使用 Tab 键自动补全表名或者显示提示）



为了实现这个功能，客户端在连接成功后，需要多做一些操作：

1. 执行 show databases
2. 切到 db1 库，执行 show tables
3. 把这两个命令的结果用于构建一个本地的哈希表

最花时间的就是`第三步在本地构建哈希表`的操作。所以，当一个库中的表个数非常多的时候，这一步就会花比较长的时间。



图中提示也说 ，如果在连接命令中`加上 -A`，就可以`关掉这个自动补全`的功能，然后客户端就可以快速返回了。



除了加 -A 以外，加`–quick(或者简写为 -q) `参数，也可以跳过这个阶段



## -quick

**–quick 是一个更容易引起误会的参数，也是关于客户端常见的一个误解**

这个参数可能会降低服务端的性能



MySQL 客户端发送请求后，`接收服务端返回结果`的方式有两种：

1. 一种是`本地缓存`，也就是在本地开一片内存，先把结果存起来

   > 如果你用 API 开发，对应的就是 mysql_store_result 方法

2. 另一种是不缓存，`读一个处理一个`

   > 如果你用 API 开发，对应的就是 mysql_use_result 方法



MySQL 客户端`默认采用第一种方式`，而如果加上`–quick `参数，就会使用第二种不缓存的方式



采用不缓存的方式时，如果本地处理得慢，就会`导致服务端发送结果被阻塞`，因此`会让服务端变慢`。



既然这样，为什么要给这个参数取名叫作 quick 呢？因为使用这个参数可以达到以下三点效果：

1. 第一点，就是前面提到的，跳过表名自动补全功能
2. 第二点，mysql_store_result 需要申请本地内存来缓存查询结果，如果查询结果太大，会耗费较多的本地内存，可能会影响客户端本地机器的性能
3. 第三点，是不会把执行命令记录到本地的命令历史文件

所以，–quick 参数的意思，是`让客户端变得更快`


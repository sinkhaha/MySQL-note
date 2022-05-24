# 创建两个表 t1 和 t2

```sql
CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;

// 存储过程往t2表插入1000行
drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();

// 创建t1表，插入100行数据
create table t1 like t2;
insert into t1 (select * from t2 where id<=100)
```

这两个表都有一个`主键索引 id `和`一个索引 a`，字段 b 上无索引

# 两种join算法

## Index Nested-Loop Join算法

**该算法跟写程序时的`嵌套查询`类似，并且可以用上被驱动表的索引，所以称为“Index Nested-Loop Join”，简称 NLJ**



### 例子

**如下语句**

```sql
-- a字段有索引
select * from t1 straight_join t2 on (t1.a=t2.a);
```

* straight_join语句，固定左边的t1是驱动表(驱动表是主动发起查询的表)，右边的t2是被驱动表

* 如果使用join，MySQL优化器可能会选择表 t1 或 t2 作为驱动表



**explain结果**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210723210632.png)



### NLJ执行过程

join 过程用上了`被驱动表t2`的字段a的索引，执行流程是：

1. 从表 t1 中`读入一行数据 R`
2. 从数据行 R 中，`取出a字段`到`表 t2` 里去查找
3. 取出表 t2 中满足条件的行，跟 R 组成一行，作为`结果集`的一部分
4. 重复执行步骤 1 到 3，直到表 t1 的末尾循环结束



这个过程是`先遍历表 t1`，然后根据从表 t1 中取出的每行数据中的` a值`，去表 t2 中查找满足条件的记录。



**NLJ对应的流程图如下所示，图2**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210723210652.png)

**扫描行数**

1. 对驱动表 t1 做了`全表扫描`，这个过程需要扫描 100 行
2. 而对于每一行 R，根据 a 字段去表 t2 查找，走的是`树搜索`过程。由于我们构造的表数据都是一一对应的，因此每次的搜索过程都`只扫描一行`，也是总共扫描 100 行
3. 所以，整个执行流程，总扫描行数是 200



**假设不使用 join，只用`单表查询`怎么实现上面join的流程**

1. 执行`select * from t1`，查出表 t1 的所有数据，这里有 100 行
2. 循环遍历这 100 行数据：
   * 从每一行 R 取出字段 a 的值 $R.a；
   * 执行`select * from t2 where a=$R.a；`
   * 把返回的结果和 R 构成结果集的一行

这个过程也是扫描了 200 行，但是总共执行了 101 条语句，`比直接 join 多了 100 次交互`。除此之外，客户端还要自己拼接 SQL 语句和结果。所以使用 join 语句，性能比强行拆成多个单表执行 SQL 语句的性能要好



### 怎么选择驱动表

这个 join 语句中，驱动表是`走全表扫描`，而被驱动表是`走树搜索`

> * 假设`被驱动表的行数是M`。每次在被驱动表查一行数据，要先搜索索引 a，再搜索`主键索引`。每次搜索一棵树近似复杂度是以 2 为底的 M 的对数，记为 log2M，所以在`被驱动表上查一行的时间复杂度是 2*log2M`
>
> * 假设`驱动表`的行数是 N，执行过程就要扫描驱动表` N行`，然后对于每一行，到被驱动表上匹配一次。
>
> 因此整个执行过程，近似复杂度是 N + N*2*log2M



显然，N 对扫描行数的影响更大，因此使用join语句时，应该让`小表`来做驱动表。

> 这个结论的前提是“可以使用被驱动表的索引”。被驱动表要有索引，那么搜索才会走N叉树搜索，否则也只能全表扫描



## Block Nested-Loop Join算法

BNL算法在被驱动表上没有用上索引的情况使用，简称BNL



### 例子

如下语句，t2表的b字段没有索引，此时被驱动表用不上索引

```sql
select * from t1 straight_join t2 on (t1.a=t2.b);
```

**explain 结果**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210712214841.png)





### BNL执行过程

1. 把表 t1 的数据读入`线程内存 join_buffer `中，由于这个语句中写的是 select *，因此是把`整个表 t1 `放入了内存
2. 扫描表 t2，把表t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回



**BNL执行流程如下图，图3**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210712214751.png)



**扫描行数**

在这个过程中，对表 t1 和 t2 都做了一次`全表扫描`，因此总的扫描行数是` 1100`



**判断次数**

由于 join_buffer 是以`无序数组`的方式组织的，因此对表 t2 中的每一行，都要`做100次判断`，总共需要在内存中做的判断次数是：`100*1000=10万次`



#### Simple Nested-Loop Join算法

注意：此算法也是用于用不上索引的情况，但MySQL没使用此算法，而是使用了BNL



由于表 t2 的`字段b上没有索引`，因此再用NLJ图 2 的执行流程时，每次到 t2 去匹配时，就要做一次`全表扫描`，这个算法叫做“Simple Nested-Loop Join”。



使用Simple Nested-Loop Join这个SQL请求就要扫描表 t2 多达 100 次，总共`扫描 100*1000=10万行`。虽然BNL的扫描行数也是10万行，从时间复杂度上来说，这两个算法是一样的。但是，BNL算法的这 10 万次判断是`内存操作`，速度上会快很多，性能也更好。



### 怎么选择驱动表

假设小表的行数是 N，大表的行数是 M，那么

1. 两个表都做一次全表扫描，所以总的扫描行数是 M+N
2. 内存中的判断次数是 M*N

这两个算式中的 M 和 N 没差别，因此这时选择大表还是小表做驱动表，执行耗时是一样的。



### 测试join_buffer放不下的情况

join_buffer 的大小是由参数` join_buffer_size `设定的，默认值是` 256k`。

如果t1表是个大表，join_buffer放不下表 t1 的所有数据的话，会进行`分段放`。



例如把 join_buffer_size 改成 1200，再执行：

```sql
select * from t1 straight_join t2 on (t1.a=t2.b);
```



**执行过程如下**

1. 扫描表 t1，`顺序读取`数据行放入 join_buffer 中，放完第 88 行 join_buffer 满了，继续第 2 步
2. 扫描表 t2，把` t2中的每一行`取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回
3. 清空 join_buffer
4. 继续扫描表 t1，顺序读取最后的 12 行数据放入 join_buffer 中，继续执行第 2 步

这个流程才体现出了这个算法名字中“Block”的由来，表示“分块去 join”。



**此时执行流程图如下 -- 两段，图 5**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210712215253.png)



图中的步骤 4 和 5，表示清空 join_buffer 再复用。



由于表 t1 被分成了`两次放入join_buffer 中`，导致`表 t2 会被扫描两次`。虽然分成两次放入 join_buffer，但是判断等值条件的次数还是不变的，依然是 (88+12)*1000=10 万次。



**在这种情况下驱动表的选择问题**

假设，驱动表的数据行数是 N，需要分 K 段才能完成算法流程，被驱动表的数据行数是 M。

> 注意，这里的 K 不是常数，N 越大K就会越大，因此把 K 表示为λ*N，显然λ的取值范围是 (0,1)。



所以，在这个算法的执行过程中：

1. 扫描行数是 N+λNM；
2. 内存判断 N*M 次。



显然，内存判断次数是不受选择哪个表作为驱动表影响的。而考虑到扫描行数，在 M 和 N 大小确定的情况下，N 小一些，整个算式的结果会更小。



所以结论是，应该让小表当驱动表。



当然，在 N+λ*N*M 这个式子里，λ才是影响扫描行数的关键因素，这个值越小越好。



N 越大，分段数 K 就越大。那么，N 固定时，什么参数会影响 K 的大小呢？

答：λ的大小，即 join_buffer_size的大小

> join_buffer_size 越大，一次可以放入的行越多，分成的段数也就越少，对被驱动表的全表扫描次数就越少



## 什么时候使用join

1. 如果使用 Index Nested-Loop Join算法，也就是可以用上被驱动表上的索引，其实是没问题的
2. 如果使用 Block Nested-Loop Join 算法，扫描行数就会过多。尤其是在大表上的 join 操作，这样可能要扫描被驱动表很多次，会占用大量的系统资源。所以这种 join 尽量不要用



在判断要不要使用 join 语句时，就是看 explain 结果里面，Extra 字段里面有没有出现`“Block Nested Loop”`字样



## 驱动表要选择小表

1. 如果是 Index Nested-Loop Join 算法，应该选择`小表`做驱动表；
2. 如果是 Block Nested-Loop Join 算法：
   * 在 join_buffer_size 足够大时，选择哪个表都是一样的
   * 在 join_buffer_size 不够大时（这种情况更常见），应该选择小表做驱动表



所以，总是应该使用`小表`做驱动表



### 什么叫作小表

**在决定哪个表做驱动表时，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与 join 的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表。**



**例1**

前面的例子是没有加条件的。如果在语句的 where 条件加上` t2.id<=50` 这个限定条件，再来看下这两条语句

```sql
select * from t1 straight_join t2 on (t1.b=t2.b) where t2.id<=50;

select * from t2 straight_join t1 on (t1.b=t2.b) where t2.id<=50;
```

> 注意，为了让两条语句的被驱动表都用不上索引，所以 join 字段都使用了没有索引的字段 b。



但如果是用第二个语句的话，join_buffer 只需要放入 t2 的前 50 行，显然是更好的。所以这里，“t2 的前 50 行”是那个相对小的表，也就是“小表”。



**例2**

```sql
select t1.b,t2.* from  t1  straight_join t2 on (t1.b=t2.b) where t2.id<=100;

select t1.b,t2.* from  t2  straight_join t1 on (t1.b=t2.b) where t2.id<=100;
```

这个例子，表 t1 和 t2 都是只有 100 行参加 join。但是，这两条语句每次查询放入 join_buffer 中的数据是不一样的：

* 表 t1 只查字段 b，因此如果把 t1 放到 join_buffer 中，则 join_buffer 中只需要放入 b 的值；
* 表 t2 需要查所有的字段，因此如果把表 t2 放到 join_buffer 中的话，就需要放入三个字段 id、a 和 b。



所以，应该选择表 t1 作为驱动表。也就是在这个例子，“只需要一列参与 join 的表 t1”是那个相对小的表。



# join的优化

## 前置知识MRR优化

Multi-Range Read优化，简写MRR，这个优化的主要目的尽量使用`顺序读盘`



### 回表

指InnoDB 在普通索引 a 上查到主键 id 的值后，再根据`一个个主键id的值`到主键索引(B+树)上去查一行数据的过程。



执行以下sql语句

```sql
select * from t1 where a>=1 and a<=100;
```



**基本回表流程下所示，图 1**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210712221920.png)



如图，随着 a 的值递增顺序查询的话，id的值是变成随机的，会出现随机访问，性能相对较差。

> 可以调整查询的顺序，加速查询



### MRR优化的设计思路

**因为大多数的数据都是按照`主键递增顺序插入`得到的，如果按照主键的递增顺序查询的话，对磁盘的读比较接近顺序读，能够提升读性能**



此时语句的执行流程变成了：

1. 根据索引a，定位到满足条件的记录，将 id 值放入` read_rnd_buffer `中 
2. 将 read_rnd_buffer 中的 `id进行递增排序`
3. 排序后的 id 数组，依次到主键 id 索引中查记录，并作为结果返回

> 如果步骤 1 中，read_rnd_buffer 放满了，就会先执行完步骤 2 和 3，然后清空 read_rnd_buffer。之后继续找索引 a 的下个记录，并继续循环。



read_rnd_buffer 的大小是由` read_rnd_buffer_size 参数`控制的。



如果要稳定地使用 MRR 优化的话，需要设置`set optimizer_switch="mrr_cost_based=off"`

> （官方文档的说法，是现在的优化器策略，判断消耗的时候，会更倾向于不使用 MRR，把 mrr_cost_based 设置为 off，就是固定使用 MRR 了



**MRR优化后的执行流程，图2 **

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210712222224.png)



**MRR优化后执行流程的 explain 结果，图3 **

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210712222233.png)

可以看到 Extra 字段多了` Using MRR`，表示的是用上了 MRR 优化。



由于在 read_rnd_buffer 中按照 id 做了排序，所以最后得到的结果集也是`按照主键id递增顺序`的，也就是与图 1 结果集中行的顺序相反。



**MRR能够提升性能的核心**

在于这条查询语句在索引 a 上做的是一个`范围查询`（也就是说，这是一个多值查询），可以得到`足够多的主键 id`。这样通过排序以后，再去主键索引查数据，才能体现出“顺序性”的优势。



## Batched Key Access算法(NLJ的优化)

MySQL 在 5.6 版本后开始引入的 `Batched Key Access(BKA) `算法了。这个 BKA 算法，其实就是对 `NLJ算法的优化`



### join_buffer

**NLJ 算法执行的逻辑**

从驱动表 t1，一行行地取出 a 的值，再到被驱动表 t2 去做 join。

> 对于表 t2 来说，每次都是匹配一个值。这时MRR的优势就用不上



**优化思路：**

利用MRR优化，从表t1里一次性地多拿些行出来，一起传给表 t2。



**具体做法**

把表 t1 的数据取出来一部分，先放到一个`临时内存`(` join_buffer`)



**join_buffer 在 BNL 算法里的作用**:`暂存驱动表`的数据

> 但在 NLJ 算法里并没有这样用join_buffer，刚好就可以复用 join_buffer 到 BKA 算法中



**怎么启用BKA**

要使用 BKA 优化算法的话，需要在执行 SQL 语句之前，先设置

```sql
set optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';
```

前两个参数的作用是要启用 MRR，因为BKA 算法的优化要依赖于 MRR。



### BKA算法流程

**BKA 算法的流程，图5**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210723210730.png)

图中，在 join_buffer 中放入的数据是 P1~P100，表示的是`只会取查询需要的字段`。

当然，如果 join buffer 放不下 P1~P100 的所有数据，就会把这 100 行数据`分成多段`执行上图的流程。



## BNL算法的性能问题

**问题**：使用BNL算法时，可能会`对被驱动表做多次扫描`。如果这个被驱动表是一个大的`冷数据表`，除了会导致 IO 压力大以外，还会对系统有什么影响呢？



由于 InnoDB 对 Bufffer Pool 的 LRU 算法做了优化，即：

> 第一次从磁盘读入内存的数据页，会先放在 old 区域。如果 1 秒之后这个数据页不再被访问了，就不会被移动到 LRU 链表头部，这样对 Buffer Pool 的命中率影响就不大。

如果一个使用 BNL 算法的 join 语句，多次扫描一个冷表，而且这个语句执行时间超过 1 秒，就会在再次扫描冷表的时候，把冷表的数据页移到 LRU 链表头部。



这种情况对应的，是冷表的数据量小于整个 Buffer Pool 的 3/8，能够完全放入 old 区域的情况。



如果这个冷表很大，就会出现另外一种情况：`业务正常访问的数据页，没有机会进入 young 区域`。



由于优化机制的存在，一个正常访问的数据页，要进入 young 区域，需要隔 1 秒后再次被访问到。但是，由于我们的 join 语句在循环读磁盘和淘汰内存页，进入 old 区域的数据页，很可能在 1 秒之内就被淘汰了。这样，就会导致这个 MySQL 实例的 Buffer Pool 在这段时间内，young 区域的数据页没有被合理地淘汰。



也就是说，这两种情况都会影响 Buffer Pool 的正常运作。



**大表 join 操作虽然对 IO 有影响，但是在语句执行结束后，对 IO 的影响也就结束了。但是，对 Buffer Pool 的影响就是持续性的，需要依靠后续的查询请求慢慢恢复内存命中率。**

> 为了减少这种影响，可以考虑增大 join_buffer_size 的值，减少对被驱动表的扫描次数。



**BNL 算法对系统的影响主要包括三个方面**

1. 可能会多次扫描被驱动表，占用`磁盘 IO 资源`
2. 判断 join 条件需要执行 M*N 次对比（M、N 分别是两张表的行数），如果是大表就会占用非常多的 `CPU 资源`
3. 可能会导致 Buffer Pool 的热数据被淘汰，`影响内存命中率`



在执行语句之前，需要通过`理论分析`和`查看 explain 结果`的方式，确认是否要使用 BNL 算法。如果确认优化器会使用 BNL 算法，就需要做优化。



优化的常见做法是，`给被驱动表的join字段加上索引，把 BNL 算法转成 BKA 算法`



## BNL转BKA的做法

一些情况下，可以直接在`被驱动表上建索引`，这时就可以直接转成 BKA 算法了。



但是，有时确实会碰到一些`不适合在被驱动表上建索引的情况`。比如下面这个语句

```sql
select * from t1 join t2 on (t1.b=t2.b) where t2.b>=1 and t2.b<=2000;
```

因为我们在表 t2 中插入了 100 万行数据，但是经过 where 条件过滤后，需要参与 join 的只有2000行数据。如果这条语句同时是一个`低频的 SQL 语句`，那么再为这个语句在表 t2 的字段 b 上创建一个索引就很浪费了。



如果使用 `BNL算法来join `的话，这个语句的执行流程是这样的：

1. 把表 t1 的所有字段取出来，存入 join_buffer 中。这个表只有 1000 行，join_buffer_size 默认值是 256k，可以完全存入
2. 扫描表 t2，取出每一行数据跟 join_buffer 中的数据进行对比
   * 如果不满足 t1.b=t2.b，则跳过
   * 如果满足 t1.b=t2.b, 再判断其他条件，也就是是否满足 t2.b 处于[1,2000]的条件，如果是，就作为结果集的一部分返回，否则跳过



对于表 t2 的每一行，判断 join 是否满足时，`都需要遍历 join_buffer 中的所有行`。因此判断等值条件的次数是 1000*100 万 =10 亿次，这个判断的工作量很大。



**图 6 explain结果**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210712223546.png)



**图 7 语句执行时间**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210712223555.png)



explain 结果里 `Extra 字段显示使用了 BNL 算法`。在我的测试环境里，这条语句需要执行 1 分 11 秒。



**问题：在表 t2 的字段 b 上创建索引会`浪费资源`，但是不创建索引的话这个语句的等值条件要判断 10 亿次。那么，有没有其他办法？**

> 可以考虑`使用临时表`。



**使用临时表的大致思路是：**

1. 把表 t2 中满足条件的数据放在临时表 tmp_t 中
2. 为了让 join 使用 BKA 算法，给临时表 tmp_t 的字段 b 加上索引
3. 让表 t1 和 tmp_t 做 join 操作。



此时，对应的 SQL 语句的写法如下

```sql
create temporary table temp_t(id int primary key, a int, b int, index(b))engine=innodb;

insert into temp_t select * from t2 where b>=1 and b<=2000;

select * from t1 join temp_t on (t1.b=temp_t.b);
```

**图 8， 使用临时表的执行效果图**

![](https://sink-blog-pic.oss-cn-shenzhen.aliyuncs.com/img/mysql/20210712223700.png)

整个过程 3 个语句执行时间的总和还不到 1 秒，相比于前面的 1 分 11 秒，性能得到了大幅提升。



**接下来，看一下这个过程的消耗：**

1. 执行 insert 语句构造 temp_t 表并插入数据的过程中，对表 t2 做了全表扫描，这里扫描行数是 100 万。
2. 之后的 join 语句，扫描表 t1，这里的扫描行数是 1000；join 比较过程中，做了 1000 次带索引的查询。相比于优化前的 join 语句需要做 10 亿次条件判断来说，这个优化效果还是很明显的



总体来看，不论是在原表上加索引，还是用有索引的临时表，我们的思路都是让 join 语句能够用上被驱动表上的索引，来触发 BKA 算法，提升查询性能。



## 扩展 -hash join

其实上面计算 10 亿次那个操作，看上去有点儿傻。如果 join_buffer 里面维护的不是一个无序数组，而是一个哈希表的话，那么就不是 10 亿次判断，而是 100 万次 hash 查找。这样的话，整条语句的执行速度就快多了。



这也正是 MySQL 的优化器和执行器一直被诟病的一个原因：不支持哈希 join。

并且，MySQL 官方的 roadmap，也是迟迟没有把这个优化排上议程。



实际上，这个优化思路，可以自己实现在业务端。实现流程大致如下：

1. `select * from t1;`取得表 t1 的全部 1000 行数据，在业务端存入一个 hash 结构，比如 C++ 里的 set、PHP 的数组这样的数据结构。
2. `select * from t2 where b>=1 and b<=2000; `获取表 t2 中满足条件的 2000 行数据。
3. 把这 2000 行数据，一行一行地取到业务端，到 hash 结构的数据表中寻找匹配的数据。满足匹配的条件的这行数据，就作为结果集的一行。



理论上，这个过程会比临时表方案的执行速度还要快一些。


### MySQL



#### 架构

![2u1bu](http://img.miilnvo.xyz/2u1bu.png)

* 服务层

  * 连接器：包括长连接和短连接
  * 查询缓存：v8.0中已经移除了
  * 分析器：分析SQL语法
  * 优化器：决定索引、多表的连接顺序等
  * 执行器：调用存储引擎的接口

  > 服务层还覆盖了所有内置函数、存储过程、触发器、视图等

* 存储引擎层

  指表的类型以及表在计算机上的存储方式
  
  * InnoDB：最常用，支持事务、并发、外键等
  * MyISAM：占用空间小，速度快。适用于Web服务器日志
  * Memory：存储在内存中



#### 日志

| 名称     | 位置         | 存储内容                 | 特点   | 作用     |
| -------- | ------------ | ------------------------ | ------ | -------- |
| redo log | InnoDB引擎层 | 每页的修改，涉及到偏移量 | 循环写 | 崩溃恢复 |
| bin log  | 服务层       | 执行的SQL语句            | 追加写 | 数据同步 |

* redo log

  > InnoDB有Buffer Pool，它是数据库页面的缓存，对InnoDB的任何修改操作（除了查询?）都会首先在缓存上进行，然后这样的页面将被标记为Dirty并被放到专门的Flush List上，后续将由Master Thread或专门的刷脏线程阶段性的将这些页面写入磁盘
  >
  > 好处：避免每次写操作都操作磁盘导致大量的随机IO，阶段性的刷脏可以将多次对页面的修改Merge成一次IO操作，同时异步写入也降低了访问的时延
  >
  > 坏处：如果在还未刷入磁盘时，Server非正常关闭，这些修改操作将会丢失，如果写入操作正在进行，甚至会由于损坏数据文件导致数据库不可用

  WAL（Write-Ahead Logging）机制：先写日志和缓存，再写磁盘

* bin log

  * Statement：记录每一条SQL

  * Row：记录哪条数据被修改

  * Mixedlevel：以上两种混合，一般的语句用第一种，其余用第二种

  ```mysql
  SHOW VARIABLES LIKE 'log_bin';  -- 查看是否开启bin log
  SHOW BINLOG EVENTS IN 'mysql-bin.000001';  -- 查看bin log内容
  ```

两阶段（即两种日志）提交的流程：

> 浅色为存储引擎层，深色为服务层

![klm89](http://img.miilnvo.xyz/klm89.png)

发生崩溃时原库用redo log恢复，备份数据时备份库用bin log恢复，两段提交保证了一致性，但无法保证不丢失数据

[1] 崩溃：重启恢复后检查redo log没有commit，则redo log回滚，由于bin log也没有此记录，所以两个日志保持一致

[2] 崩溃：重启恢复后检查redo log没有commit，但bin log有此记录，则会继续commit，所以两个日志保持一致



#### 事务

> InnoDB支持，MyISAM不支持

```mysql
SHOW VARIABLES LIKE '%AUTOCOMMIT%';  -- 查看是否开启事务自动提交
SET AUTOCOMMIT = 1/0;  -- 开启/关闭事务自动提交
BEGIN / START TRANSACTION;  -- 手动开启新事务
COMMIT;  -- 手动提交
ROLLBACK;  -- 手动回滚
```

事务自动提交会把每一条SQL都当作一个事务来处理，SELECT语句可以不提交事务来执行，MyBatis中对SELECT语句会提交事务

##### 四个特性

* A（Atomicity）：原子性

* C（Consistency）：一致性（转账后两人的总额不变）

* I（Isolation）：隔离性

  ```sql
  SELECT @@transaction_isolation;  -- 查询隔离级别，默认为可重复读
  ```
  
  * 读取未提交内容（Read Uncommitted）：可能引起脏读、不可重复读、幻读
  
    > B事务没提交，A事务就可以查到B事务修改过的数据。但是若B事务进行回滚后，则A事务之前读取到结果就是脏读数据
  
  * 读取提交内容（Read Committed）：可能引起不可重复读、幻读
  
    > B事务提交后，A事务才可以查到B事务的修改过的数据。但是若A事务在B事务开始前读取一次，B事务进行了修改后，A事务再次读取时的结果与上次不一致，即不可重复读
  
  * 可重复读（Repeatable Read）：可能引起幻读
  
    > 与B事务无关，A事务在自己事务的整个阶段查到的内容都不变。如果需要修改，则使用的数据是B事务提交后的结果。A事务新增或删除的行可能已经被B事务新增或删除，进而导致错误，即幻读。

    > 使用了MVCC。可重复读的核心就是一致性读，而事务更新数据的时候，只能用当前读。如果当前的记录的行锁被其他事务占用的话，就需要进入锁等待
  
  * 可串行化（Serializable）
  
    > 写时加写锁，读时加读锁

* D（Durability）：持久性

##### MVCC

Multi-Version Concurrency Control（多版本并发控制）

每条记录在更新时都会同时记录一条回滚操作（快照），查询这条记录时，不同时刻启动的事务会有不同的快照版本，即同一行数据可以存在多个不同版本

作用：解决并发问题，可以代替低效的锁机制

实现方式：在每行记录后面保存两个隐藏的列，分别保存了这个行的创建版本号和删除版本号，每开始一个新的事务，此版本号就会自动递增

优点：读操作永远不会被阻塞

缺点：存储多个版本的冗余数据

具体操作：

​	`INSERT`：插入一行，创建版本号为当前版本号

​	`SELECT`：查找满足条件的行（创建版本号 / 未定义 <= 当前版本号 < 删除版本号 / 未定义）

​	`UPDATE`：旧数据的删除版本号为当前版本号，新数据的创建版本号为当前版本号

​	`DELETE`：删除一行，删除版本号为当前版本号

* 快照读：不加锁

  ```sql
  select * from table where X
  ```

* 当前读：需要加锁

  ```sql
  insert into table values ...  -- 自动排他锁
  update table set X where X  -- 自动排他锁
  delete from table where X  -- 自动排他锁
  select * from table where X lock in share mode  -- 手动共享锁
  select * from table where X for update  -- 手动排他锁（可以解决幻读）
  ```


【参考】

<http://www.zsythink.net/archives/1233>



#### 锁

* 全局锁

  ```mysql
  FLUSH TABLES WITH READ LOCK;  -- 加锁
  UNLOCK TABLES;  -- 解锁
  ```

  使用场景：全库备份

* 表行锁

  * 表锁

    ```mysql
    LOCK TABLES t READ / WRITE;  -- 加锁
    UNLOCK TABLES;  -- 解锁
    ```

  * 元数据锁（v5.5）

    当对一个表做增删改查操作时，加MDL读锁；当要对表做结构变更操作时，加MDL写锁

    不需要显示使用，在每次访问一个表时会自动加上

    在做表结构变更时，要避免因获取不到MDL写锁而导致之后的查询全部被阻塞

* 行锁

  > InnoDB支持，MyISAM不支持

  * 共享锁 / 读锁

    ```sql
    + LOCK IN SHARE MODE
    ```
  
  * 排他锁 / 写锁
  
    ```sql
    + FOR UPDATE
    ```
  
  通过给索引上的索引项加锁来实现的，如果匹配的字段没有索引，那么使用的是表锁
  
  如果同时存在多种锁（MDL锁和行锁），必须全部不互斥才能并行



#### 索引

> 不同的存储引擎对索引的实现方式是不同的

##### 存储结构

N阶B+树：

![iqoob](http://img.miilnvo.xyz/iqoob.png)

一个节点的长度（默认16kb）为页（4kb）的整数倍，通过磁盘预读（局部性原理），一次磁盘I/O操作可以读取多个页

非叶节点通常会在初始化时加载到内存中

节点内的搜索使用二分法（二叉搜索树）

每搜索一层需要一次磁盘I/O，千万量级甚至上亿量级的索引树的高度通常都在2~5之间

| B+树的特点                                          | 带来的优点   |
| --------------------------------------------------- | ------------ |
| 每个非叶节点可以存储多个Key，叶子节点存储Key和Value | 降低树的高度 |
| 叶子节点依次头尾相连                                | 方便范围扫描 |

理论基础：一般来说，索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储的磁盘上。这样的话，索引查找过程中就要产生磁盘I/O消耗，相对于内存存取，I/O存取的消耗要高几个数量级，所以评价一个数据结构作为索引的优劣最重要的指标就是在查找过程中磁盘I/O操作次数的渐进复杂度。换句话说，索引的结构组织要尽量减少查找过程中磁盘I/O的存取次数

【参考】

http://blog.codinglabs.org/articles/theory-of-mysql-index.html

##### 索引类别

> InnoDB是聚簇索引，MyISAM是非聚簇索引

![f92um](http://img.miilnvo.xyz/f92um.png)



* 聚簇索引 / 主键索引：索引和数据为同一份文件，每个叶子节点的值都包含了主键及其它所有列

  > 每张表都会有主键， 如果没有则寻找非空的唯一索引作为主键，仍没有则创建六个字节大小的隐藏主键

  > 主键应使用自增ID：因为数据的物理存放顺序与索引顺序一致，所以在中间插入数据会导致大量随机I/O、页分裂等情况（插入后可以用OPTIMIZE TABLE来优化碎片）

* 非聚簇索引 / 二级索引：索引和数据为不是同一份文件，每个叶子节点的值只包含了主键

  > 主键长度应尽量小：因为所有的非聚簇索引都会保存主键，所以主键过大会导致索引文件也过大

  > 从二级索引的主键找到主键索引是随机I/O，可以使用[MRR](#MRR)优化

  > 如何选择普通索引和唯一索引：查询时唯一索引略微快于普通索引，更新时唯一索引要把数据页读入内存后判断是否有冲突，比普通索引慢。推荐使用普通索引

##### 索引优化

* 最左前缀原则（即第一颗星）

  索引 [ a，b，c ] 可以支持 [ a ] [ a，b ] [ a，b，c ] 三种查找

  如果是`WHERE b = xxx AND a = xxx`，那么优化器会对条件的顺序进行优化

  `LIKE '张%'`可以使用索引

* 三星索引

  1. 第一颗星：where后面的谓词和索引列顺序匹配

  2. 第两颗星：order by中的排序和索引列顺序匹配

  3. 第三颗星：索引列覆盖查询语句中需要的所有列，不需要再回表查询

     > 即执行计划的"Using index"

  如果同时满足三颗星，那么一次查询只需要进行一次磁盘随机读写以及一次窄索引片的扫描

  > 在查询数据条数约占总条数五分之一以下时能够使用到索引，但超过五分之一时，则使用全表扫描了

* 延迟关联

  随着偏移量的增加，需要花费大量的时间去扫描肯定要丢弃的数据

  先通过覆盖索引获取主键ID（这一步不需要访问主键索引，也就不需要访问其它列），再根据主键ID获取需要的数据

  优化前：

  ```mysql
  SELECT <cols> FROM <t> WHERE <cond1> ORDER BY <cond2> LIMIT 100000,10;
  ```

  优化后：

  ```sql
  SELECT <cols> FROM <t> AS a JOIN (
  	SELECT id FROM <t> WHERE <cond1> ORDER BY <cond2> LIMIT 100000,10
  ) AS b ON a.id = b.id;
  ```

* <span id="ICP">ICP（Index Condition Pushdown）</span>（v5.6）

  ```sql
  SELECT * FROM user WHERE name LIKE '张%' AND age=10 AND ismale=1;  -- index(name,age)
  ```

  在老版本中会在扫描完name后就回表找到数据行，再筛选age和ismale字段

  在新版本中会在索引中筛选age字段 ，之后再回表

  ```mysql
  set optimizer_switch='index_condition_pushdown=on';  -- 开启ICP
  ```
  
  > 对主键索引无效果
  
  > 即执行计划的"Using index condition"


* <span id="MRR">MRR（Multi-Range Read）</span>（v5.6）

  对根据二级索引获取的结果集（多行），缓存后按照主键进行排序，最后再回表查询。目的是把访问主键索引的随机I/O转换成顺序I/O
  
  ```mysql
  SET optimizer_switch='mrr=on';  -- 开启MRR
  SET optimizer_switch='mrr_cost_based=on';  -- 开启是否值得进行MRR的判断
  SHOW VARIABLES LIKE 'read_rnd_buffer_size';  -- 查看缓存区大小
  ```
  
  > 即执行计划的"Using MRR"
  
  【参考】
  
  https://zhuanlan.zhihu.com/p/110154066




#### 数据类型

int(11) 数字表示的是显示长度

varchar(X) 最大能存储65535字符，最大长度与编码方式有关

> varchar(10)和varchar(100)占用的磁盘空间是相同的，但消耗的内存空间不同（例如排序时）



#### SQL

执行顺序：`FROM` => `WHERE` => `GROUP BY` => `HAVING` => `SELECT` =>  `ORDER BY`

![i81ou](http://img.miilnvo.xyz/i81ou.png)

##### JOIN

> MySQL仅支持一种方式：嵌套循环 NLJ（Nested-Loop Join）

```mysql
-- 驱动表t1的行数是N=100，被驱动表t2的行数是M=1000
SELECT * FROM t1 straight_join t2 ON (t1.a = t2.a);
```

* SNLJ（Simple Nested-Loop Join）

  > 扫描行数：N + M，判断行数：N * M

* BNL（Block Nested-Loop Join）（v5.5）

  > 当Join Buffer大于驱动表容量时，扫描行数：N + M，判断行数：N * M	

  > 当Join Buffer小于驱动表容量时，扫描行数：N + KM，判断行数：N * M（K表示分块次数，K = λ * N）

  在SNLJ的基础上使用了Join Buffer，把t1表的多行缓存起来

  因为是在内存中比较，所以比SNLJ快一点
  
  当join_buffer_size值越大，一次可以放入的行越多，需要扫描被驱动表的次数也就越少
```mysql
  SET optimizer_switch='block_nested_loop=on';  -- 开启BNL
SHOW VARIABLES LIKE 'join_buffer_size';  -- 查看Join Buffer的缓存大小
```

  > 大表之间的Join操作虽然对IO有影响，但是在语句执行结束后，对IO的影响也就结束了。但是，对Buffer Pool的影响就是持续性的，需要依靠后续的查询请求慢慢恢复内存命中率

* INLJ（Index Nested-Loop Join）

  > 扫描行数：N，判断行数：N2log~2~M（2表示可能需要回表）

  此时的t2表上的字段a有索引

  使用`JOIN`：一共要执行1条SQL语句

  1. 从表t1中读入一行数据 R（all级别，扫描100行）
  2. 从数据行R中，取出a字段到表t2的索引里去查找（ref级别，扫描100行）
  3. 取出表t2中满足条件的行，跟R组成一行，作为结果集的一部分
  4. 重复执行步骤1~3，直到表t1的100行都循环结束

  不使用`JOIN`：一共执行101条SQL语句

  1. 执行`SELECT * FROM t1`，查出表t1的所有数据（all级别，扫描100行）
  2. 在客户端（代码层面）循环遍历这100行数据
     1. 从每一行R取出字段a的值$R.a
     2. 执行`SELECT * FROM t2 WHERE a = $R.a`（ref级别，扫描100行）
     3. 把返回的结果和R构成结果集的一行

  结论：建议使用`JOIN`，且需要用小表驱动大表

  * BKA（Batched Key Access）（v5.6）

    与BNL同理，把t1表的多行缓存起来
    
    基于[MRR](#MRR)，利用了BNL的Join Buffer缓存区

    ```mysql
    SET optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';  -- 开启BKA
    ```
  
  
  【参考】
  
  https://blog.csdn.net/huxiaodong1994/article/details/91668304

**三表Join的流程**

1. 调用InnoDB接口，从t1中取一行数据，数据返回到Server层

2. 调用InnoDB接口，从t2中取满足条件的数据，数据返回到Server层

3. 调用InnoDB接口，从t3中取满足条件的数据，数据返回到Server层

4. 上面三步之后，驱动表 t1的一条数据就处理完了，接下来重复上述过程

> 如果使用BKA优化，就不是一行行提取数据，而是一个范围内提取数据

**使用临时表的优化案例**

t1表有1000条数据，t2表有1,000,000条数据，经过b字段筛选后只有2000条数据，所以没有必要在字段b上加索引

优化前：

```mysql
SELECT * FROM t1 JOIN t2 ON (t1.b = t2.b) WHERE t2.b >= 1 AND t2.b <= 2000;
```

一共需要扫描1000（all）*1000000（all）= 10亿次，耗时40多秒

优化后：

```mysql
-- 创建临时表temp_t，并设置索引
CREATE TEMPORARY TABLE temp_t(id INT PRIMARY KEY, a INT, b INT, INDEX(b))ENGINE=INNODB;
-- 把大数据量的小结果插入到临时表中
INSERT INTO temp_t SELECT * FROM t2 WHERE b>=1 AND b<=2000;
-- 利用NLJ(BKA)查询
SELECT * FROM t1 JOIN temp_t ON (t1.b=temp_t.b);
```

一共需要扫描1000000（all）+1000（all）+1000\*2\*log~2~2000（ref）=1023000次 （相差977倍），总耗时0.39秒（包括插入2000条的时间）

【参考】

<https://www.cnblogs.com/chenpingzhao/p/6720531.html>



##### <span id="WHERE">WHERE</span>

使用`WHERE`的三种情况：

> 即执行计划的"Using where"

1. 在存储引擎层的索引列中过滤不匹配的索引（[ICP](#ICP)）

2. 在服务层使用覆盖索引返回记录的结果中过滤

   ```mysql
   SELECT t.a FROM t WHERE t.a='X';  -- "Using index"
   SELECT t.a FROM t WHERE t.a='X' AND t.b='Y';  -- index(a)，"Using where"
   ```

3. 在服务层中过滤不匹配的数据行（最常见）

   ```mysql
   SELECT * FROM t WHERE t.b='Y';
   ```



##### HAVING

用于筛选满足条件的组

可直接代替`WHERE`的情况：当`SELECT`的也存在相同的列



##### COUNT函数

实现方式：一条一条的全表扫描

> 优化器会找到最小的索引树来遍历，尽量减少扫描的数据量

如果像MyISAM事先记录总行数，那么B事务更新后A事务要读取最新值，这样就违背了可重复读

而`SHOW TABLE STATUS`虽然返回很快，但是不准确（最大误差有50%）

最佳方案是用计数表记录总行数，并在一个事务中读取总行数和读取所有记录，可以保证两者数据一致

`COUNT(*)`：已专门优化

`COUNT(1)`：遍历整表但不取值，判断buweinull时累加

`COUNT(主键ID)`：遍历整表取ID，判断不为null时累加

`COUNT(字段)`：遍历整表取数据，先判断字段是否允许为null，再判断字段值，不为null时累加

效率：`COUNT(字段)`  <  `COUNT(主键id) ` <  `COUNT(1) ` ≈ ` COUNT(*)`



##### IN函数

会先对`IN()`列表里的数据进行排序，然后通过二分法查找的方式来确定是否满足条件

会使用到索引（v5.5）



##### USING函数

```mysql
-- 等价
SELECT * FROM t1 JOIN t2 ON t1.id = t2.id
SELECT * FROM t1 JOIN t2 USING(id)
```



##### 练习题

* TopN

  s1表里满足条件的成绩<u>小于其他成绩的个数</u>只能是0（最高）或者1（第二高）

  ```mysql
  -- 查询各科成绩前两名的记录
  SELECT s_id, c_id, s_score FROM Score s1 WHERE (
  	SELECT COUNT(1) FROM Score s2 WHERE s1.c_id = s2.c_id AND s1.s_score < s2.s_score
  ) < 2
  ```

  【参考】

  https://blog.csdn.net/weixin_40844116/article/details/93141543

* 行列转换

  学生、课程、成绩 => 学生、课程1的成绩、课程2的成绩、课程3的成绩

  ```mysql
  SELECT
  	s_id,
  	SUM(if(c_id=01, s_score, 0)) AS '01',  -- SUM换成MAX也可以
  	SUM(if(c_id=02, s_score, 0)) AS '02',
  	SUM(if(c_id=03, s_score, 0)) AS '03',
  	SUM(s_score) AS '总分'
  FROM Score GROUP BY s_id;
  ```

* 薪水第二多

  因为存在相同工资的人，所以工资要先分组再排序



#### 关联查询

* 交叉连接（笛卡尔积）

  ```mysql
  SELECT * FROM t1, t2
  SELECT * FROM t1 (INNER/CROSS) JOIN t2
  ```

* 内连接

  ```mysql
  SELECT * FROM t1, t2 WHERE t1.id = t2.id
  SELECT * FROM t1 (INNER/CROSS) JOIN t2 ON t1.id = t2.id
  ```

  > 在MySQL中，cross join与inner join相同

* 左外连接

  ```mysql
  SELECT * FROM t1 LEFT (OUTER) JOIN t2 ON t1.id = t2.id
  ```

* 右外连接

  ```mysql
  SELECT * FROM t1 RIGHT (OUTER) JOIN t2 ON t1.id = t2.id
  ```

* 全外连接

  ```mysql
  左连接 union 右连接
  ```

外连接时如果没有对应的数据，则在相应位置上的值为null

`HAVING`子句将过滤条件应用于每组的分行，而`WHERE`子句将过滤条件应用于每个单独的行

如果省略`GROUP BY`子句，则`HAVING`子句的行为与`WHERE`子句类似



#### Explain 

| 列名  | 含义                                                         |
| :---- | :----------------------------------------------------------- |
| type  | system：表中仅此一行数据<br>const：通过主键索引或唯一索引，扫描一次就找到唯一数据<br>eq_ref：多表联查时，关联条件为主键索引或唯一索引，返回唯一数据<br>ref：即普通索引，可能会返回多条数据<br/>range：遍历索引的某个区间，通常是存在>、<、IN、BETWEEN<br/>index：遍历全索引<br/>all：遍历全表 |
| rows  | 表示预计扫描行数，由采样统计得到，可能会影响优化器选择错误的索引 |
| extra | Using filesort：没有使用索引进行排序<br/>Using temporary：使用了内存或磁盘的内部临时表<br/>Using index：使用了覆盖索引（第三颗星）<br/>Using where：[WHERE](#WHERE)<br/>Using join buffer：使用了缓存区，通常是因为没有有效的索引（BNL） |



#### 性能调优

* [ICP](#ICP)（Index Condition Pushdown）

* [MRR](#MRR)（Multi-Range Read）

* [BKA](#BKA)（Batched Key Access）

* Buffer Pool

  访问表和索引数据时会在其进行高速缓存，使用LRU算法

  ```mysql
  innodb_buffer_pool_size  -- 配置大小，推荐物理内存的50%~80%
  ```

* Change Buffer

  更新数据页时先把操作缓存在Change Buffer中，避免了从磁盘读取数据后再修改的读I/O消耗。当下次查询这个数据页或经过某个间隔后再执行更新。只有非唯一的普通索引能使用。适用于写多读少的场景

  ```mysql
  innodb_change_buffer_max_size  -- 配置写缓冲的大小，占整个缓冲池的比例，默认25%
  innodb_change_buffering  -- 配置哪些写操作需要启用写缓冲
  ```

  > redo log主要节省的是随机写磁盘的IO消耗（转成顺序写），而Change Buffer主要节省的则是随机读磁盘的IO消耗

  【参考】

  https://www.cnblogs.com/zhumengke/articles/12170806.html

* 其它配置

  ```mysql
  innodb_flush_log_at_trx_commit=1  -- 控制redo log刷新到磁盘的策略，0/2性能更好但稳定性更差
  innodb_max_dirty_pages_pct=30  -- 脏页占Buffer Pool一定比例时触发刷脏页到磁盘，推荐值25%~50%
  innodb_io_capacity=200  -- 刷新脏页的速度，推荐为设备的IOPS最大值的50%~75%
  innodb_io_capacity_max  -- 刷新脏页的最大速度，推荐为设备的IOPS最大值
  long_qurey_time=1  -- 慢查询的阈值，单位秒
  innodb_log_file_size  -- redo log的空间大小，推荐的大小为应该足够容纳服务器一个小时的活动内容
  ```

  【参考】

  https://www.jianshu.com/p/fc74e946729b

  https://cloud.tencent.com/developer/article/1596411

  https://www.cnblogs.com/glon/p/6497377.html

  https://www.centos.bz/2016/11/mysql-performance-tuning-15-config-item/



#### 其它

在Navicat中，查询页面是一个session，表页面也是一个session

`show processlist`：查看所有连接状态

`alter table xxx engine innodb`和`optimize table`：清理索引碎片

添加外键：

```mysql
ALTER TABLE 副表名 ADD CONSTRAINT 外键名 FOREIGN KEY(副表的外键) REFERENCES 主表名(主表的主键)
```
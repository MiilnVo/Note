### Redis

全称：**Re**mote **Di**ctionary **S**erver（远程字典服务）

定位：NoSQL（Not Only SQL）数据库

存储形式：键值对（Key - Value）

特点：单线程执行，采用队列将并发访问变成串行访问，所有的命令都是原子性

作用：缓存、分布式锁



####  线程模型

文件事件处理器（**单进程单线程**）：

![s4wbv](http://img.miilnvo.xyz/s4wbv.png)

* I/O多路复用程序：监听多个Socket，将产生事件的Socket放入一条队列中依次处理

  > 多路指多个网络连接，复用指共用一条线程

  > 会自动选择系统中性能最高的I/O多路复用函数（select、epoll、evport、kqueue）

* 文件事件分配器：根据Socket产生的事件类型，调用相应的处理器

* 文件事件处理器：包括连接应答处理器、命令请求处理器和命令回复处理器


> Q：为什么要设计成单线程？
>
> A：多线程主要是为了解决I/O设备的读写速度往往比CPU的处理速度慢而造成的单线程运行阻塞的问题，多线程能提高因等待某个资源阻塞时其它资源的利用率（是利用率不是效率），只有存在磁盘I/O等耗时操作的情况下才适合使用多线程。Redis都是基于内存的操作，不存在磁盘I/O操作的耗时，如果改成多线程，那么反而会因为线程间切换、同步锁等问题降低执行速度

【参考】

<https://blog.csdn.net/qq_30228707/article/details/90263203>

<https://cloud.tencent.com/developer/article/1403767>

<https://www.jianshu.com/p/ccae497c0ebb>



#### 底层数据结构

* SDS（Simple Dynamic String，简单动态字符串）

   ```c
   struct sdshdr {
   		int len;  // buf数组中已使用的字节数量
   		int free;  // buf数组中未使用的字节数量
   		char buf[];  // 用于保存字符串
   }
   ```
   
   ![1qphy](http://img.miilnvo.xyz/1qphy.png)

   > 应用：Key、AOF缓冲区

* Linked List（双端无环链表）

   ```c
   typedef struct listNode {
   		struct listNode *prev;
   		struct listNode *next;
   		void *value;
   } listNode;
   ```

   ```c
   typedef struct list {
   		listNode *head;
   		listNode *tail;
   		unsigned long len;
   		void *(*dup)(void *ptr);  // 节点值复制函数
   		void (*free)(void *ptr);  // 节点值释放函数
   		int (*match)(void *ptr, void *key);  // 节点值对比函数
   } list;
   ```

   > 应用：List数据类型、发布订阅、慢查询、监视器等

* Dict（字典）

   ```c
   typedef struct dict {
   		dictType *type;  // 特定类型函数
   		void *privdata;  // 私有数据
   		dictht ht[2];  // 哈希表，[0]存储数据，[1]扩容时使用
   		int rehashidx;  // 当不在进行rehash时值为-1
   }
   ```
   
   ```c
   typedef struct dictht {
   		dictEntry **table;  // 哈希表数组
   		unsigned long size;  // 数组大小
   		unsigned long sizemask;  // 总等于size-1
   		unsigned long used;  // 已使用的数量
   } dictht;
   ```
   
   ```c
   typedef struct dictEntry {
   		void *key;
   		union {
   				void *val;
   				uint64_tu64;
   				int64_ts64;
   		} v;
   		struct dictEntry *next;  // 指向下个哈希表节点，形成链表
   }
   ```
   
   <img src="http://img.miilnvo.xyz/h6712.png" alt="h6712" style="zoom: 50%;" />

   哈希算法为MurmurHash3

   使用单向链表解决键冲突

   > 应用：Hash数据类型、Redis的数据库

* Skip List（跳跃表）

   ```c
   typedef struct zkiplistNode {
   		// 层级
   		struct zkiplistLevel {
   				// 前进指针
   				struct zkiplistNode *forward;
   				// 跨度
   				unsigned int span;
   		} level[];
   		// 后退指针
   		struct zskiplistNode *backward;
   		// 分值，按照此值从小到大排列
   		double score;
   		// 对象
   		robj *obj;
   } zskiplistNode;
   ```

   ```c
   typedef struct zkiplist {
   		// 头节点和尾节点
   		structz skiplistNode *header, *tail;
   		// 节点数量
   		unsigned long length;
   		// 节点层数
   		int level;
   }
   ```

   <img src="http://img.miilnvo.xyz/k4iob.png" alt="k4iob" style="zoom: 60%;" />

   > 应用：Zset数据类型

   【参考】https://www.bilibili.com/video/BV1tK4y1X7de

* Int Set（整数集合）

   ```c
   typedef struct intset {
   		// 编码方式：int16_t，int32_t，int64_t
   		uint32_t encoding;
   		// 元素数量
   		uint32_t length;
   		// 元素数组，从小到大排列，保证不出现重复元素
   		int8_t contents[];
   } intset;
   ```

   升级机制：当新元素的类型比现有的长时，要重新分配空间，所以向整数集合添加新元素的时间复杂度为O(N)

   > 升级的优点：提升灵活性、节约内存

   > 应用：Set数据类型

* Zip List（压缩列表）

   <img src="http://img.miilnvo.xyz/n219y.png" alt="n219y" style="zoom:60%;" />

   ![q8xd3](http://img.miilnvo.xyz/q8xd3.png)
   
   entry结构：
   
   <img src="http://img.miilnvo.xyz/jydor.png" alt="jydor" style="zoom:60%;" />
   
   * previous_entry_length：前一个节点的长度，1字节或5字节，会受新节点的content影响而产生连锁更新问题
   
   * encoding：字节数组编码和纯整数编码，1字节、2字节或5字节
   
   * content：具体的节点值
   
   不像数组那样以最大长度的字符串大小作为所有元素的大小，避免浪费空间
   
   使用紧凑的连续内存块顺序存储数据（最小2字节，最大10字节），未使用Linked List的listNode和Dict的dictEntry结构（都是24字节）
   
   访问元素时采用的是从后往前遍历，因此热点Key是放在最后的，以此可以提升性能
   
   > 应用：Set数据类型、Hash数据类型



#### 五大数据类型

每次在Redis的数据中创建一个键值对时，至少会创建两个`redisObject`对象

Key始终是一个字符串类型，Value则有5种

```c
typedef struct redisObject {
		// 类型，共5种
		unsigned type:4;
  	// 编码，共8种
  	unsigned encoding:4;
  	// 指向底层数据结构的指针
  	void *ptr;
  	// 引用计数：当被引用时加1，不再引用是减1，为0时被释放
  	int refcount;
  	// 最后一次访问的时间差：当前时间减去创建时间
  	// 与内存淘汰策略中的volatile-lru和allkeys-lru有关
  	unsigned lru:22;
}
```

5种类型：`REDIS_STRING`、`REDIS_LIST`、`REDIS_HASH`、`REDIS_SET`、`REDIS_ZSET`

8种编码：`REDIS_ENCODING_INT`、`REDIS_ENCODING_EMBSTR`、`REDIS_ENCODING_RAW`、

`REDIS_ENCODING_HT`、`REDIS_ENCODING_LINKEDLIST`、`REDIS_ENCODING_ZIPLIST`、

`REDIS_ENCODING_INTSET`、`REDIS_ENCODING_SKIPLIST`

> 8种编码 = 6种数据结构 + INT + RAW

每种类型至少对应两种编码，可以根据不同的使用场景来为一个对象设置不同的编码，极大地提升了灵活性和效率

一个命令可以处理多种不同类型的键，一个命令也可以处理多种不同的编码

> 对象共享：初始化时创建了0-9999的所有整数值

* String（字符串）

  ![x3a4n](http://img.miilnvo.xyz/x3a4n.jpg)

  ```redis
  SET <key> <value>：增加值
  GET <key>：返回值
  GETSET <key> <value>：增加值，并返回旧值
  INCR / DECR <key>：值增/减1
  INCRBY / DECRBY <key> <inc>：值增/减inc
  APPEND <key> <value>：增加到原来值的末尾
  ```

  如果是整数值则用INT，如果是字符串且长度小于32字节则用EMBSTR，大于32字节则用RAW。EMBSTR比RAW的优势在于只调用一次内存分配函数

  > 应用场景：原子计数

* List（列表）

  ![6bnfx](http://img.miilnvo.xyz/6bnfx.png)

  ```redis
  LPUSH / RPUSH <key> [value1]：增加多个值到列表头/列表尾
  LPOP / RPOP <key>：删除并返回第一个值/最后一个值
  LSET <key> <index> <value>：通过索引设定值
  LINDEX <key> <index>：通过索引返回值
  ```

  如果保存的所有元素长度都小于64字节且数量小于512个则用LINKEDLIST，否则用ZIPLIST

  > 应用场景：关注列表、评论列表

* Hash（哈希）

  ![ntwha](http://img.miilnvo.xyz/ntwha.png)

  ```redis
  HMSET <key> [field1 value1]：把多个field-value设定到key中
  HKEYS <key>：返回key的所有字段
  HGET <key> <field>：返回key的字段的值
  HDEL <key> [field1]：删除多个key的字段
  ```

  如果保存的所有元素长度都小于64字节且数量小于512个则用ZIPLIST，否则用HT（Dict）

  > 应用场景：结构化数据（用户信息）

* Set（无序集合）

  与List类似，但是元素唯一

  ```redis
  SADD <key> [value1]：增加多个值
  SREM <key> [value1]：删除多个值
  SMEMBERS <key>：返回所有值
  ```

  如果保存的所有元素都是整数值且数量小于512个则用INTSET，否则用HT（Dict）

  > 应用场景：微博的所有关注人、QQ个人标签

* Zset（有序集合）

  与List类似，但元素唯一且具有权重值

  ![0q9ni](http://img.miilnvo.xyz/0q9ni.png)

  ```redis
  ZADD <key> [score1 value1]：增加多个值
  ZREM <key> [value1]：删除多个值
  ZRANK <key> <value>：返回值的分数值
  ```
  
  如果保存的所有元素长度都小于64字节且数量小于128个则用ZIPLIST，否则用SKIPLIST。后者同时使用Dict和Skip List，优点是既能以O(1)查找元素，又能以O(NlogN)进行排序
  
  ```c
  // REDIS_ENCODING_SKIPLIST编码
  typedef struct zset {
    zskiplist *zsl;
    dict *dict;
  } zset;
  ```
  
  > 应用场景：排行榜（按照点赞排序）、过期项目（把score设置成过期时间）

【参考】

<https://cloud.tencent.com/developer/article/1349738>



#### 数据库

```c
struct redisServer {
  	redisDB *db;  // 数据库数组
  	int dbnum;  // 数据库个数
}
```

```c
typedef struct redisDB {
  	dict *dict;  // 保存着所有键值对
  	dict *expires;	// 保存着所有键的过期时间
}
```

![2lgml](http://img.miilnvo.xyz/2lgml.png)



---



#### 持久化

> Redis的数据都存储在内存中，那么必然有某种机制用来保存这些数据

* RDB（Redis DataBase）

  将数据库快照保存在名称为的dump.rdb的二进制文件中

  * 手动执行`save`命令：会阻塞主进程，不建议使用
  * 手动执行`bgsave`命令：使用操作系统的`fock()`生成一个子进程
  * 指定时间内达到某种条件时自动执行`bgsave`命令（默认开启）

* AOF（Append Olny File）

  记录每次对服务器写的命令，追加到名称为appendonly.aof文件的末尾，文件内容遵循RESP协议

  * `always`策略：每条新命令执行一次同步
  * `everysec`策略：每秒执行一次同步（推荐使用）
  * `no`策略：由操作系统来决定何时同步

  AOF重写：把冗余的命令合并成一条命令（直接把当前内存的数据生成对应命令，不需要读取旧AOF文件）。在子进程重写过程中会继续将新命令追加到重写缓存区（3-2）和旧的AOF文件中（3-1），后者的操作是为了避免重写过程中发生停机导致的数据丢失。此操作会自动触发或手动执行`bgrewriteaof`命令

  ![78rgm](http://img.miilnvo.xyz/78rgm.jpg)

|    区别    |            RDB             |             AOF             |
| :--------: | :------------------------: | :-------------------------: |
| 启动优先级 |             后             |             先              |
|  文件体积  |             小             |             大              |
|  恢复速度  |             快             |             慢              |
| 数据安全性 | 最多丢失在两次间隔内的数据 | 默认策略下最多丢失1秒的数据 |

<img src="http://img.miilnvo.xyz/mogua.png" alt="image-20200827145436082" style="zoom:50%;" />

【参考】

<https://segmentfault.com/a/1190000015983518>

<https://segmentfault.com/a/1190000016021217>



#### 过期策略

`EXPIRE <key> <seconds>`：设置Key的过期时间（单位秒）

`PEXPIRE <key> <millisecond>`：设置Key的过期时间（单位毫秒）

`EXPIREAT <KEY> <timestamp>`：设置Key的过期时间（单位秒）

`PEXPIREAT <KEY> <timestamp>`：设置Key的过期时间（单位毫秒）

* 惰性删除

  Key刚过期的时候不删除，每次获取Key的时候通过expireIfNeeded函数去检查是否过期，对已过期的Key执行删除

* 定期删除

  每隔一段时间（默认每隔100ms）通过serverCron函数里的activeExpireCycle函数分批遍历各个数据库，从数据库的expires字典里扫描一定数量的Key，对已过期的Key执行删除
  

> 已过期的Key不会持久化到RDB文件，以主服务器模式启动时也不会导入回Redis，但以从模式启动时会导入，不过在进行主从同步时从服务器的数据库会被清空，所以也没有影响

> 已过期的Key会向AOF文件追加一条DEL命令，也不会保存到重写后的AOF文件中

> v3.2版本之前读从库时不会通过惰性删除的方式删除已过期的Key，依赖主节点同步过来的删除命令

【参考】

https://www.cnblogs.com/java-zhao/p/5205771.html

https://blog.csdn.net/itomge/article/details/121298609



#### 内存淘汰策略

`maxmemory`：最大可用内存

`maxmemory-policy`：超过最大可用内存后的淘汰策略

`maxmeory-samples`：策略中的随机取样个数

* volatile-lru：从设置了过期时间的Key中随机取样挑选出其中最近最少使用的Key进行删除
* volatile-ttl：从设置了过期时间的Key中随机挑选出其中快要过期的进行删除
* volatile-random：从设置了过期时间的Key中随机挑选进行删除
* allkeys-lru：从所有Key中随机取样挑选出最近最少使用的进行删除
* allkeys-random：从所有Key中随机挑选进行删除
* no-enviction：禁止删除，返回错误

> 通过此策略删除的Key，会影响到持久化的结果，即持久化文件中数据的大小不会超过可用内存大小

【参考】

<https://blog.csdn.net/caishenfans/article/details/44902651>

<https://juejin.im/post/5d107ad851882576df7fba9e>



#### 发布与订阅

PUBLISH（发送）、SUBSCRIBE（订阅）、PSUBSCRIBE（订阅模式）

订阅模式发送时要遍历列表



#### 数据通知

键空间通知：对某个键执行了什么命令

键事件通知：某个命令被什么键执行了



#### 事务

保证命令的批量顺序执行，执行过程中其他客户端提交的命令请求不会插入到事务执行命令的序列中

批量操作在发送`EXEC`命令前被放入队列缓存，收到`EXEC`命令后进入事务执行，事务中任意命令执行失败，其余的命令依然被执行（不会回滚！）

`MULTI`：开始一个事务

`EXEC`：执行事务块内的命令

`DISCARD`：放弃一个事务

`WATCH [key]`：监视Key，如果在事务执行前Key有改动，则事务将不会执行



---



#### RESP协议

Redis客户端和服务端之间的通信协议，基于TCP协议

全称：Redis Serialization Protocol

规则：

​	单行回复：回复的第一个字节是"+"

​	错误信息：回复的第一个字节是"-"

​	整形数字：回复的第一个字节是":"

​	多行字符串：回复的第一个字节是"$"

​	数组：回复的第一个字节是"*"

​	每行信息以换行符结尾

​	*后面数量表示存在几个$

​	$后面数量表示字符串的长度

`SET myKey myValue` ：

```redis
*3\r\n    // 表示有3个$
$3\r\n    // 表示后面跟着的字符串SET为3位
SET\r\n   // SET命令
$5\r\n
mykey\r\n
$7\r\n
myValue\r\n
```



#### 集群

* Redis Sharding

  客户端分区方案，通过一致性哈希算法决定映射的节点

* 主从复制

  主节点定期把数据复制到从节点

  * 同步机制
    1. 全量同步：从节点初始化时，向主节点发送`psync?-1`命令，主节点进行RDB操作并返回此文件，同时记录此期间新增的命令也一起返回。之后主节点会将当前命令都发送一遍给从节点
    
    2. 部分同步：从节点断连重连时，向主节点发送`psync <master_id> <offset>`命令，通过判断ID和偏移量来判断是否需要对断连后的命令进行同步

  <img src="http://img.miilnvo.xyz/29yk3.jpg" alt="29yk3" style="zoom: 70%;" />

  * 特点

    1. 数据备份：主从复制实现了数据的热备份，并且当主节点出现问题时可以由从节点提供服务

    2. 负载均衡：在主从复制的基础上，配合读写分离，可以由主节点提供写服务，由从节点提供读服务（手动切换）

    3. 高可用基石：哨兵和集群能够实施的基础

* 哨兵（Sentinel）

  在主从复制的基础上加上Sentinel节点进行监控，解决了故障时自动转移的问题

* Redis Cluster（v3.0）

  哈希槽：先用CRC16算法，再把结果对16384求余数，最后把值放到对应范围间的Redis节点中（0-16383）

  <img src="http://img.miilnvo.xyz/6pe3x.png" alt="image-20200827162819831" style="zoom:50%;" />

  连接任意节点后，根据同样的算法在内部跳转到相应的节点上获取值
  
  无中心化结构，每个节点都通过特殊的二进制协议与其它的节点进行通信
  
  扩容时需要人工介入，把部分哈希槽导入到新节点中

​       主节点失效后要依赖从节点，若超过一半以上的节点失效，那么整个集群不可用。最小配置3主3从

​       CAP理论：牺牲C（强一致性），从机和各个主从间相互关联保证P（分区容忍性）

* 代理分区
  * Twemproxy
  * Codis

【参考】

<https://juejin.im/post/5b8fc5536fb9a05d2d01fb11#heading-14>

https://zhuanlan.zhihu.com/p/129640817

https://www.pianshen.com/article/9575185943/



#### 分布式锁

> 分布式与单机情况下最大的不同在于其不是多线程而是多进程。多线程由于可以共享堆内存，因此可以简单的采用内存作为标记存储位置。而进程之间甚至可能都不在同一台物理机上，因此需要将标记存储在一个所有进程都能看到的地方，例如Redis

* 单机部署

  * 加锁

    ```redis
    - value可以是UUID
    SET key value NX PX 30000
    ```

    ```java
    String result = jedis.set(String key,
                              String value,
                              String nxxx,  // nxxx：当Key不存在时才进行Set操作
                              String expx,  // expx：过期时间，避免当前客户端挂掉后一直无法释放锁  
                              int time);
    ```
    
  * 解锁

    ```lua
    - 释放锁（Lua脚本）
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    ```
    
    ```java
    String script = "if redis.call('get', KEYS[1]) == ARGV[1] then"
                    + "return redis.call('del', KEYS[1])"
    				+ "else return 0 end";
    // eval命令具有原子性
    Object result = jedis.eval(script,
                               Collections.singletonList(lockKey),
                               Collections.singletonList(requestId));
    ```

  value值要具有唯一性，释放锁时要验证value值，防止其它进程误解锁

  > 在Lock的实现中也有类似的逻辑，解锁时会判断当前线程与获取锁的线程是否一致

  为了保证过期时间大于业务执行时间，在其它线程中定期延长过期时间的操作

  【参考】

  <https://wudashan.cn/2017/10/23/Redis-Distributed-Lock-Implement/>

  <https://juejin.im/post/5b737b9b518825613d3894f4>

* 集群部署  

  一般集群的缺陷：客户端A先在master节点拿到了锁，master节点在把A创建的Key写入slave之前宕机了（主从复制是异步的），slave变成了master节点。因为这时的slave里还没有A持有锁的信息，所以客户端B也可以得到和A还持有的相同的锁

  Redlock（红锁）算法：从单节点变成多个多节点，要求节点间完全互相独立，不存在主从复制或者其他集群协调机制（即重新部署一个专门的集群）。当且仅当从大多数的Redis节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功

  【参考】

  <http://redis.cn/topics/distlock.html>



#### Redisson

提供了很多用于分布式环境的对象和服务，源码中使用了大量的Lua语句来保证命令的原子性，底层使用Netty框架

```java
// 实现了Redlock介绍的加锁算法
RLock lock1 = redissonInstance1.getLock("lock1");
RLock lock2 = redissonInstance2.getLock("lock2");
RLock lock3 = redissonInstance3.getLock("lock3");
RedissonRedLock lock = new RedissonRedLock(lock1, lock2, lock3);
// 同时加锁：lock1 lock2 lock3
// 在大部分节点上加锁成功就算成功。
lock.lock();
// ...
lock.unlock();
```



#### RedisTemplate

Jedis是Redis官方推荐的面向Java的操作Redis的客户端，而RedisTemplate是SpringDataRedis中对Jedis Api的封装

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

默认使用JdkSerializationRedisSerializer转换成字节数组的形式进行序列化，如果Key和Value都是String类型，那么应该使用StringRedisSerializer，即StringRedisTemplate的默认配置

```java
redisTemplate.opsForValue();  // 操作String
redisTemplate.opsForHash();   // 操作Hash
redisTemplate.opsForList();   // 操作List
redisTemplate.opsForSet();    // 操作Set
redisTemplate.opsForZSet();   // 操作Zset

// 执行事务
SessionCallback sessionCallback = new SessionCallback() {
    @Override
    public Object execute(RedisOperations operation) throws DataAccessException {
        operation.multi();
        // 其它命令
        return operation.exec();
    }
};
redisTemplate.execute(sessionCallback);
```



#### 其它问题

* 缓存问题

  * 缓存雪崩

    现象：一般缓存都是定时任务去刷新，或者是查不到之后去更新的，定时任务刷新就有一个问题。大量Key同时失效，导致所有请求都直接访问数据库

    方案：在每个Key的过期时间上加个随机值，热点数据取消过期时间，有修改时再更新，流控

  * 缓存穿透

    现象：用户不断发起缓存和数据库中都没有此数据的请求，导致数据库压力过大

    方案：参数校验

  * 缓存击穿

    现象：是指一个Key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个Key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库

    方案：热点数据取消过期时间

    > 与雪崩的区别在于前者是单点，后者是大面积

* 使用`scan`命令代替`keys`命令，避免数据量大时卡顿

  【参考】

  https://www.jianshu.com/p/4370bc75f5a6

  https://blog.csdn.net/raoxiaoya/article/details/93639617

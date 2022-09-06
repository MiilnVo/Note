### Kafka

采用Scala语言开发的分布式消息中间件



#### 架构

<img src="http://img.miilnvo.xyz/1k8q7.png" alt="1k8q7" style="zoom:50%;" />



#### 核心概念

<img src="http://img.miilnvo.xyz/03akk.png" alt="03akk" style="zoom:50%;" />

* 主题（Topic）

  逻辑概念，以此进行归类，可以跨越多个Broker

* 分区（Partition）

  同一主题下不同的分区包含的消息不同

  分区之间的消息不保证有序

  新增Broker后在原来Broker上的主题分区不会自动均衡到新的Broker，只能通过手动的重新分配

  分区数也有上限，不能超过Linux的文件描述符数量

* 副本（Replica）
  
  * Leader副本：负责读写请求
  * Follower副本：只负责与leader进行消息同步
  
  同一分区下的不同副本包含的消息相同（同一时刻未必完全相同）
  
  AR（所有副本） = ISR（已同步的副本） + OSR（同步滞后的副本）
  
  当Leader副本发生故障时，只有ISR中的副本才有资格被选举为Leader
  
  副本失效：同步时间超时

> 一个分区对应一份物理文件，每次消息来都是顺序写这份文件，并且是定时刷盘（写效率高）。但是当分区数量变多，总体上就会从顺序写变为随机写



#### 偏移量

<img src="http://img.miilnvo.xyz/4nbwp.png" alt="4nbwp" style="zoom:50%;" />

HW：下一条可消费消息的offset

LEO：下一条待写入消息的offset

> ISR中最小的LEO即为整个分区的HW



#### 生产者

![gfqnw](http://img.miilnvo.xyz/gfqnw.png)

KafkaProducer是线程安全的，由两个线程（主线程+Sender线程）协调运行

* 发送模式

  * 发后即忘：`producer.send()`
  * 同步：`producer.send().get()`
  * 异步：`producer.send(record, new Callback() {...})`

* 发送参数（消息可靠性）

  * acks = 1：只要Leader副本成功

  * acks = 0：不需要等待任何响应

  * acks = -1：所有副本都成功

* 发送异常

  * 可重试：网络、分区Leader不可用等异常，会按照retries参数的值重试

  * 不可重试：消息太大等异常

* 发送流程

  1. 拦截器

     优先于Callback前执行

  2. 序列化器

  3. 分区器

     如果ProducerRecord没有指定Partition，那么要根据Key来计算Partition：如果Key不为null，默认按照哈希算法，如果Key为null，则以轮训的方式

  4. 消息累加器（RecordAccumulator）

     为每个分区缓存消息后批量发送

     ProducerRecord x N

     => ProducerBatch

     ​	=> <分区，Deque\<ProducerBatch\>> 

     ​		=> <Node，List\<ProducerBatch\>>

     ​			=> <Node，Request>

  5. 请求中的缓存（InFlightRequests）
  
     缓存已发出去但还没收到响应的请求



#### 消费者

<img src="http://img.miilnvo.xyz/djl7h.png" alt="djl7h" style="zoom:50%;" />

```java
Properties props = new Properties();
props.setProperty("bootstrap.servers", "localhost:9092");
props.setProperty("group.id", "1");  // 设置消费组
props.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList(Topic));  // 订阅主题
consumer.assign(Arrays.asList(Partition));  // 订阅分区
```

每条消息只会投递到每个消费组的**其中**一个消费者

一个消费者及消费组可以订阅多个主题及分区

消费是拉模式（循环调用`poll()`方法），一次拉取的消息包含了多个分区

KafkaConsumer非线程安全，无法在多个线程间共用同一个Consumer

* 位移提交
  * 自动提交：默认定期每五秒提交最大的消息位移
  * 手动同步提交：阻塞当前线程直到提交成功
  * 手动异步提交：额外保存递增的序号来避免重复消费的问题

* 重平衡

  规定了如何让消费组下的所有消费这来分配每一个分区

  触发条件：消费者数变化、消费组数变化、分区数变化、主题数变化

  阶段：`FIND_COORDINATOR` => `JOIN_GROUP` => `SYNC GROUP` => `HEARTBEAT`

  缺点：平衡过程中无法消费消息

  通知：通过心跳线程来完成
  
  分区策略：Range（基于主题）、RoundRobin（基于分区）、Sticky（尽可能移动较少的分区进行平衡）



####消息存储

<img src="http://img.miilnvo.xyz/sfb1g.png" alt="sfb1g" style="zoom:50%;" />

<img src="http://img.miilnvo.xyz/91r1o.jpg" alt="91r1o" style="zoom:50%;" />

<img src="http://img.miilnvo.xyz/fij5h.png" alt="fij5h" style="zoom:50%;" />

根据消息数量的偏移量进行命名

__consumer_offsets-N：一种自动创建的特殊Topic，用于保存已消费的偏移量（消费者-位移提交）。里面的每条消息包含<Consumer Group，Topic，Partition\>和<Offset\>

* 消息格式

  v2版本，RecordBatch对应ProducerBatch

<img src="http://img.miilnvo.xyz/jwiav.png" alt="jwiav" style="zoom:50%;" />

* 索引

  稀疏索引 + 二分法

  * 偏移量索引（.index）

    相对偏移量（4B） + 物理地址（4B）

    ![96g7x](http://img.miilnvo.xyz/96g7x.png)

  * 时间戳索引（.timeindex）

    时间戳（8B）+ 相对偏移量（4B）

    ![sjwae](http://img.miilnvo.xyz/sjwae.png)

* 磁盘I/O



#### 协议

基于TCP的二进制协议

请求头（4个字段） + 请求内容 

ProduceRequest/ProduceResponse

FetchRequest/FetchResponse



#### 时间轮

![4kp20](http://img.miilnvo.xyz/4kp20.png)

底层是一个环形数组，而数组中每个元素都存放一个双向链表TimerTaskList，链表中封装了很多延时任务

多层：类似钟表的设计

复用：已经过的地方可以给其他时间节点使用

降级：上层时间轮的任务添加到下层时间轮并准备执行

推进：利用DelayQueue来进行时间推进

![ogloo](http://img.miilnvo.xyz/ogloo.png)

> 每个bucket是一个双向链表，等价于TimerTaskList

【参考】

https://juejin.cn/post/6844904110399946766

https://xie.infoq.cn/article/c789ee7f317d4b62ff88c7087



#### 控制器

有一个Broker会被选举为控制器，由其负责分区Leader的选举

> Leader是针对分区副本的，Controller是针对控制器的

> 默认选举策略为从AR顺序查找第一个在ISR中的副本



#### 幂等&事务

在Broker端按照分区建立了唯一PID，可以保证分区下的消息Exactly Once



#### 部署

```bash
# 启动ZooKeeper
$ bin/zookeeper-server-start.sh config/zookeeper.properties
# 启动Broker
$ bin/kafka-server-start.sh config/server.properties
# 创建TopicX （--replication-factor 2 --partitions 4）
$ bin/kafka-topics.sh --create --topic TopicX --bootstrap-server localhost:9092
# 查看Topic信息
$ bin/kafka-topics.sh --describe --topic TopicX --bootstrap-server localhost:9092
# 消费消息
$ bin/kafka-console-consumer.sh --topic TopicX --from-beginning --bootstrap-server localhost:9092
```



#### 其他

![6vih0](http://img.miilnvo.xyz/6vih0.png)

因为Kafka的每个分区都会对应一个物理文件，当Topic数量增加时，分区数量也同时增加，消息分散的落盘策略反而会导致磁盘IO竞争激烈进而成为瓶颈



为什么不支持读写分离：可以简化代码的实现逻辑；将负载粒度细化均摊；与主写从读相比，负载性能更好；没有延迟的影响

提高可靠性：使用3~5个副本，设置ack=-1并设置最小ISR副本数，同步刷盘（不推荐），手动提交位移

【参考】https://blog.csdn.net/ETTTTTSS/article/details/103553324

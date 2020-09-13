### RabbitMQ

消息队列的三大作用：解耦、异步、削峰



#### AMQP协议

基于TCP的应用层协议，RabbitMQ是此协议的Erlang的实现

![byiw1](http://img.miilnvo.xyz/byiw1.png)



#### Connection & Channel

在一个TCP连接（Connection）上可以创建多个信道（Channel），建议每个线程持有一个信道

```java
ConnectionFactory connectionFactory = new ConnectionFactory();
connectionFactory.setHost("127.0.0.1");
connectionFactory.setPort(5672);
connectionFactory.setUsername("guest");
connectionFactory.setPassword("guest");
Connection connection = connectionFactory.newConnection();
Channel channel = connection.createChannel();
```



#### Exchange

* Direct：单条路由，转发到与RouteKey完全匹配的队列
* Topic：多条路由，转发到由点号分割的RouteKey相匹配的队列
* Fanout：广播，转发到所有与交换器绑定的队列
* Headers：转发到头部信息中规定的队列



#### API

> Channel对象的方法

* 创建交换器
  ```java
  Exchange.DeclareOk exchangeDeclare(
      String exchange,       // 交换器名称
      String type,           // 交换器类型
      boolean durable,       // 是否持久化
      boolean autoDelete,    // 是否自动删除(当所有绑定的队列或交换器都解绑时)
      boolean internal,      // 是否内置(生产者无法直接发送到此交换器)
      Map<String, Object> arguments    // 其它参数
  ) throws IOException;
  ```
  ```java
  arguments.put("alternate-echange", "myAE");  // 添加备份交换器，优先级比mandatory参数大
  ```
  `exchangeDeclareNoWait(...)`方法：创建后不需要等待服务器的返回结果

  `exchangeDeclarePassive(String exchange)`方法：检测交换器是否存在

  * 备份交换器

    ![5yv9d](http://img.miilnvo.xyz/5yv9d.png)

* 创建队列
  ```java
  Queue.DeclareOk queueDeclare(
      String queue,        // 队列名称
      boolean durable,     // 是否持久化
      boolean exclusive,   // 是否排他(只对首次声明它的Connection可见，断开时自动删除)
      boolean autoDelete,  // 是否自动删除(当所有连接的消费者都断开时)
      Map<String, Object> arguments    // 其它参数
  ) throws IOException;
  ```
  ```java
  arguments.put("x-message-ttl", 6000);   // 设置队列中所有消息的TTL
  arguments.put("x-expires", 1000);       // 设置队列被自动删除前处于未使用状态的时间
  arguments.put("x-dead-letter-exchange", "myDLX");    // 添加死信交换器
  arguments.put("x-max-priority", 10);    // 设置队列中消息的最大优先级为10
  ```
  `queueDeclareNoWait(...)`方法：创建后不需要等待服务器的返回结果

  `queueDeclarePassive(String queue)`方法：检测队列是否存在

  * <span id = "tips">死信交换器和死信队列</span>

    消息变成死信的原因：消息被拒绝、消息过期、队列达到最大长度

    变成死信的消息会被投递到死信交换器和死信队列

    > 如果消费者订阅的不是queue.normal队列而是queue.dlx队列，那么死信队列就变成了延迟队列

    ![zyx1j](http://img.miilnvo.xyz/zyx1j.png)

* 队列与交换器绑定
  ```java
  Queue.BindOk queueBind(
      String queue,       // 队列名称
      String exchange,    // 交换器名称
      String routingKey,  // 路由键
      Map<String, Object> arguments    // 其它参数
  ) throws IOException;
  ```

* 交换器相互绑定
  ```java
  Exange.BindOk exchangeBind(
      String desination,  // 后一个交换器
      String source,      // 前一个交换器
      String routingKey,  // 路由键
      Map<String, Object> arguments   // 其它参数
  ) throws IOException;
  ```

* 发送消息
  ```java
  void basicPublish(
      String exchange,    // 交换器名称
      String routingKey,  // 路由键
      boolean mandatory,  // 是否把消息返回给生产者
      boolean immediate,  // 是否把消息返回给生产者
      BasicProperties props,  // 消息属性
      byte[] body         // 消息内容
  ) throws IOException;
  ```
  ```java
  AMQP.BasicProperties.Builder builder = new AMQP.BasicProperties.Builder();
  bulider.deliverMode(2);      // 设置消息持久化(当队列也设置了持久化才有意义)
  builder.expiration("6000");  // 设置每条消息的TTL为6000ms(只有当消息到达队列头部时才会判断是否过期，而在队列中的消息即使过期也不会被移除)
  builder.priority(5);         // 设置每条消息的优先级为5
  AMQP.BasicProperties props = builder.build();
  ```
  * `mandatory`参数

    当交换器找不到队列时会把消息返回给生产者，生产者需要调用`ReturnListener`监听器

    ```java
    channel.addReturnListener(new ReturnListener() {
        @Override
        public void handleReturn(int replyCode,
                                 String replyText,
                                 String exchange,
                                 String routingKey,
                                 AMQP.BasicProperties props,
                                 byte[] body) throws IOException {
            // ...消息处理代码
        }
    });
    ```

  * `immediate`参数

    当队列找不到消费者时会把消息返回给生产者（已弃用，[可以用TTL和DLX的方法代替](#tips)）

* 消费消息：
  * 推模式：持续订阅

    ```java
    String basicConsume(
        String queue,       // 队列名称
        boolean autoAck,    // 是否自动确认
        String consumeTag,  // 消费者标签
        boolean noLocal,    // 是否可以将同一个Connection中生产者发送的消息传给消费者
        boolean exclusive,  // 是否排他(只对首次声明它的Connection可见，断开时自动删除)
        Map<String, Object> arguments,  // 其它参数
        Consumer callback   // 回调函数
    ) throws IOException;
    ```
    ```java
    Consumer callback = new DefaultConsumer(channel) {
        @Override
        public void handleDelivery(String consumerTag,
                                   Envelope envelope,
                                   AMQP.BasicProperties props,
                                   byte[] body) throws IOException {
        // ...消息处理代码
        channel.basicAck(envelope.getDeliveryTag(), false);  // 手动确认
        }
        
        handleConsumeOk(String consumerTag) {...}      // 会在其它方法之前调用
        handleCancelOk(String consumerTag) {...}       // 显示取消消费者的订阅
        hangdleCanCel(String consumerTag) {...}        // 隐示取消消费者的订阅
        handleShutdownSinal(String consumerTag) {...}  // Connection或Channel关闭时调用
        handleRecoverOk(String consumerTag) {...}      // （略）
    };
    ```

  * 拉模式：单条

    ```java
    GetResponse basicGet(
        String queue,    // 队列名称
        boolean autoAck  // 是否自动确认
    ) throws IOException;
    ```



#### 生产者确认

* 事务

  只有消息被成功接收，事务才算提交成功，此方案会严重降低吞吐量

  ```java
  try {
      channel.txSelect();     // 事务开启
      channel.basicPublish(exchange, routingKey, properties, msg);
      int result = 1 / 0;
      channel.txCommit();     // 事务提交
  } catch (Exception e) {
      channel.txRollback();   // 事务回滚
  }
  ```

* 确认机制

  所有消息都会加上一个唯一ID，持久化后返回的确认消息会包含此ID

  * 同步确认：性能只比事务略高一点

    ```java
    channel.confirmSelect();     // 设置为确认模式
    channel.basicPublish(exchange, routingKey, properties, msg);
    if (!channel.waitForConfirms()) {
        // ...错误处理
    }
    ```

  * 批量确认：性能大幅提高，但如果出现异常，需要把这一批次的消息全部重发
    ```java
    channel.confirmSelect();
    int msgCount = 0;
    while (true) {
        channel.basicPublish(exchange, routingKey, properties, msg);
        if (++msgCount >= BATCH_COUNT) {
            msgCount = 0;
            try {
                if (channel.waitForConfirms()) {
                    // ...清空缓存中的消息
                } else {
             		// ...重新发送缓存中的消息       
                }
            } catch (InterruptedException e) {
                // ...重新发送缓存中的消息
            }
        }
    }
    ```

  * 异步确认：性能与批量确认相同

    `deliveryTag`参数表示消息的唯一ID

    `multiple`参数表示到这个序号之前的所有消息都已经得到了处理，提供了多条确认的机制

    ```java
    channel.confirmSelect();
    channel.addConfirmListener(new ConfirmListener() {
        @Override
    	public void handleAck(long deliveryTag, boolean multiple) throws IOException 	{
            if (mutiple) {
                confirmSet.headSet(deliveryTag + 1).clear();
            } else {
                confirmSet.remove(deliveryTag);
            }
        }
        @Override
        public void handleNack(long deliveryTag, boolean multiple) throws IOException {
            if (multiple) {
                confirmSet.headSet(deliveryTag + 1).clear();
            } else {
                confirmSet.remove(deliveryTag);
            }
            // ...重新发送缓存中的消息
        }
    });
    ```



#### 消费者确认

消息在等待消费者确认之后才会从内存或磁盘中移除

当消费者没有确认且断开连接时，消息会重新进入队列等待下一次发送（存在重复消费的问题）

> 只要与消费者的连接不断，就会一直等待确认消息




#### 延迟队列

1. 设置队列中所有消息的TTL
2. 设置DLX
3. 消费者订阅死信队列

   > 只适用于预设延迟时间的场景，无法利用每条消息的TTL来进行精确地定时发送



#### 优先级队列

1. 设置队列中消息的最大优先级
2. 设置发送时每条消息的优先级

   > 只有当队列发生消息堆积时，优先级才有意义，优先级高的消息先被消费

   > 即使优先级较高的消息先过期了，也不会被直接移除
   
   > 堆结构



#### 消息分发

当队列拥有多个消费者时，消息将以轮询的方式发送

Channel#`basicQos(10)`方法：设定消费者最多未确定消息的数量为10条，可以避免单个消费者因执行缓慢而堆积了过多的消息

在不使用任何高级特性，也没有消息丢失、网络故障之类的异常，并且只有一个生产者和消费者的时候，才可以保证顺序性



#### 消息传输保障

| 级别 | 表现 | 方式 |
| :----- | :--- | :--- |
| 最多一次 | 消息可能会丢失，但绝不会重复传输 | 默认 |
| 最少一次 | 消息绝不会丢失，但可能重复传输 | 生产者 - 交换器：生产者确认<br />交换器 - 队列：mandatory参数或备份交换器<br />交换器 / 队列：设置持久化<br />队列 - 消费者：手动确认 |
| 正好一次 | 每条消息肯定会被传输一次且仅此一次 | 不支持，只能在客户端实现去重 |



#### 集群

* 普通模式

  只会同步元数据，而不会复制队列中的数据（数据只存在其中一个节点中），节点间要通过路由转发才能找到目标节点中的数据

  * 优点：可以通过节点扩容提高消息积压能力（空间角度），不需要将消息复制到每一个从节点，减少了网络和磁盘的开销（性能角度）

  * 缺点：单节点宕机会导致该节点的数据丢失

* 镜像模式（v2.6）

  可以理解为拥有一个隐藏的Fanout交换器，会将消息分发到其它的从队列上（数据存在多个节点中）

  * 优点：实现HA

  * 缺点：集群内部的网络带宽将会被这种大量的同步通讯消耗掉，无法线性扩容

  ![wvnt0](http://img.miilnvo.xyz/wvnt0.png)

【参考】

<https://blog.51cto.com/11134648/2155934>

<https://juejin.im/post/5b586b125188257bcb59005e>
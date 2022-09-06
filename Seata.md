### Seata



#### 架构

![85kag](http://img.miilnvo.xyz/85kag.png)

* TC（事务协调者）：维护全局和分支事务的状态

* TM（事务管理器）：负责定义事务的范围，负责全局事务的开启、提交和回滚

  > 例如Business调用其他三个服务，则在Business中开启全局事务@GlobalTransactional

* RM（资源管理器）：事务执行者，控制自己分支的事务



#### CAP

1998年 Eric Brewer 提出分布式系统有三个指标，且这三个指标不可能同时满足：

* **C**onsistency（一致性）

  写操作后的读操作必须返回最新值，有可能因为多个服务间的同步延迟导致不一致

* **A**vailability（可用性）

  只要收到请求就必须给出回应（无论对错），有可能因为一致性的锁定而导致不可用

* **P**artition Tolerance（分区容忍性）

  大多数分布式系统都部署在不同的子网络（区间）中，有可能因网络问题产生区间通信失败


> 因为P无法避免，所以只能满足，导致要在C和A中进行取舍

> CA：MySQL、Oracle等单点架构
>
> CP：Zookeeper、Nacos、Seata（XA）
>
> AP：Eureka、Redis、RabbitMQ、RocketMQ、Nacos、Seata（AT、TCC、Sage）



#### BASE

* **Ba**sically Available（基本可用）

  系统在出现故障的时候，允许损失部分可用性，保证核心系统可用

* **S**oft State（软状态）

  允许系统在不同节点间副本同步的时候存在延时，可以接受在一段时间内不同步

* **E**ventually Consistent（最终一致性）

  系统中所有的数据副本在经过一段时间的同步后最终能达到一致的状态

这是对CAP中一致性和可用性权衡的结果：我们无法做到强一致，但每个应用都可以根据自身的业务特点，采用适当的方式来使系统达到最终一致性



#### 分布式事务

* 种类：跨多个数据源、跨多个服务和数据源
* 解决思路：
  * CP：各个分支事务执行后相互等待，最后全部提交，全部回滚，保证强一致性。只是等待过程中事务不可用，牺牲了可用性
  * AP：各个分支事务分别执行和提交，牺牲了一致性



#### 四种模式

##### AT

<img src="http://img.miilnvo.xyz/2wo66.png" alt="2wo66" style="zoom:50%;" />

第一阶段：执行本地事务

<img src="http://img.miilnvo.xyz/wepuw.png" alt="image-20220825103231522" style="zoom:50%;" />

```sql
-- undo_log的表结构
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,  -- 保存了JSON格式的beforeImage和afterImage
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;  
```

第二阶段：全部移除反向操作或执行反向操作

<img src="http://img.miilnvo.xyz/7vb26.png" alt="image-20220825103652178" style="zoom:50%;" />

第一阶段和第二阶段的提交和反向操作均由框架自动生成，用户只需专注编写业务SQL即可，解决了TCC中的代码侵入

> 可以通过数据库的binlog查看到反向操作的SQL

> 优点：无侵入式，适用于不希望对业务进行改造的场景



##### TCC

第一阶段（**T**ry）：资源的检测和预留/冻结

第二阶段（**C**onfirm / **C**ancel）：全部提交或回滚，对资源进行释放，即Try阶段的反向操作（业务层面）

每个阶段都会提交本地事务并释放锁，并不需要等待其它事务的执行结果

> 优点：高性能式，适用于对性能有很高要求的场景
>
> 缺点：代码侵入高，开发成本高



##### Saga

长事务解决方案，适用于业务流程长且需要保证事务最终一致性的场景



##### XA

即2PC（两阶段提交）

第一阶段：准备阶段

第二阶段：全部提交或回滚（数据库层面）

> 优点：强一致性
>
> 缺点：需要等待其它事务的执行结果，阻塞时间较长，性能较差



#### 元配置

registry.conf

```conf
# 注册中心信息：为了相互建立连接
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "file"
  # ...
}

# 配置中心信息：指定配置文件的位置
config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "file"  # 默认为file类型，即会加载本地file.conf里面的属性
  nacos {
    serverAddr = "localhost"
    namespace = ""
  }
  file {
    name = "file.conf"
  }
  # ...
}
```

> 服务端（TC）和客户端（TM、RM）各有一套registry.conf和file.conf

【参考】事务分组：https://blog.csdn.net/zhuocailing3390/article/details/123377195

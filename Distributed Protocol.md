### Distributed Protocol



#### CAP

1998年 Eric Brewer 提出分布式系统有三个指标，且这三个指标不可能同时满足：

* **C**onsistency（一致性）

  写操作后的读操作必须返回最新值，有可能因为多个服务间的同步延迟导致不一致

* **A**vailability（可用性）

  只要收到请求就必须给出回应（无论对错），有可能因为一致性的锁定而导致不可用

* **P**artition Tolerance（分区容忍性）

  大多数分布式系统都部署在不同的子网络（区间）中，有可能因网络问题产生区间通信失败


> 因为P无法避免，所以必须满足，导致要在C和A中进行取舍

> CA：MySQL、Oracle等单点架构
>
> CP：ZooKeeper、Nacos（配置管理、持久化服务注册发现）、Seata（XA）
>
> AP：Eureka、Redis、RabbitMQ、RocketMQ、Nacos（临时服务注册发现）、Seata（AT、TCC、Sage）



#### BASE

* **Ba**sically Available（基本可用）

  系统在出现故障的时候，允许损失部分可用性，保证核心系统可用

* **S**oft State（软状态）

  允许系统在不同节点间副本同步的时候存在延时，可以接受在一段时间内不同步

* **E**ventually Consistent（最终一致性）

  系统中所有的数据副本在经过一段时间的同步后最终能达到一致的状态

这是对CAP中一致性和可用性权衡的结果：我们无法做到强一致，但每个应用都可以根据自身的业务特点，采用适当的方式来使系统达到最终一致性

> AP牺牲了一致性，只是不再要求强一致性，而是能达到最终一致性即可
>
> 弱一致性 / 最终一致性：DNS
>
> 强一致性：Paxos、Raft、ZAB



#### Paxos协议

##### 角色

Client（民众）

Propser（议员）

Accepetor（国会）

Leraner（记录员）

##### 存在的问题

活锁和执行顺序



#### Raft协议

##### 角色

Leader（领袖）：领袖由群众投票选举得出，每次选举，只能选出一名领袖

Follower（群众）：可以进行投票

Candidate（候选人）：当没有领袖时，某些群众可以成为候选人，然后去竞争领袖的位置

##### 步骤

1. Follower在经过随机的选举定时时间后变成Candidate（可能同时会有多个Candidate）
2. Candidate给每个Follower发送选票，Follower返回选票
3. 当第一个Candidate获取超过一半的选票后便成为Leader
4. Leader会周期性的发送心跳给所有的Follower，Follower收到心跳后会重置选举定时时间

5. 当Leader挂掉时：Follower的选举定时时间超时后会重新执行步骤1~4
6. 当发生脑裂时：只有拥有大多数Follower的网络可以正常执行请求。网络恢复后，旧Leader发现网络中的新Leader的Term比自己大，则自动降级为Follower

【参考】https://raft.github.io/



#### ZAB协议

##### 角色

Leader（领袖）：Leader负责接收所有请求

Follower（群众）：参与选举，提供读服务

Observer（观察者）：没有投票权的Follower，仅提供读服务

##### 步骤

（略）

【参考】Raft和ZAB的区别：https://zhuanlan.zhihu.com/p/543583983



#### 脑裂

##### 定义

集群中的Master或Leader节点往往是通过选举产生的，但当两个机房之间的网络通信出现故障时，选举机制就有可能在不同的网络分区中选出两个Leader

##### 如何解决

（1）部署奇数节点，选举时少数服从多数（节点投票数 > 总节点数 / 2）

（2）每当新Leader产生时，会生成一个epoch标号（标识当前属于那个Leader的统治时期），epoch是递增的，followers如果确认了新的Leader存在，知道其epoch，就会拒绝epoch小于现任Leader epoch的所有请求

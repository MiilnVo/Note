### Nacos

全称：Dynamic **Na**ming and **Co**nfiguration **S**ervice

作用：服务注册发现、配置管理



#### 总体设计

##### 架构

![keizv](http://img.miilnvo.com/keizv.png)

* Nacos Server
  * 接口层：OpenAPI、Console、SDK、Agent、CLI

  * 业务层：服务注册发现、配置管理、元数据管理
  * 内核层：略
  * 插件：略
* Nacos Client
* Nacos Console

##### 数据模型

Namespace + Group + Service / Data Id



#### 	核设计

##### 一致性协议

* Distro（AP）：临时实例的服务注册发现

  > 写操作会转发到责任节点后操作。读操作直接响应，因为每台机子都存放了全量数据

* Raft（CP）：配置管理、持久化实例的服务注册发现

【参考】https://blog.csdn.net/yaoxie1534/article/details/125657808



#### 服务注册发现

```xml
nacos-config-spring-boot-starter.jar
spring-cloud-starter-alibaba-nacos-discovery.jar
```

##### Service模型

服务 - 集群 - 实例

##### 服务分类

> 依照ephemeral字段

临时实例/服务 和 持久化实例/服务

##### 注册原理

客户端的服务启动时，通过监听事件（ApplicationListener）触发后，把参数拼接后通过RESTful接口请求服务端。服务端的服务注册表结构：

```java
// Map(Namespace, Map(Group::serviceName, Service/Data Id))
ServiceManager.Map<String, Map<String, Service>>
```

> 注册时会丢到一个阻塞队列中，异步+死循环处理

##### 保护阈值（0~1）

当健康数的实例小于这个值后，启动保护，会把所有的实例拉取下来，在所有的实例中进行负载均衡（原先只会在健康的实例中进行负载均衡），这样做可以保护剩下健康的实例不被大量请求冲垮

##### 客户端权重

除了@LoadBalanced，还需要注入NacosRule

##### 通信方式

心跳检测 & 短轮询 + UDP推送：5秒发送一次<u>心跳</u>，超过15秒没有客户端的心跳会把实例标记为不健康，超过30秒会删除临时实例，但不会删除持久实例。服务端在发生心跳检测异常、服务列表变更或者健康状态改变时会触发推送ServiceChangeEvent事件，在推送事件中会基于<u>UDP</u>将最新的服务列表推送到各个客户端。虽然通过UDP不能保证消息的可靠抵达，但是由于Nacos客户端会开启定时任务，每隔一段时间更新客户端缓存的服务列表，通过定时<u>短轮询</u>更新服务列表做兜底，所以不用担心数据不会更新的情况。这样既保证了实时性，又保证了数据更新的可靠性

> 临时实例使用客户端心跳检测模式，而持久化实例使用服务端反向探测模式（默认时间间隔20秒）

##### 与其它注册中心的区别

Eureka（AP）：相互注册，完全去中心化，也就没有主从之分，只要有一台Eureka节点存在整个微服务就可以实现通讯

ZooKeeper（CP）：采用ZAB原子广播协议（基于2PC），当Leader宕机或出现了故障，会自动重新选举新的Leader，整个选举的过程中为了保证数据一致性，所有微服务无法实现通讯（本地有缓存除外）。还有可运行的节点必须满足过半机制，整个zk才可以使用

> Eureka不能支持大量的服务实例的原因：
>
> Eureka所有的服务实例在每一台Eureka Server中都保存了
>
> 大量的服务实例会产生大量的心跳检测等信息，导致Eureka Server无法支持高并发的操作

> ZooKeeper不能支持大量的服务实例的原因：
>
> ZooKeeper会将服务实例的上线下线通知到每一个服务实例
>
> 如果频繁的上下线的话，会去通知大量的服务实例，导致短时间网络压力增大，性能下降



#### 配置管理

```xml
nacos-discovery-spring-boot-starter.jar
spring-cloud-starter-alibaba-nacos-config.jar
```

Server间的协议：2种

SDK和Server间的协议：长连接 + MD5

@RefreshScope：在@Value注入变量所在的类上添加此注解，服务无需重启就可以感知配置的变动

集群模式启动需要本地数据库，集群模式配置的地址依然是单个地址

##### 通信方式

长轮询：客户端发起长轮询，如果服务端的数据没有发生变更，会Hold住请求（定时休眠），直到服务端的数据发生变化，或者等待一定时间超时会返回304

> 默认定时休眠30秒，所以客户端的超时时间需要比30秒长	

【参考】https://www.cnkirito.moe/nacos-and-longpolling


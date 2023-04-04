### Spring Cloud GateWay

> Nginx通常作为流量网关，GateWay通常作为应用网关



#### 三大概念

Route（路由）

Predicates（断言）

Filter（过滤器）

```properties
spring.cloud.gateway.routes[0].id=application-x
spring.cloud.gateway.routes[0].uri=http://xxx:8080
spring.cloud.gateway.routes[0].predicates[0]= Path=/data-compass/web/**
spring.cloud.gateway.routes[0].filters[0]= StripPrefix=1
spring.cloud.gateway.routes[0].filters[1]= AddUserInfo=systemId, 389
```



#### 流程

<img src="http://img.miilnvo.com/nf5pz.png" alt="nf5pz" style="zoom: 50%;" />



#### Predicate

```properties
spring.cloud.gateway.routes[0].predicates[N]= {类型N}={举例N}
```

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: xxx
        predicates:
        - {类型N}={举例N}
```

| 类型       | 可生效的请求         | 示例值                                                       |
| :--------- | :------------------- | ------------------------------------------------------------ |
| After      | 在某时间之后         | 2022-03-26T13:00:00+08:00[Asia/Shanghai]                     |
| Before     | 在某时间之前         | 2022-03-26T13:00:00+08:00[Asia/Shanghai]                     |
| Between    | 在时间范围内         | 2022-03-26T13:00:00+08:00[Asia/Shanghai], <br/>2022-03-27T18:00:00+08:00[Asia/Shanghai] |
| Cookies    | 匹配指定Cookie名和值 | mySessionName, c52e52258f86b61ab25fa0268be<br/>（name, regex） |
| Header     | 匹配指定Header名和值 | X-Request-Id, \d+<br/>（name, regex）                        |
| Host       | 匹配指定Host         | \*\*.miilnvo.com, \*\*.anotherhost.cn                        |
| Method     | 匹配HTTP方法         | GET, POST                                                    |
| Path       | 匹配指定路径         | /wage/list, /user/\*\*                                       |
| Query      | 匹配参数名及值       | green, pu.<br/>（name, regex）                               |
| RemoteAddr | 匹配IP地址           | 192.168.1.1/24                                               |
| Weigth     | 匹配组权重           | group1, 8<br/>（group, int）                                 |

【参考】https://blog.csdn.net/zhuocailing3390/article/details/123151458



#### Filter

##### 1. GateWayFilter

```properties
spring.cloud.gateway.routes[0].filters[N]= {类型N}={举例N}
```

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: xxx
        filters:
        - {类型N}={举例N}
```

| 类型                                                         | 作用                 | 示例值                                  |
| ------------------------------------------------------------ | -------------------- | --------------------------------------- |
| AddRequestHeader<br/>SetRequestHeader<br/>RemoveRequestHeader | 添加/修改/删除请求头 | X-Request-red, blue<br/>（key，value）  |
| AddRequestParameter<br/>RemoveRequestParameter               | 添加/删除请求参数    | name, jay<br/>（key，value）            |
| AddResponseHeader<br/>SetResponseHeader<br/>RemoveResponseHeader | 添加/修改/删除响应头 | X-Response-red, blue<br/>（key，value） |
| StripPrefix                                                  | 截断前N级的路径      | 1                                       |
| ......等31种                                                 |                      |                                         |

> 示例值中的（key，value）对应NameValueConfig类，只支持一组Key和Value

```yaml
filters:
- name: Hystrix
	args:
		name: fallbackCmdA
		fallbackUri: forward:/fallbackA
```

【参考】https://segmentfault.com/a/1190000040995083

##### 2. GlobalFilter

不需要在配置文件中配置，作用在所有的路由上

##### 3. 自定义Filter

两种Filter都要实现`GatewayFilter`/`GlobalFilter`和`Ordered`接口

```java
// GatewayFilter不需要加此注解
@Component
public class CustomGatewayFilter implements GatewayFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 从请求中获取是否有token参数
        String token = exchange.getRequest().getQueryParams().getFirst("token");
        if (token == null) {
            // 直接拒绝
            exchange.getResponse().setStatusCode(HttpStatus.NOT_ACCEPTABLE);
            return exchange.getResponse().setComplete();
        }
        // 通过这个过滤器，进入过滤链中的下一个过滤器
        return chain.filter(exchange);
    }
  
    @Override
    public int getOrder() {
        return 0;
    }
}
```

```java
// GatewayFilterFactory还需要实现此接口后进行配置，GlobalFilter则不需要
@Component
public class CustomGatewayFilterFactory extends AbstractGatewayFilterFactory<Object> {
		@Override
    public CustomGatewayFilter apply(Object config) {
        return new CustomGatewayFilter();
    }
}
```

```yaml
filters:
- Custom  #不需要加后缀
```

##### 4. 执行顺序

规则一：没有设置Order的Global Filter则Order默认值为`Ordered.LOWEST_PRECEDENCE`

规则二：没有设置Order的Route Filter的Order则从1开始，根据配置中定义的顺序再依次赋值

规则三：Order值越小，优先级越高，执行顺序越靠前

<img src="http://img.miilnvo.com/gfpno.png" alt="gfpno"  />

【参考】

https://cloud.tencent.com/developer/article/1424570

https://blog.51cto.com/u_15127655/4107292



#### 其他

* 与Nacos整合后无需添加任何配置，即可实现路由配置动态刷新

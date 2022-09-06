### Feign

> 目标：熟悉Spring Cloud常用组件的最佳实践和核心源码

定义：封装了HTTP请求的轻量级框架

特点：面向接口编程



#### 架构图

![601748-20200425103656218-1328309497](image/feign1.png)

* Contract：适配Feign、JAX-RS 1/2、Spring Web MVC等REST声明式规范，将方法的参数解析成MethodMetadata
* Encoder/Decoder：适配编码器/解码器

* Client：适配JDK HttpURLConnection、Apache HttpComponnets、OkHttp3
* 支持负载均衡与熔断：负载均衡 Ribbon是对Client进行包装。熔断器Hystrix在HystrixFeign中自定义InvocationHandlerFactory的实现，创建 HystrixInvocationHandler，对method.invoke请求做拦截



#### 简单API

```java
public static void main(String[] args) {
    // 1.生成代理对象
    FeignInterface target = Feign.builder()
        .decoder(new GsonDecoder())
        .client(new OkHttpClient())
        .target(FeignInterface.class, "http://127.0.0.1:8080");
    // 2.执行请求
    String result = target.createIssue("MiilnVo");
}
```



#### 源码分析

1. 生成代理对象：`Feign#target()`

   1.1 执行`Feign#build()`生成`ReflectiveFeign`

   1.2 执行`ReflectiveFeign#newInstance()`

   ​	1.2.1 执行`ParseHandlersByName#apply()`

   ​		1.2.1.1 执行`Contract#parseAndValidateMetadata()`：每个方法都将生成一个`MethodMetadata`

   ​		1.2.1.2 根据不同的类型（表单、Body）将`MethodMetadata`解析成`MethodHandler`（`SynchronousMethodHandler`）

   ​	1.2.2 执行`InvocationHandlerFactory#create()`：获得`InvocationHandler`

   ​	1.2.3 通过JDK反射获得代理对象

2. 执行请求

   2.1 执行 `ReflectiveFeign$FeignInvocationHandler#invoke()`

   ​	2.1.1 执行`SynchronousMethodHandler#invoke()`

   ​		2.1.1.1 执行`RequestTemplate$Factory#create()`：将方法参数转换成`RequestTemplate`

   ​		2.1.1.2 执行`SynchronousMethodHandler#executeAndDecode`

   ​			2.1.1.2.1 执行`RequestInterceptor#apply()`拦截器

   ​			2.1.1.2.2 执行`Client#execute()`：发起HTTP请求！

   ​			2.1.1.2.3 执行`Decoder#decode()`：对Response进行解码

   ​		2.1.1.3 若`executeAndDecode`抛异常则会休眠重试



#### OpenFeign

> 在Feign的基础上支持了Spring MVC的注解，且支持Spring Boot

**@FeignAutoConfiguration**

1. 添加spring-cloud-starter-openfeign的Jar包

2. Spring.factories文件引入了FeignAutoConfiguration

   ```xml
   org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
   org.springframework.cloud.openfeign.ribbon.FeignRibbonClientAutoConfiguration,\
   org.springframework.cloud.openfeign.FeignAutoConfiguration
   ```

3. FeignAutoConfiguration注入了一些Bean

   <img src="image/image-20210813152040385.png" alt="image-20210813152040385" style="zoom: 67%;" />

   * FeignContext：保存了每个FeignClient Context

     > 不同名称的@FeignClient都会初始化一个子Spring容器

   * Targeter：支持两种实现DefaultTargeter（即直接调用`feign#target()`）和HystrixTargeter

   * Client：支持两种实现ApacheHttpClient和OkHttpClient



**@EnableFeignClients**

1. 注册FeignClientFactoryBean

```java
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
    ...
}
```

```java
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {

    // 核心方法
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
                                        BeanDefinitionRegistry registry) {
        // 注册@EnableFeignClients#defaultConfiguration默认配置类
        registerDefaultConfiguration(metadata, registry);
        // 扫描所有的@FeignClient注解，生成FeignClientFactoryBean并注册到Spring容器中
        registerFeignClients(metadata, registry);
    }
    
}
```

2. 通过FeignClientFactoryBean生成Bean

```java
// FeignClientFactoryBean (FactoryBean实现类)
@Override
public Object getObject() throws Exception {
    return getTarget();
}

<T> T getTarget() {
    // FeignContext也是一种容器，是全局唯一的
    FeignContext context = this.applicationContext.getBean(FeignContext.class);
    
    // 每个FeignClientFactoryBean都拥有contextId属性
    // 通过contextId在FeignContext找到专属的AnnotationConfigApplicationContext
    // 在子Context中可以获取FeignClient各自的Builder、Logger、Encoder、Decoder、Contract等
    Feign.Builder builder = feign(context);

    // 省略了处理负载均衡Client的逻辑...
    
    Targeter targeter = get(context, Targeter.class);
    // 底层调用Feign#target()生成代理对象
    return (T) targeter.target(this, builder, context,
                               new HardCodedTarget<>(this.type, this.name, url));
}
```

![image-20210813165754383](image/image-20210813165754383.png)



#### 其它

适配器模式：OkHttpClient 继承 Feign的 Client接口，内部维护了okhttp3.OkHttpClient对象

https://www.cnblogs.com/binarylei/p/11563952.html#feign

https://juejin.cn/post/6948224170496884773
### Spring



#### IOC

全称：Inversion of Control（控制反转）

实现方式：Dependency Injection（依赖注入）

作用：分离提供资源和使用资源的双方，由第三方（Spring）提供资源（创建对象）

优点：对象集中管理，减低对象间的依赖关系



循环依赖的问题

1. 构造器循环依赖：无法解决

2. prototype范围的依赖：无法解决

3. setter循环依赖：暴露一个单例工厂



Bean的生命周期

`createBean()`

【1】InstantiationAwareBeanPostProcessor#`postProcessBeforeInstantiation()`

> 若此步返回的对象不为null，则执行第【11】步

​	 `doCreateBean()`

​    【2】通过反射实例化对象

​    【3】MergedBeanDefinitionPostProcessor#`postProcessMergedBeanDefinition()`

​    【4】创建ObjectFactory对象，实现`getObject()`方法，保存到缓存singletonFactories中，用于解决循环依赖

​	`populateBean()`

​	【5】InstantiationAwareBeanPostProcessor#`postProcessAfterInstantiation()`

​	【6】InstantiationAwareBeanPostProcessor#`postProcessProperties()`（对属性进行处理）

​	【7】属性赋值，如果依赖其它Bean则会调用`getBean()`方法获取，这里会存在循环依赖的问题

​	`initializeBean()`

​	【8】BeanPostProcessor#`postProcessBeforeInitialization()`（例如`@PostConstruct`注解）

​	【9】InitializingBean#`afterPropertiesSet()`（此Bean是InitializingBean子类才会执行）

​	【10】`init-method()`

​	【11】BeanPostProcessor#`postProcessAfterInitialization()`

容器销毁：

【12】DisposableBean#`afterPropertiesSet()`（此Bean是DisposableBean子类才会执行）

【13】`destroy－method()`

【参考】

https://www.jianshu.com/p/1dec08d290c1



@Autowired

由AutowiredAnnotationBeanPostProcessor进行处理，在【3】中寻找Bean中被注释的所有属性，然后封装成InjectedElement保存到缓存中，再在【6】中注入属性

@Resource





#####BeanFactory

```java
// 利用BeanDefinitionReader读取承载配置信息的Resource，通过XML解析器解析配置信息的DOM对象，生成对应的BeanDefinition对象
XmlBeanFactory beanFactory = new XmlBeanFactory(new ClassPathResource("spring.xml"));

// 懒加载
People people = (People) beanFactory.getBean("student");
```

```java
// Bean名称和Bean对象
Map<String, Object> singletonObjects

// Bean名称和创建Bean的工厂对象
Map<String, ObjectFactory<?>> singletonFactories

// Bean名称和Bean对象，表示Bean还在创建中（与singletonFactories互斥）
Map<String, Object> earlySingletonObjects

// 已经注册的Bean名称
Set<String> registeredSingletons

// 正在创建的Bean名称
Set<String> singletonsCurrentlyInCreation
```



AbstractBeanFactory#`getBean()` => `doGetBean()`

```java
// 1.转换对应的BeanName
final String beanName = transformedBeanName(name);
// 2.尝试从缓存中加载Bean
Object sharedInstance = getSingleton(beanName);
if (sharedInstance != null && args == null) {
    // 3.对FactoryBean进行处理，然后返回Bean
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
} else {
    // 4.处理parentBeanFactory的逻辑
    BeanFactory parentBeanFactory = getParentBeanFactory();
    // 5.对象转换
    final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
    // 6.寻找<bean>标签里depends-on的依赖，递归进行实例化
    String[] dependsOn = mbd.getDependsOn();
    if (dependsOn != null) {
        for (String dep : dependsOn) {
            registerDependentBean(dep, beanName);
            getBean(dep);
        }
    }
    // 7.根据不同的作用域创建对象
    if (mbd.isSingleton()) {
        sharedInstance = getSingleton(beanName, () -> {
            try {
                return createBean(beanName, mbd, args);
            }
        });
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    } else if (mbd.isPrototype()) {
        // ...
    } else {
        // ...
    }
    // 8.类型转换的逻辑
}
```


```java
// 2.尝试从缓存中加载
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    // 2-1.
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            // 2-2.
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                // 2-3.
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    // 2-4.执行回调方法
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

```java
// 7.根据不同的作用域创建对象
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    synchronized (this.singletonObjects) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            // 7-1.表示此Bean正在加载
            // 添加到singletonsCurrentlyInCreation中（为了解决循环依赖）
            beforeSingletonCreation(beanName);
            try {
                // 7-2.回调方法，准备创建Bean
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
            finally {
                // 7-3.表示此Bean已加载完成（为了解决循环依赖）
                afterSingletonCreation(beanName);
            }
            if (newSingleton) {
                // 7-4.添加到缓存中
                addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}
```

```java
// 7-2.回调方法，准备创建Bean
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {
    RootBeanDefinition mbdToUse = mbd;
    mbdToUse.prepareMethodOverrides();
    // 实例化的前置处理
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
        return bean;
    }
    // 常规创建
    Object beanInstance = doCreateBean(beanName, mbdToUse, args); 
    return beanInstance;
}
```

```java
// 常规创建
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    throws BeanCreationException {
    BeanWrapper instanceWrapper = null;
    if (instanceWrapper == null) {
        // 真正的创建对象！
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    // 这里的bean就是已经创建的对象，但属性值都为null
    final Object bean = instanceWrapper.getWrappedInstance();
    // 应用MergedBeanDefinitionPostProcessor
    // AutoWired？
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            mbd.postProcessed = true;
        }
    }

    // 是否需要提早曝光
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                      isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        // this.singletonFactories.put(beanName, singletonFactory);
        // 在2-3时会用到（解决循环依赖！），在调用getObject()前还会应用BeanPostProcessor
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    Object exposedObject = bean;
    try {
        // 属性注入，在这一步会遍历注入所有的属性
        populateBean(beanName, mbd, instanceWrapper);
        // 执行一些初始化后的方法
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    return exposedObject;
}

```



##### ApplicationContext

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        
        // 一些准备工作
        prepareRefresh();

        // 初始化BeanFactory（DefaultListableBeanFactory，XmlBeanFactory的父类）
        // 包括定制一些属性、加载XML等
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 配置一些特殊的Bean和一些特殊的规则等，即扩展了BeanFactory
        prepareBeanFactory(beanFactory);

        try {
            // 空方法
            postProcessBeanFactory(beanFactory);

            // 注册并激活BeanFactoryPostProcessor（容器级别的作用域）
            invokeBeanFactoryPostProcessors(beanFactory);

            // 注册后置处理器BeanPostProcessor
            registerBeanPostProcessors(beanFactory);

            // 国际化处理
            initMessageSource();

            // 注册广播器ApplicationEventMulticaster
            initApplicationEventMulticaster();

            // 空方法
            onRefresh();

            // 注册监听器ApplicationListener
            registerListeners();

            // 创建所有非延迟加载的单例
            finishBeanFactoryInitialization(beanFactory);

            // 执行一些通知（包括上面的事件监听器）
            finishRefresh();
        }

        catch (BeansException ex) {
            
            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // 
            resetCommonCaches();
        }
    }
}
```

BeanFactory和ApplicationContext的区别：

1. 前者是延迟加载
2. 前者使用BeanPostProcessor、BeanFactoryPostProcessor等需要手动加载
3. 前者不支持AOP（个人理解）

4. 后者提供了国际化、事件机制等



#### AOP

全称：Aspect Oriented Programming（面向切面编程）

实现方式：由代理模式生成代理对象，再对满足条件的方法进行拦截，执行通知

作用：将系统中非核心的业务提取出来后进行统一处理，例：日志、缓存和权限等

优点：方便维护、代码复用、降低核心业务和辅助业务间的耦合性



两种方式

1. JDK动态代理：只能对实现了接口的类产生代理

2. CGLIB代理：针对目标类生成一个子类并覆盖其中的方法来实现代理，底层使用了字节码框架ASM

> 目标对象实现了接口，默认情况采用JDK动态代理，也可以强制使用CGLIB代理
>
> 目标对象没有实现接口，只能使用CGLIB代理



实现方式

XML：

`<aop:aspectj-autoproxy/>`

=> AspectJAutoProxyBeanDefinitionParser#`parse()`

=> AnnotationAwareAspectJAutoProxyCreator（将其作为Bean注册到IOC容器中）



 @EnableAspectJAutoProxy：

```java
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
    // ...
}
```

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
	public void registerBeanDefinitions() {
        // 将AnnotationAwareAspectJAutoProxyCreator作为Bean注册到IOC容器中
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
    }
}
```



SpringBoot：

在Spring.factories文件中

```xml
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration
```

如果引入了AspectJ相关包，那么就会满足以下的条件装配，引入了@EnableAspectJAutoProxy

```java
@Configuration
@ConditionalOnClass({ EnableAspectJAutoProxy.class, Aspect.class, Advice.class,
		AnnotatedElement.class })
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {
    @Configuration
	@EnableAspectJAutoProxy(proxyTargetClass = false)
    @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false", matchIfMissing = false)
    public static class JdkDynamicAutoProxyConfiguration {}
    
    @Configuration
	@EnableAspectJAutoProxy(proxyTargetClass = false)
    @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true", matchIfMissing = true)
    public static class CglibAutoProxyConfiguration {}
}
```



AnnotationAwareAspectJAutoProxyCreator类

```java
public abstract class AbstractAutoProxyCreator {
    // AnnotationAwareAspectJAutoProxyCreator是BeanPostProcessor的子类，所以会执行【11】
    @Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyProxyReferences.remove(cacheKey) != bean) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
}
```

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
	// 1.获取增强器
	Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
	if (specificInterceptors != DO_NOT_PROXY) {
		this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // 2.创建代理对象
		Object proxy = createProxy(
				bean.getClass(),
				beanName, 
				specificInterceptors, 
				new SingletonTargetSource(bean));
		this.proxyTypes.put(cacheKey, proxy.getClass());
		return proxy;
	}

	this.advisedBeans.put(cacheKey, Boolean.FALSE);
	return bean;
}
```

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        // CGIB代理
        return new ObjenesisCglibAopProxy(config);
    }
    // JDK动态代理
    else {
        return new JdkDynamicAopProxy(config);
    }
}
```


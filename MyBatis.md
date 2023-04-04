### MyBatis



#### 整体架构

<img src="http://img.miilnvo.com/9t4ol.png" alt="9t4ol" style="zoom: 67%;" />



#### 核心流程

```java
// 项目部分（MyBatis-Plus顶层接口）
baseMapper.selectOne(queryWrapper);
// MyBatis-Plus部分
MybatisMapperProxy#invoke();
MybatisMapperMethod#execute();
// MyBatis-Spring部分
SqlSessionTemplate#selectOne();
（SqlSessionInterceptor#invoke();
// MyBatis部分
DefaultSqlSession#selectOne();
```

```java
public class DefaultSqlSessionFactory implements SqlSessionFactory {
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    final Environment environment = configuration.getEnvironment();
    final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
    tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
    final Executor executor = configuration.newExecutor(tx, execType);
    // 0.每次数据库操作都会创建新的DefaultSqlSession，所以一级缓存每次都会重置，当然还有相应的修改操作也会清空一级缓存
    return new DefaultSqlSession(configuration, executor, autoCommit);
  }
}
```

```java
public class DefaultSqlSession implements SqlSession {
	private final Configuration configuration;
	private final Executor executor;
  
  // statement = com.xxx.xxx.xxxMapper.get
  @Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    // 1.对应mapper.xml文件中的一个insert|update|delete|select标签
    MappedStatement ms = configuration.getMappedStatement(statement);
    // 2.执行
		return executor.query(...);
  }
}
```

```java
public abstract class BaseExecutor implements Executor {
  
  // 一级缓存，底层使用的是HashMap
  protected PerpetualCache localCache;
  
  @Override
  public <E> List<E> query(...) throws SQLException {
    // 2-1.获取SQL语句和参数,SQL中已带有=?
    BoundSql boundSql = ms.getBoundSql(parameter);
    // 2-2.生成一级缓存Key
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    // 2-3.调用自身或子类进行查询（注意：下文中有两种2-3-1）
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
  }
  
  // 包含一级缓存localCache的查询
  @Override
  public <E> List<E> query(...) throws SQLException {
    List<E> list;
    try {
      queryStack++;
      // 2-3-1.一级缓存的作用域为SqlSession
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        // 2-3-2.查询数据库
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    return list;
  }
  
}
```

```java
public final class MappedStatement {
  public BoundSql getBoundSql(Object parameterObject) {
    // 2-1-1.
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    return boundSql;
  }
}
```

```java
public class DynamicSqlSource implements SqlSource {
  
  private final Configuration configuration;
  private final SqlNode rootSqlNode;
  
  @Override
  public BoundSql getBoundSql(Object parameterObject) {
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    // 2-1-1-1.遍历构造SQL节点
    rootSqlNode.apply(context);
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    // 2-1-1-2.将SQL中的#{ew.paramNameValuePairs.MPGENVAL1}替换成?,并转换成StaticSqlSource
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    context.getBindings().forEach(boundSql::setAdditionalParameter);
    return boundSql;
  }
}
```


```java
public class CachingExecutor implements Executor {
  
  private final Executor delegate;
  // 二级缓存，底层使用的是
  private final TransactionalCacheManager tcm = new TransactionalCacheManager();
  
  // 包含二级缓存Cache的查询
  @Override
  public <E> List<E> query(...) throws SQLException {
    // 2-3-1.二级缓存的作用域为namespace,所以从MappedStatement里面获取
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          // 2-3-2.delegate即为Executor
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list);
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
}
```

```java
public class SimpleExecutor extends AbstractBaseExecutor {

  @Override
  public <E> List<E> doQuery(...) throws SQLException {
      Statement stmt = null;
      try {
        Configuration configuration = ms.getConfiguration();
        // 2-3-2-1.构造StatementHandler
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
        // 2-3-2-2.把SQL中的?替换成真正的参数
        stmt = prepareStatement(handler, ms.getStatementLog());
        // 2-3-2-3.继续查询
        return stmt == null ? Collections.emptyList() : handler.query(stmt, resultHandler);
      } finally {
        closeStatement(stmt);
      }
  }
}
```

```java
public class MybatisDefaultParameterHandler extends DefaultParameterHandler {
  @Override
    @SuppressWarnings("unchecked")
    public void setParameters(PreparedStatement ps) {
    	// 2-3-2-2-1.循环赋值
      for (int i = 0; i < parameterMappings.size(); i++) {
				MetaObject metaObject = configuration.newMetaObject(parameterObject);
				value = metaObject.getValue(propertyName);
      }
    }
}
```

```java
public class PreparedStatementHandler extends BaseStatementHandler {
  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    // 2-3-2-3-1.执行真正的SQL
    ps.execute();
    // 2-3-2-3-2.通过ResultHandler封装结果并返回
    return resultSetHandler.handleResultSets(ps);
  }
}
```



#### 接口层

* SqlSession

  <img src="http://img.miilnvo.com/lxlpk.png" alt="lxlpk" style="zoom:50%;" />

  DefaultSqlSession与SimpleExecutor的关系（策略模式）：

  <img src="http://img.miilnvo.com/jrc3i.png" alt="image-20220624094717634" style="zoom:50%;" />



#### 核心处理层

> 前四个为四大核心对象

* Executor：获取XML解析后的信息、使用缓存、构造SQL

  ![1pd0z](http://img.miilnvo.com/1pd0z.png)

  > 左侧为MyBatis-Plus重写右侧的类

  > BaseExecutor：包含了一级缓存、事务等功能（模板方法模式）
  >
  > SimpleExecutor：最常用的简单实现类
  >
  > ReuseExecutor：重用Statement对象的实现类
  >
  > BatchExecutor：提供批量处理多条SQL的实现类，调用doFlushStatements()方法执行批处理
  >
  > CachingExecutor：为Executor添加了二级缓存的功能（装饰器模式）

  > MappedStatement：对应mapper.xml文件中的一个insert|update|delete|select标签
  >
  > SqlSource：用于获取BoundSql
  >
  > BoundSql：包含构造SQL需要的所有信息
  >
  > SqlNode：一个SQL里的节点

* StatementHandler：调度ParameterHandler和ResultHandler，生成Statement（承上启下）

  ![sljl8](http://img.miilnvo.com/sljl8.png)

  > RoutingStatementHandler：根据类型创建（左侧）对应的实现类
  >
  > SimpleStatementHandler：对应JDBC的 java.sql.Statement，SQL中不能包含占位符
  >
  > PreparedStatementHandler：对应JDBC的 java.sql.PreparedStatement，SQL中有占位符
  >
  > CallableStatementHandler：对应JDBC的 java.sql.CallableStatement，用于执行存储过程

  > Statement / PreparedStatement：JDBC的接口，用于执行SQL

* ParameterHandler：SQL参数赋值

* ResultHandler：封装结果

* Interceptor：拦截器

  ```java
  public interface Interceptor {
    // 执行拦截器
    Object intercept(Invocation invocation) throws Throwable;
    // 判断是否要生成代理对象(即是否要执行拦截器)
    Object plugin(Object target);
    // 初始化此拦截器
    void setProperties(Properties properties);
  }
  ```

  ```java
  // 表示拦截Executor里的两个query()方法
  @Intercepts({
          @Signature(type = Executor.class,
                     method = "query",
                     args = {
                       MappedStatement.class,
                       Object.class,
                       RowBounds.class,
                       ResultHandler.class}),
          @Signature(type = Executor.class,
                     method = "query",
                     args = {
                       MappedStatement.class,
                       Object.class,
                       RowBounds.class,
                       ResultHandler.class,
                       CacheKey.class,
                       BoundSql.class})
  })
  ```

  1. 在四个Configuration#newXXX()方法中创建代理对象，其会执行`pluginAll()`方法

     <img src="http://img.miilnvo.com/saivb.png" alt="saivb" style="zoom:50%;" />

     ```java
     public class InterceptorChain {
       public Object pluginAll(Object target) {
         for (Interceptor interceptor : interceptors) {
           // 1-1.循环调用plugin()方法
           target = interceptor.plugin(target);
         }
         return target;
       }
     }
     ```
     ```java
     public class XXXInterceptor implements Interceptor {
       // 虽然plugin()方法的内容是自定义的,但推荐使用以下写法
       @Override
       public Object plugin(Object target) {
         if (target instanceof Executor) {
           // 1-2.在自定义拦截器中利用MyBatis自带的Plugin工具创建代理对象
           return Plugin.wrap(target, this);
         }
         return target;
       }
     }
     ```
     ```java
     public class Plugin implements InvocationHandler {
       public static Object wrap(Object target, Interceptor interceptor) {
         Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
         Class<?> type = target.getClass();
         Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
         if (interfaces.length > 0) {
            // 1-3.JDK动态代理
            return Proxy.newProxyInstance(
                type.getClassLoader(),
                interfaces,
                new Plugin(target, interceptor, signatureMap));
         }
         return target;
       }
     }
     ```

  2. 执行代理方法，即`@Signature`里的方法

     ```java
     public class Plugin implements InvocationHandler {
       
       // 原对象
       private final Object target;
       // 拦截器
       private final Interceptor interceptor;
       // 方法签名Map
       private final Map<Class<?>, Set<Method>> signatureMap;
       
       @Override
       public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
         try {
           Set<Method> methods = signatureMap.get(method.getDeclaringClass());
           // 如果当前方法是需要拦截的方法,则调用相应的拦截器方法（args参数的值见下文）
           if (methods != null && methods.contains(method)) {
             return interceptor.intercept(new Invocation(target, method, args));
           }
           return method.invoke(target, args);
         } catch (Exception e) {
           throw ExceptionUtil.unwrapThrowable(e);
         }
       }
     }
     ```
     
     ```java
     // 如何获取拦截方法的参数：
     // 比如说拦截了executor#query()方法
     executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
     // 在自定义插件中调用Invocation#args()方法返回的数组值为
     [0] = MappedStatement，即上面的ms
     [1] = 用户的参数，即wrapCollection(parameter)
     [2] = RowBounds
     [3] = (Null)
     ```
     
  
* BaseBuilder
  * XMLConfigBuilder： 解析mybatis-config.xml配置文件
  * XMLMapperBuilder：解析xxxMapper.xml映射文件

* Configuration：All-in-One配置



#### 基础支持层

* DataSource

  * PooledDataSource

    实现了一个简易的数据库连接池

    ![tiu4a](http://img.miilnvo.com/tiu4a.png)

* Cache（缓存）

  查询顺序：二级缓存 => 一级缓存 => 数据库

  * 二级缓存
  
    因为可以跨越SqlSession，所以为了避免发生脏读，引入了事务缓存管理器TransactionalCacheManager，只有在commit后才会刷新缓存区
  
    ```java
    public class TransactionalCacheManager {
    		private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<>();
    }
    
    public class TransactionalCache implements Cache {
      	// 底层依然是PerpetualCache
        private final Cache delegate;
      	private boolean clearOnCommit;
      	// 暂存区
      	private final Map<Object, Object> entriesToAddOnCommit;
      	private final Set<Object> entriesMissedInCache;
    }
    ```
  
  【参考】
  
  https://www.cnblogs.com/wuzhenzhao/p/11103043.html
  
  https://www.jianshu.com/p/81dd8e9686d6

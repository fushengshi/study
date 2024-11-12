# MyBatis

MyBatis 是 Java 生态的一款 ORM 框架。MyBatis 中一个重要的功能就是可以帮助 Java 开发封装重复性的 JDBC 代码，相较于 Hibernate 和各类 JPA 实现框架更加灵活、更加轻量级、更加可控。

> ORM 框架的核心功能：根据配置（配置文件或是注解）实现对象模型、关系模型两者之间无感知的映射。

## DataSource

JDK 提供的 javax.sql.DataSource 接口在 MyBatis 数据源中扮演了 Product 接口的角色。

### UnpooledDataSource

Java中几乎所有数据源实现的底层都是依赖 JDBC 操作数据库的，使用 JDBC 的第一步就是向 DriverManager 注册 JDBC 驱动类，之后才能创建数据库连接。

> 详见 JDK SPI 在 JDBC 中的应用。

```java
public class UnpooledDataSource implements DataSource {
    private static Map<String, Driver> registeredDrivers = new ConcurrentHashMap<>();
    static {
        // 从DriverManager中读取JDBC驱动
        Enumeration<Driver> drivers = DriverManager.getDrivers();
        while (drivers.hasMoreElements()) {
            Driver driver = drivers.nextElement();
            // 复制
            registeredDrivers.put(driver.getClass().getName(), driver);
        }
    }
    ...
}
```

MyBatis 的 UnpooledDataSource 实现中定义了静态代码块，在 UnpooledDataSource 加载时，将已在 DriverManager 中注册的 JDBC 驱动器实例复制一份到 UnpooledDataSource.registeredDrivers 集合中。

### PooledDataSource

JDBC 连接的创建是非常耗时的，数据库能够建立的连接数也是有限的，需要使用数据库连接池来缓存、复用数据库连接。

```java
public class PooledDataSource implements DataSource {
    // 管理连接
    private final PoolState state = new PoolState(this);
    private final UnpooledDataSource dataSource;
	...
}
```

PooledConnection 是 MyBatis 中定义的一个 InvocationHandler 接口实现类，其中封装了真正的 java.sql.Connection 对象以及相关的代理对象。proxyConnection 这个 Connection 代理对象的初始化，使用的是 **JDK 动态代理**的方式实现的，其中传入的 InvocationHandler 实现是 PooledConnection 自身。

```java
class PooledConnection implements InvocationHandler {
    private final PooledDataSource dataSource;
    // 当前 PooledConnection 底层的真正数据库连接对象。
    private final Connection realConnection;
    // 指向了 realConnection 数据库连接的代理对象。
    private final Connection proxyConnection;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        // 如果调用close()方法，并没有直接关闭底层连接，而是将其归还给关联的连接池
        if (CLOSE.equals(methodName)) {
            dataSource.pushConnection(this);
            return null;
        }
        try {
            if (!Object.class.equals(method.getDeclaringClass())) {
                checkConnection();
            }
            return method.invoke(realConnection, args);
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
    }
}
```



## Mapper 

MapperProxy 是生成 Mapper 接口**代理对象**的关键，它实现了 InvocationHandler 接口。

只需要编写接口，然后调用接口方法就可以实现对数据库的操作，底层的实现是通过 MapperProxy 对象，通过层层封装调用，最后都会映射到 SqlSession 的调用。

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
    ...
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            if (Object.class.equals(method.getDeclaringClass())) {
                return method.invoke(this, args);
            } else {
                // 拦截所有非 Object 方法
                return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
            }
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
    }
}

private MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
    MapperMethodInvoker invoker = methodCache.get(method);
    if (invoker != null) {
        return invoker;
    }

    return methodCache.computeIfAbsent(method, m -> {
        if (m.isDefault()) { // default方法
            try {
                if (privateLookupInMethod == null) {
                    // 通过底层维护的 MethodHandle 完成方法调用
                    return new DefaultMethodInvoker(getMethodHandleJava8(method));
                } else {
                    return new DefaultMethodInvoker(getMethodHandleJava9(method));
                }
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        } else {
            // 通过底层维护的 MapperMethod 完成方法调用
            return new PlainMethodInvoker(new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
        }
    });
}
```

DefaultMethodInvoker 和 PlainMethodInvoker 都是 MapperMethodInvoker 接口的实现。

- 在 DefaultMethodInvoker.invoke() 方法中，会通过底层维护的 MethodHandle 完成方法调用。
- 在 PlainMethodInvoker.invoke() 方法中，会通过底层维护的 MapperMethod 完成方法调用。

### MapperMethod

MapperMethod 是最终执行 SQL 语句的地方，同时也记录了 Mapper 接口中的对应方法。

- SqlCommand 维护了关联 SQL 语句的相关信息。

  SqlCommand 的构造方法中， `resolveMappedStatement()` 方法会尝试根据 SQL 语句的唯一标识从 Configuration 全局配置对象中查找关联的 MappedStatement 对象，还会尝试顺着 Mapper 接口的继承树进行查找，直至查找成功为止。

  > MappedStatement对象就是 **Mapper.xml** 配置文件中一条SQL语句解析之后得到的对象。

- MapperMethod 维护了 Mapper 接口中方法的相关信息。

`execute()` 方法是 MapperMethod 中最核心的方法之一。`execute()` 方法会根据要执行的 SQL 语句的具体类型执行 SqlSession 的相应方法完成数据库操作。



# 附录

## Reflector 

Reflector 是 MyBatis 反射模块的基础。要使用反射模块操作一个 Class，都会先将该 Class 封装成一个 Reflector 对象，在 Reflector 中缓存 Class 的元数据信息，对获取方法对象进行了优化。

```java
public class Reflector {
    private final Map<String, Invoker> setMethods = new HashMap<>();
    private final Map<String, Invoker> getMethods = new HashMap<>();
    ...
}
    
public class MethodInvoker implements Invoker {
    private final Class<?> type;
    private final Method method;
    ...

    @Override
    public Object invoke(Object target, Object[] args) throws IllegalAccessException, InvocationTargetException {
        try {
            return method.invoke(target, args);
        } catch (IllegalAccessException e) {
            if (Reflector.canControlMemberAccessible()) {
                method.setAccessible(true);
                return method.invoke(target, args);
            } else {
                throw e;
            }
        }
    }
}
```

Reflector 构造时直接把反射的类给拆解存入对应的成员变量中。`setMethods()` 和 `getMethods()` 是反射优化的核心，将属性封装成了调用类进行缓存，后续遇到同样的属性名称，不用重新通过反射获取方法对象，直接用调用类来发起调用，减少了反射获取方法的损耗。



## LogFactory 

在 LogFactory 类中有一段静态代码块，其中会依次加载各个第三方日志框架的适配器。

```java
static {
    tryImplementation(LogFactory::useSlf4jLogging);
    tryImplementation(LogFactory::useCommonsLogging);
    tryImplementation(LogFactory::useLog4J2Logging);
    tryImplementation(LogFactory::useLog4JLogging);
    tryImplementation(LogFactory::useJdkLogging);
    tryImplementation(LogFactory::useNoLogging);
}
```

首先会检测 logConstructor 字段是否为空，如果不为空，则表示已经成功确定当前使用的日志框架，直接返回。如果为空，则在当前线程中执行传入的 `Runnable.run()` 方法，尝试确定当前使用的日志框架。



## java MethodHandle 

从 Java 7 开始，除了反射之外，在 java.lang.invoke 包中新增了 MethodHandle 这个类，它的基本功能与反射中的 Method 类似，但它比反射更加灵活。反射是 Java API 层面支持的一种机制，MethodHandle 则是 JVM 层支持的机制，相较而言反射更加重量级，MethodHandle 则更轻量级，性能比反射更好。

```java
public static class Demo {
    public String say(String s) {
        return s;
    }
}

public static void main(String[] args) throws Throwable {
    // 定义sayHello()方法的签名，第一个参数是方法的返回值类型，第二个参数是方法的参数列表
    MethodType methodType = MethodType.methodType(String.class, String.class);

    // 根据方法名和MethodType在MethodHandleDemo中查找对应的MethodHandle
    MethodHandle methodHandle = MethodHandles.lookup().findVirtual(Demo.class, "say", methodType);

    // 调用Demo对象的方法
    Demo demo = new Demo();
    Object result = methodHandle.bindTo(demo).invokeWithArguments("hello");
    System.out.println(result);
}
```

在 `DefaultMethodInvoker.invoke()` 方法中，会通过底层维护的 MethodHandle 完成方法调用，核心实现如下：

```java
public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
    // 首先将MethodHandle绑定到一个实例对象上，然后调用invokeWithArguments()方法执行目标方法
    return methodHandle.bindTo(proxy).invokeWithArguments(args);
}
```



## Spring 整合

Spring在 Mapper 扫描的时候，将所有Mapper bean定义中的 beanClass 设置成了 MapperFactoryBean（继承 FactoryBean），所以通过 `createBean()` 方法创建的 Mapper 实例实际上是 MapperFactoryBean 对象。

```java
public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
    this.checkDaoConfig();

    try {
        this.initDao();
    } catch (Exception var2) {
        throw new BeanInitializationException("Initialization of DAO failed", var2);
    }
}
```

MapperFactoryBean 实现了 InitializingBean 接口，Spring容器会在所有必需的属性被填充后（也就是依赖注入完成后）调用 `afterPropertiesSet()`，`afterPropertiesSet()` 方法是在 父类 DaoSupport 中实现，进行对 Dao 配置的验证。

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
    @Override
    public T getObject() throws Exception {
        // 通过MapperProxy生成代理对象
        return getSqlSession().getMapper(this.mapperInterface);
    }
    ...
}
```

MapperFactoryBean 实现了 FactoryBean 接口，所以通过 `getBean()` 方法获取实例是该类 `getObject()` 函数返回的实例。

> 自定义 FactoryBean 原理详见 Spring 源码。






























































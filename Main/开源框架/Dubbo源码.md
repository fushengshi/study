# Dubbo

- Registry：注册中心

  负责服务地址的注册与查找，服务的 Provider 和 Consumer 只在启动时与注册中心交互。注册中心通过长连接感知 Provider 的存在，在 Provider 出现宕机的时候，注册中心会立即推送相关事件通知 Consumer。

- Provider：服务提供者

  在它启动的时候，会向 Registry 进行注册操作，将自己服务的地址和相关配置信息封装成 URL 添加到 ZooKeeper 中。

- Consumer：服务消费者

  在它启动的时候，会向 Registry 进行订阅操作。订阅操作会从 ZooKeeper 中获取 Provider 注册的 URL，并在 ZooKeeper 中添加相应的监听器。获取到 Provider URL 之后，Consumer 会根据负载均衡算法从多个 Provider 中选择一个 Provider 并与其建立连接，最后发起对 Provider 的 RPC 调用。 如果 Provider URL 发生变更，Consumer 将会通过之前订阅过程中在注册中心添加的监听器，获取到最新的 Provider URL 信息，进行相应的调整，比如断开与宕机 Provider 的连接，并与新的 Provider 建立连接。Consumer 与 Provider 建立的是长连接，且 Consumer 会缓存 Provider 信息，所以一旦连接建立，即使注册中心宕机，也不会影响已运行的 Provider 和 Consumer。

- Monitor：监控中心

  用于统计服务的调用次数和调用时间。Provider 和 Consumer 在运行过程中，会在内存中统计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。监控中心在上面的架构图中并不是必要角色，监控中心宕机不会影响 Provider、Consumer 以及 Registry 的功能，只会丢失监控数据而已。

![image-20240729205105560](https://gitee.com/fushengshi/image/raw/master/image-20240729205105560.png)

![image-20240729205147853](https://gitee.com/fushengshi/image/raw/master/image-20240729205147853.png)



## 配置总线

URL 是整个 Dubbo 中基础的核心组件，称为 Dubbo 的配置总线。一个 URL 可以包含非常多的扩展点参数，URL 作为上下文信息贯穿整个扩展点设计体系。

URL 采用标准格式：`protocol://username:password@host:port/path?key=value&key=value`

- 使用 URL 公共契约进行上下文信息传递，形成一个统一的规范，使得代码易写、易读。
- 使用 URL 作为方法的入参（相当于一个 Key/Value 都是 String 的 Map)，它所表达的含义比单个参数更丰富，当代码需要扩展的时候，可以将新的参数以 Key/Value 的形式追加到 URL 之中，而不需要改变入参或是返回值的结构。



## Dubbo SPI

JDK SPI 在查找扩展实现类的过程中，遍历 SPI 配置文件中定义的所有实现类，该过程中会将这些实现类全部实例化。Dubbo SPI 解决了上述资源浪费的问题，并对 SPI 配置文件扩展和修改。

- META-INF/services/ 目录：该目录下的 SPI 配置文件用来兼容 JDK SPI 。
- META-INF/dubbo/ 目录：该目录用于存放用户自定义 SPI 配置文件。
- META-INF/dubbo/internal/ 目录：该目录用于存放 Dubbo 内部使用的 SPI 配置文件。

注解：

- @SPI

  Dubbo 中某个接口被 @SPI 注解修饰时，就表示该接口是扩展接口。

  - 扩展点：通过 SPI 机制查找并加载实现的接口（又称 扩展接口）。例如 com.mysql.cj.jdbc.Driver 接口是扩展点。
  - 扩展点实现：实现了扩展接口的实现类。

  @SPI 注解的 value 值指定了默认的扩展名称，如果没有明确指定扩展名，则默认会将 @SPI 注解的 value 值作为扩展名。

  ```java
  Protocol protocol = ExtensionLoader 
     .getExtensionLoader(Protocol.class).getExtension("dubbo");
  ```

- @Adaptive

  - @Adaptive 注解标注在**接口**

    @Adaptive 注解标注的方法中，其参数中必须有一个参数类型为 URL，或者其某个参数提供了某个方法，该方法可以返回一个URL 对象。

    Dubbo 会为拓展接口生成具有代理功能的代码，通过 Javassist（JavassistCompiler） 或 jdk 编译这段代码得到代理 Class 类，最后通过反射创建代理实例。

    例如 Transporter 接口中方法上的 @Adaptive 注解生成适配器类：

    ```java
    public class Transporter$Adaptive implements Transporter { 
        public org.apache.dubbo.remoting.Client connect(URL arg0, ChannelHandler arg1) throws RemotingException { 
            // 必须传递URL参数 
            if (arg0 == null) throw new IllegalArgumentException("url == null"); 
            URL url = arg0; 
    
            // 确定扩展名，优先从URL中的client参数获取，其次是transporter参数 
            // 这两个参数名称由@Adaptive注解指定，最后是@SPI注解中的默认值 
            String extName = url.getParameter("client", url.getParameter("transporter", "netty")); 
    
            if (extName == null) 
                throw new IllegalStateException("..."); 
    
            // 通过ExtensionLoader加载Transporter接口的指定扩展实现 
            Transporter extension = (Transporter) ExtensionLoader 
                .getExtensionLoader(Transporter.class) 
                .getExtension(extName); 
    
            return extension.connect(arg0, arg1); 
        } 
    }  
    ```

  - @Adaptive 注解标注在**类**

    被标注的扩展类将直接作为默认实现类来调用方法。在 Dubbo 框架中类级别的仅有 AdaptiveExtensionFactory  和 AdaptiveCompile 两个类。

    > AdaptiveExtensionFactory 不实现任何具体的功能，而是用来适配 ExtensionFactory 的 SpiExtensionFactory 和 SpringExtensionFactory 这两种实现。

- @Activate

  @Activate 注解标注在扩展实现类上，有 group、value 以及 order 三个属性。

  - group 属性：修饰的实现类是在 Provider 端被激活还是在 Consumer 端被激活。
  - value 属性：修饰的实现类只在 URL 参数中出现指定的 key 时才会被激活。
  - order 属性：用来确定扩展实现类的排序。

  > org.apache.dubbo.rpc.Filter 接口有非常多的扩展实现类，在一个场景中可能需要某几个 Filter 扩展实现类协同工作，而另一个场景中可能需要另外几个实现类一起工作，需要一套配置来指定当前场景中哪些 Filter 实现是可用的。

- 自动装配

  Dubbo SPI 在拿到扩展实现类的对象（以及 Wrapper 类的对象）之后，会调用 `injectExtension()` 方法扫描其全部 setter 方法，并根据 setter 方法的名称以及参数的类型，加载相应的扩展实现，然后调用相应的 setter 方法填充属性，这就实现了 Dubbo SPI 的自动装配特性。

  自动装配属性就是在加载一个扩展点的时候，将其依赖的扩展点一并加载，并进行装配。

  例如：

  ```java
  public class RegistryProtocol implements Protocol {
      private Protocol protocol;
      
      public void setProtocol(Protocol protocol) {
          this.protocol = protocol;
      }
  }
  ```



### createExtension()

`createExtension()` 方法中完成了 SPI 配置文件的查找以及相应扩展实现类的实例化，同时还实现了自动装配以及自动 Wrapper 包装等功能。

```java
private T createExtension(String name) {
    // 根据扩展名从 cachedClasses 缓存中获取扩展实现类，如果 cachedClasses 未初始化，则会扫描三个SPI目录获取查找相应的SPI配置文件，然后加载其中的扩展实现类，最后将扩展名和扩展实现类的映射关系记录到 cachedClasses 缓存中。
    // 根据扩展类的类型给分别给cachedClasses、cachedAdaptiveClass、cachedWrapperClasses 和 cachedNames赋值。
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        // 根据扩展实现类从 EXTENSION_INSTANCES 缓存中查找相应的实例。
        // 如果查找失败，会通过反射创建扩展实现对象。
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            // 保存到缓存
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }

        // 自动装配
        injectExtension(instance);

        // 自动包装（Wrapper）扩展实现对象。
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            // 如果有包装类则返回包装类。
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        // 扩展实现类实现了Lifecycle接口，调用 initialize() 方法初始化。
        initExtension(instance);
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}
```



### injectExtension()

`injectExtension()` 方法是 Dubbo SPI 的 IOC 实现，通过 setter 方法注入依赖。

```java
private T injectExtension(T instance) {
    ...

    try {
        for (Method method : instance.getClass().getMethods()) {
            ...

            try {
                // 根据setter方法的名称确定属性名称
                String property = getSetterProperty(method);

                // 加载并实例化扩展实现类
                Object object = objectFactory.getExtension(pt, property);
                if (object != null) {
                    // 通过反射调用setter方法设置依赖。
                    method.invoke(instance, object);
                }
            } catch (Exception e) {
                logger.error("Failed to inject via method " + method.getName()
                        + " of interface " + type.getName() + ": " + e.getMessage(), e);
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

`injectExtension()` 方法实现的自动装配依赖了 ExtensionFactory（objectFactory 字段），ExtensionFactory 有 SpringExtensionFactory 和 SpiExtensionFactory 两个真正的实现。

SpiExtensionFactory 代码逻辑：

```java
private T createAdaptiveExtension() {
    try {
        // IOC注入时候，setter值也是反射创建并且也是调用 injectExtension 注入属性
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}

 private Class<?> getAdaptiveExtensionClass() {
     // 调用 getExtensionClasses() 方法，触发 loadClass() 方法，完成 cachedAdaptiveClass 字段的填充。
     getExtensionClasses();

     // 如果存在 @Adaptive 注解修饰的扩展实现类，该类就是适配器类，通过 newInstance() 实例化。
     if (cachedAdaptiveClass != null) {
         return cachedAdaptiveClass;
     }
     // 如果不存在 @Adaptive 注解修饰的扩展实现类，
     // 通过 createAdaptiveExtensionClass()方法扫描扩展接口中方法上的 @Adaptive 注解，动态生成适配器类，然后实例化
     return cachedAdaptiveClass = createAdaptiveExtensionClass();
 }
```

Dubbo SPI 实现的 IOC 没有处理循环依赖的情况，如果出现了循环依赖会死循环。



## Service层



## config层



## Proxy层

Dubbo 使用动态代理机制来屏蔽底层的网络传输以及服务发现。

Consumer 进行调用的时候，Dubbo 会通过**动态代理**将业务接口实现对象转化为相应的 Invoker 对象，然后 Cluster 层、Protocol 层都会使用 Invoker。在 Provider 暴露服务的时候，也会有 Invoker 对象与业务接口实现对象之间的转换，同样通过动态代理实现。

> 虽然在架构图中 Proxy 层与 Protocol 层距离很远，但 Proxy 的具体代码实现就位于 dubbo-rpc-api 模块中。

### ProxyFactory

```java
@SPI("javassist")
public interface ProxyFactory {
    // 为传入的Invoker对象创建代理对象
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;

    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException;

    // 将传入的代理对象封装成Invoker对象
    @Adaptive({PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;
}
```

定义了两个核心方法：

- `getProxy()` 方法，将 Invoker 对象创建代理对象。
- `getInvoker()` 方法，将代理对象反向封装成 Invoker 对象。

#### getProxy()

用于 Consumer 端 RPC 服务接口生成动态代理类。

> 在 `createProxy()` 方法中，通过 **Protocol 的 `refer()` 方法** 根据接口类型（Class<T> type）返回一个 Invoker 对象。这个 Invoker 对象作为 `getProxy()` 的参数，Consumer 端可以通过这个 Invoker 请求到 Provider 端的服务。

JavassistProxyFactory 对 `getProxy()` 方法直接委托给 dubbo-common 模块中的 Proxy 工具类进行代理类的生成。

- 查找 PROXY_CACHE_MAP 代理类缓存（WeakHashMap类型），其中第一层 Key 是 ClassLoader 对象，第二层 Key 是上面整理得到的接口拼接而成的，Value 是被缓存的代理类的 WeakReference（弱引用）。

  > WeakReference（弱引用）的特性是 WeakReference 引用的对象生命周期是两次 GC 之间，当垃圾收集器扫描到只具有弱引用的对象时，无论当前内存空间是否足够，都会回收该对象（由于垃圾收集器是一个优先级很低的线程，不一定会很快发现那些只具有弱引用的对象）。
  >
  > WeakReference 的特性决定了它特别适合用于数据可恢复的内存型缓存。

- 从 PROXY_CLASS_COUNTER 字段（AtomicLong 类型）中获取一个 id 值，作为代理类的后缀，这主要是为了避免类名重复发生冲突。

- 遍历全部接口，获取每个接口中定义的方法，对每个方法进行处理。

- 开始创建代理实例类（ProxyInstance）和代理类。

  - 创建代理实例类

    向 ClassGenerator 中添加相应的信息，例如，类名、默认构造方法、字段、父类以及一个 `newInstance()` 方法。

    生成的方法（javassist）示例如下：

    ```java
    public java.lang.String sayHello(java.lang.String arg0){
      Object[] args = new Object[1]; 
      args[0] = ($w)$1; 
    
      // 通过InvocationHandler.invoke()方法调用目标方法
      // InvokerInvocationHandler中维护了一个Invoker对象，是getProxy()方法传入的第一个参数，通过Protocol的refer()方法根据接口类型生成
      Object ret = handler.invoke(this, methods[3], args); 
      return (java.lang.String) ret;
    }
    ```

  - 创建代理类

    实现了 Proxy 接口，并实现了 `newInstance()` 方法，该方法会直接返回上面代理实例类的对象。



#### getInvoker()

Provider 的业务层实现会被包装成一个 ProxyInvoker，然后这个 ProxyInvoker 还会被 Filter、Listener 以及其他装饰器包装。ProxyFactory 的 `getInvoker()` 方法是将业务接口实现封装成 ProxyInvoker 入口。



## Registry层

Registry是应用本地的注册中心客户端，本地的 Registry 通过和 ZooKeeper 等进行实时的信息同步，维持内容的一致性，从而实现了注册中心这个特性。

> 真正的注册中心服务是其他独立部署的进程，或进程组成的集群，比如 ZooKeeper 集群。

Dubbo 在微服务架构中解决了各个服务间协作的难题，作为 Provider 和 Consumer 的底层依赖，它会与服务一起打包部署。dubbo-registry 也仅仅是其中一个依赖包，负责完成与 ZooKeeper、etcd、Consul 等服务发现组件的交互。

### AbstractRegistry

- 本地缓存

  当 Provider 端暴露的 URL 发生变化时，ZooKeeper 等服务发现组件会通知 Consumer 端的 Registry 组件，Registry 组件会调用 `notify()` 方法，被通知的 Consumer 能匹配到所有 Provider 的 URL 列表并写入 properties 集合中。

  AbstractRegistry 的核心是本地文件缓存的功能。在 AbstractRegistry 的构造方法中，会调用 `loadProperties()` 方法写入的本地缓存文件加载到 properties 对象中。在网络抖动等原因而导致订阅失败时，Consumer 端的 Registry 就可以调用 `getCacheUrls()` 方法获取本地缓存，从而得到最近注册的 Provider URL，**AbstractRegistry 通过本地缓存提供了容错机制，保证了服务的可靠性**。

- 注册/订阅

  AbstractRegistry 实现了 Registry 接口，它实现的 `registry()` 方法会将当前节点要注册的 URL 缓存到 registered 集合，而 `unregistry()` 方法会从 registered 集合删除指定的 URL，例如当前节点下线的时候。

  `subscribe()` 方法会将当前节点作为 Consumer 的 URL 以及相关的 NotifyListener 记录到 subscribed 集合，`unsubscribe()` 方法会将当前节点的 URL 以及关联的 NotifyListener 从 subscribed 集合删除。

  > AbstractRegistry 的实现基础的注册、订阅方法都是内存操作，AbstractRegistry 的子类会覆盖上述方法进行增强。

### FailbackRegistry

dubbo-registry 将**重试机制**的相关实现放到了 AbstractRegistry 的子类 FailbackRegistry 中。

FailbackRegistry 设计核心是：覆盖了 AbstractRegistry 中 `register()/unregister()`、`subscribe()/unsubscribe()` 以及 `notify()` 这五个核心方法，结合**时间轮**实现失败重试的能力。真正与服务发现组件的交互能力则是放到了 `doRegister()/doUnregister()`、`doSubscribe()/doUnsubscribe()` 以及 `doNotify()` 这五个抽象方法中，由具体子类实现。这是典型的模板方法模式的应用。

### ZookeeperRegistryFactory（服务发现）

Dubbo 可以接入多种服务发现组件，例如，ZooKeeper、etcd、Consul、Eureka 等。其中 Dubbo 特别推荐使用 ZooKeeper。

在 dubbo-registry-zookeeper 模块中的 SPI 配置文件中，指定了 RegistryFactory 的实现类  ZookeeperRegistryFactory。ZookeeperRegistryFactory 实现了 AbstractRegistryFactory，其中的 `createRegistry()` 方法会创建 ZookeeperRegistry 实例，后续将由该 ZookeeperRegistry 实例完成与 Zookeeper 的交互。ZookeeperRegistryFactory 中还提供了一个 `setZookeeperTransporter()` 方法，会通过 SPI 完成自动装载。

dubbo-remoting-zookeeper 模块是在 Apache Curator 的基础上封装了一套 Zookeeper 客户端，将与 Zookeeper 的交互融合到 Dubbo 的体系之中。两个核心接口：ZookeeperTransporter 接口和 ZookeeperClient 接口。

> dubbo-remoting-zookeeper 模块是 dubbo-remoting 模块的子模块，但它并不依赖 dubbo-remoting 中的其他模块，是相对独立的。



## Cluster层

Dubbo 独立出了一个实现集群功能的模块 dubbo-cluster，主要功能是将多个 Provider 伪装成一个 Provider 供 Consumer 调用，其中涉及集群的容错处理、路由规则的处理以及负载均衡。

当调用进入 Cluster：

- Cluster 会创建一个 AbstractClusterInvoker 对象
- 在 AbstractClusterInvoker 中，首先会从 Directory 中获取当前 Invoker 集合。
- 按照 Router 集合进行路由，得到符合条件的 Invoker 集合。
- 按照 LoadBalance 指定的负载均衡策略得到最终要调用的 Invoker 对象。

### Directory

Directory 接口表示的是一个集合，该集合由多个 Invoker 构成，后续的路由处理、负载均衡、集群容错等一系列操作都是在 Directory 基础上实现的。

Directory 接口有 RegistryDirectory 和 StaticDirectory 两个具体实现。

- RegistryDirectory 实现中维护的 Invoker 集合会随着注册中心中维护的注册信息动态发生变化，这就依赖了 ZooKeeper 等注册中心的推送能力。
- StaticDirectory 实现中维护的 Invoker 集合则是静态的，在 StaticDirectory 对象创建完成之后，不会再发生变化。

### Router（路由）

Router 的主要功能就是根据用户配置的路由规则以及请求携带的信息，过滤出符合条件的 Invoker 集合，供后续负载均衡逻辑使用。

在 RouterChain 的构造函数中，会在传入的 URL 参数中查找 router 参数值，并根据该值获取确定激活的 RouterFactory，之后通过 Dubbo SPI 机制加载激活的 RouterFactory 对象，由 RouterFactory 创建当前激活的内置 Router 实例。

```java
private RouterChain(URL url) {
    // 通过ExtensionLoader加载激活的RouterFactory。
    List<RouterFactory> extensionFactories = ExtensionLoader.getExtensionLoader(RouterFactory.class)
            .getActivateExtension(url, "router");
    // 遍历所有RouterFactory，调用其getRouter()方法创建相应的Router对象。
    List<Router> routers = extensionFactories.stream()
            .map(factory -> factory.getRouter(url))
            .collect(Collectors.toList());
    // 初始化builtinRouters字段以及routers字段。
    initWithRouters(routers);
}
```

Router 决定了一次 Dubbo 调用的目标服务，Router 接口的每个实现类代表了一个路由规则。

- ConditionRouterFactory

  ConditionRouter 是基于条件表达式的路由实现类。

  例如 `host = 192.168.0.100 => host = 192.168.0.150`

- ScriptRouterFactory

  ScriptRouter 对脚本路由，支持 JDK 脚本引擎的所有脚本，例如JavaScript、JRuby、Groovy 等，通过 `type=javascript` 参数设置脚本类型，缺省为 javascript。

- FileRouterFactory

  FileRouterFactory 是 ScriptRouterFactory 的装饰器，其扩展名为 file，FileRouterFactory 在 ScriptRouterFactory 基础上增加了读取文件的能力。

- TagRouterFactory

  通过 TagRouter，可以将某一个或多个 Provider 划分到同一分组，约束流量只在指定分组中流转，可以达到流量隔离的目的，从而支持灰度发布等场景。

- ServiceRouterFactory

  ServiceRouter继承了 ListenableRouter 抽象类，ListenableRouter 在 ConditionRouter 基础上添加了动态配置的能力。

### LoadBalance（负载均衡）

LoadBalance 的职责是将网络请求或者其他形式的负载“均摊”到不同的服务节点上，从而避免服务集群中部分节点压力过大、资源紧张，而另一部分节点比较空闲的情况。

Dubbo 对 Consumer 的调用请求进行分配，避免少数 Provider 节点负载过大，其他 Provider 节点处于空闲的状态。因为当 Provider 负载过大时，就会导致一部分请求超时、丢失等一系列问题发生，造成线上故障。

> 客户端负载均衡。

- ConsistentHashLoadBalance

  使用**一致性 Hash 算法**实现负载均衡。

  `doSelect()` 方法的实现，其中会根据 ServiceKey 和 methodName 选择一个 ConsistentHashSelector 对象，核心算法都委托给 ConsistentHashSelector 对象完成。

- RandomLoadBalance

  使用加权随机负载均衡算法。

- LeastActiveLoadBalance

   使用最小活跃数负载均衡算法。

- RoundRobinLoadBalance

  使用加权轮询负载均衡算法。

- ShortestResponseLoadBalance

  使用最短响应时间的负载均衡算法。

  > Dubbo 2.7 版本之后新增加的 LoadBalance 实现类。

### Cluster（集群容错）

Dubbo 中常见的容错方式如下：

- FailoverCluster（默认）

  失败自动切换。

  在请求一个 Provider 节点失败的时候，自动切换其他 Provider 节点，默认执行 3 次，适合幂等操作。

  重试次数越多，在故障容错的时候带给 Provider 的压力就越大，在极端情况下甚至可能造成雪崩式的问题。

- FailbackCluster

  失败自动恢复。失败后记录到队列中，通过定时器重试。

- FailfastCluster

  快速失败。请求失败后返回异常，不进行任何重试。

- FailsafeCluster

  失败安全。请求失败后忽略异常，不进行任何重试。

- ForkingCluster

  并行调用多个 Provider 节点，只要有一个成功就返回。

- BroadcastCluster

  广播多个 Provider 节点，只要有一个节点失败就失败。

- AvailableCluster

  遍历所有的 Provider 节点，找到每一个可用的节点，就直接调用，如果没有可用的 Provider 节点，则直接抛出异常。

- MergeableCluster

  请求多个 Provider 节点并将得到的结果进行合并。

在每个 Cluster 接口实现中，都会创建对应的 Invoker 对象，这些都继承自 AbstractClusterInvoker 抽象类。AbstractCluster 抽象类的核心逻辑是在 ClusterInvoker 外层包装一层 ClusterInterceptor，从而实现类似切面的效果。

> 责任链模式。
>
> Interceptor调用链 `buildClusterInterceptors()`，逻辑同 `buildInvokerChain()`。
>
> InterceptorInvokerNode(InterceptorInvokerNode(InterceptorInvokerNode(FailoverCluster)))

```java
// clusterInvoker为传入的容错对象，如FailoverCluster对象。
private <T> Invoker<T> buildClusterInterceptors(AbstractClusterInvoker<T> clusterInvoker, String key) {
    AbstractClusterInvoker<T> last = clusterInvoker;
    // 通过SPI方式加载ClusterInterceptor扩展实现。
    List<ClusterInterceptor> interceptors = ExtensionLoader.getExtensionLoader(ClusterInterceptor.class).getActivateExtension(clusterInvoker.getUrl(), key);

    if (!interceptors.isEmpty()) {
        for (int i = interceptors.size() - 1; i >= 0; i--) {
            // 形成调用链。
            final ClusterInterceptor interceptor = interceptors.get(i);
            
            final AbstractClusterInvoker<T> next = last;
            last = new InterceptorInvokerNode<>(clusterInvoker, interceptor, next);
        }
    }
    return last;
}
```



## Monitor层



## Protocol层

Protocol 层是 Remoting 层的使用者，会通过 Exchangers 门面类创建 ExchangeClient 以及 ExchangeServer，还会创建相应的 ChannelHandler 实现以及 Codec2 实现并交给 Exchange 层进行装饰。

dubbo-rpc-api 是对具体协议、服务暴露、服务引用、代理等的抽象，是整个 Protocol 层的核心。模块例如 dubbo-rpc-dubbo、dubbo-rpc-grpc、dubbo-rpc-http 等，都是 Dubbo 支持的具体协议。

### Invoker

Invoker 是 Dubbo 的核心模型之一，它代表一个可执行体，可以向它发起 invoke 调用。在 Dubbo 中，万物皆是 Invoker，即便是Exporter 也是由 Invoker 进化而成的。

```java
public interface Invoker<T> extends Node {
    // 服务接口
    Class<T> getInterface();

    // 一次调用
    Result invoke(Invocation invocation) throws RpcException;
}
```

在 AbstractInvoker 中实现了 Invoker 接口中的 `invoke()` 方法（模板方法模式），对 URL 中的配置信息以及 RpcContext 中携带的附加信息进行处理，添加到 Invocation 中作为附加信息，然后调用 `doInvoke()` 方法（子类具体实现）发起远程调用，最后得到 AsyncRpcResult 对象返回。

### DubboInvoker

DubboInvoker 是 AbstractInvoker 的实现类，在 `doInvoke()` 方法中首先会选择此次调用使用 ExchangeClient 对象，然后确定此次调用是否需要返回值，最后调用 `ExchangeClient.request()` 方法发送请求，对返回的 Future 进行简单封装并返回。

### ProtocolFilterWrapper.buildInvokerChain()

Filter 相关的 Invoker 装饰器，将各个 Filter 串联成 Filter 链并与 Invoker 实例相关。

构造 Filter 链的核心逻辑位于 `ProtocolFilterWrapper.buildInvokerChain()` 方法中，ProtocolFilterWrapper 的 `refer()` 方法和 `export()` 方法都会调用该方法。`buildInvokerChain()` 方法的核心逻辑如下：

- 首先会根据 URL 中携带的配置信息，确定当前激活的 Filter 扩展实现有哪些，形成 Filter 集合。
- 遍历 Filter 集合，将每个 Filter 实现封装成一个匿名 Invoker，在这个匿名 Invoker 中，会调用 Filter 的 `invoke()` 方法执行 Filter 的逻辑，然后由 Filter 内部的逻辑决定是否将调用传递到下一个 Filter 执行。

> 责任链模式。
>
> 倒序遍历 Filter 集合，将 `filter.invoke(next, invocation)` 逻辑封装为 Invoker，其中 next 为后一个 Filter 封装的 Invoker，由 Filter 内部的逻辑决定是否将调用传递到 next 执行。
>
> 整个调用链可表示为 Filter(Filter(Filter(Filter(Invoker0)))

```java
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
    Invoker<T> last = invoker;
    // 根据URL中携带的配置信息，确定激活的Filter扩展形成Filter集合
    List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);

    if (!filters.isEmpty()) {
        // 遍历Filter集合，将每个Filter实现封装成一个匿名Invoker
        for (int i = filters.size() - 1; i >= 0; i--) {
            final Filter filter = filters.get(i);

            final Invoker<T> next = last;
            last = new Invoker<T>() {
                @Override
                public Result invoke(Invocation invocation) throws RpcException {
                    Result asyncResult;
                    try {
                        // 调用Filter的 invoke() 方法执行 Filter 的逻辑，
                        // 由Filter内部的逻辑决定是否将调用传递到下一个Filter(next)执行。
                        asyncResult = filter.invoke(next, invocation);
                    } catch (Exception e) {
                        ...
                    } finally {
                    }
               ...
            };
        }
    }
    return last;
}
```

### Filter

Filter 是扩展 Dubbo 功能的首选方案，Dubbo 自身也提供了非常多的 Filter 实现来扩展自身功能。

Filter 链的组装逻辑设计得非常灵活，其中可以通过 `-` 配置手动剔除 Dubbo 原生提供的、默认加载的 Filter，通过 default 来代替 Dubbo 原生提供的 Filter。

- ActiveLimitFilter

  ActiveLimitFilter 是 Consumer 端用于限制一个 Consumer 对于一个服务端方法的并发调用量，也可以称为 客户端限流 。

- ExecuteLimitFilter

  ExecuteLimitFilter 是 Dubbo 在 Provider 端限流的实现。

- TpsLimitFilter

  TpsLimitFilter 是 Provider 端对 TPS 限流的实现。

- AccessLogFilter

  AccessLogFilter 主要用于记录日志，它的主要功能是将 Provider 或者 Consumer 的日志信息写入文件中。

### DubboProtocol

Dubbo 默认使用 DubboProtocol。

#### 服务注册

- 服务端初始化

  `openServer()` 方法会调用 Exchange 层、Transport 层，并最终创建 NettyServer 来接收客户端的请求。

- 序列化优化处理

  完成 ProtocolServer 的启动之后，`export()` 方法最后会调用 `optimizeSerialization()` 方法对指定的序列化算法进行优化。

  在使用某些序列化算法（例如 Kryo、FST 等）时，为了让其能发挥出最佳的性能，最好将需要被序列化的类提前注册到 Dubbo 系统中。在 `DubboProtocol.optimizeSerialization()` 方法中，会获取该优化器中注册的类，通知底层的序列化算法进行优化，序列化的性能将会被大大提升。

  在进行序列化的时候，会级联到很多 Java 内部的类（例如，数组、各种集合类型等），Kryo、FST 等序列化算法已经自动将JDK 中的常用类进行了注册，无须重复注册。

  > 按照 Dubbo 官方文档的说法，即使不注册任何类进行优化，Kryo 和 FST 的性能依然普遍优于Hessian2 和 Dubbo 序列化。

#### 服务引用

创建共享连接的实现细节是在 `getSharedClient()` 方法中，它首先从 referenceClientMap 缓存（Map 类型）中查询 Key（host 和 port 拼接成的字符串）对应的共享 Client 集合，如果查找到的 Client 集合全部可用，则直接使用这些缓存的 Client，否则要创建新的 Client 来补充替换缓存中不可用的 Client。

> 当使用共享连接的时候，会区分不同的网络地址（host:port），一个地址只建立固定数量的共享连接。Consumer 调用 Provider 中的多个服务时，通过固定数量的共享 TCP 长连接进行数据传输，可以达到减少服务端连接数的目的。

创建独享 Client 的入口在 `DubboProtocol.initClient()` 方法，它首先会在 URL 中设置一些默认的参数，然后根据 LAZY_CONNECT_KEY 参数决定是否使用 LazyConnectExchangeClient 进行封装，实现懒加载功能。

### HttpProtocol

gRPC、HTTP、WebService、Hessian、Thrift 等协议对应的 Protocol 实现，都是继承自 AbstractProxyProtocol 抽象类。

Dubbo 中使用 HTTP 协议 + JSON-RPC 的方式实现跨语言调用，HTTP 协议和 JSON 都是天然跨语言的标准。

#### JSON-RPC

Dubbo 中支持的 HTTP 协议实际上使用的是 JSON-RPC 协议，JSON-RPC 是基于 JSON 的跨语言远程调用协议。Dubbo 使用 jsonrpc4j 库来实现 JSON-RPC 协议

> 成熟的 JSON-RPC 框架，例如 jsonrpc4j、jpoxy。
>
> Dubbo 使用 jsonrpc4j 库来实现 JSON-RPC 协议，jsonrpc4j 本身体积小巧，使用方便，既可以独立使用也可以与 Spring 无缝集合。



## Remoting层

Remoting 层包括 Exchange、Transport和Serialize 三个子层次。

Dubbo 并没有自己实现一套完整的网络库，而是使用现有的、相对成熟的第三方网络库，例如，Netty、Mina 或是 Grizzly 等 NIO 框架。可以根据自己的实际场景和需求修改配置，选择底层使用的 NIO 框架。每个子模块对应一个第三方 NIO 框架，例如 dubbo-remoting-netty4 子模块使用 Netty4 实现 Dubbo 的远程通信，dubbo-remoting-zookeeper Apache Curator 实现了与 Zookeeper 的交互。

- buffer 包：定义了缓冲区相关的接口、抽象类以及实现类。缓冲区在NIO框架中是一个不可或缺的角色，在各个 NIO 框架中都有自己的缓冲区实现。这里的 buffer 包在更高的层面，抽象了各个 NIO 框架的缓冲区，同时也提供了一些基础实现。
- exchange 包：抽象了 Request 和 Response 两个概念，并为其添加很多特性。这是**整个远程调用核心的部分**。
- transport 包：对网络传输层的抽象，但它只负责抽象单向消息的传输。有很多网络库可以实现网络传输的功能，例如 Netty、Grizzly 等， transport 包是在这些网络库上层的一层抽象。
- 其它接口：Endpoint、Channel、Transporter、Dispatcher 等顶层接口放到了 org.apache.dubbo.remoting 这个包，这些接口是 Dubbo Remoting 的核心接口。

### ChannelBuffer

Buffer 是一种字节容器，在 Netty 等 NIO 框架中都有类似的设计，例如，Java NIO 中的 ByteBuffer、Netty4 中的 ByteBuf。

Dubbo 抽象出了 ChannelBuffer 接口对底层 NIO 框架中的 Buffer 设计进行统一。ChannelBuffer 接口的设计与 Netty4 中 ByteBuf 抽象类的设计基本一致，也有 readerIndex 和 writerIndex 指针的概念。

- HeapChannelBuffer

  HeapChannelBuffer 是基于字节数组的 ChannelBuffer 实现。

- ByteBufferBackedChannelBuffer

  ByteBufferBackedChannelBuffer 是基于 Java NIO 中 ByteBuffer 的 ChannelBuffer 实现。

- NettyBackedChannelBuffer

  NettyBackedChannelBuffer 是基于 Netty 中 ByteBuf 的 ChannelBuffer 实现。



## Exchange层

Dubbo 将信息交换行为抽象成 Exchange 层，封装了请求-响应的语义，即关注一问一答的交互模式，实现了同步转异步。

### ExchangeChannel

HeaderExchangeChannel 是 ExchangeChannel 的实现，主要是实现了数据接收过来数据的分发，能够根据接收过来的数据类型不同作出不同的处理。它本身是 Channel 的装饰器，封装了一个 Channel 对象，其 `send()` 方法和 `request()` 方法的实现都是依赖底层修饰的这个 Channel 对象实现的。

- `HeaderExchangeChannel.request()` 方法中完成 DefaultFuture 对象的创建之后，会将请求通过底层的 Dubbo Channel 发送出去，发送过程中会触发沿途 ChannelHandler 的 `sent()` 方法。

  NettyChannel 的 `send()` 方法：

  ```java
  ChannelFuture future = channel.write(message); // io.netty.channel
  ```

  > Channel 执行写入方法时会执行所有的 OutboundHandler。
  >
  > ChannelHandlerContext 执行写入方法时只会执行当前 handler 之前的 OutboundHandler。

- Consumer 收到对端返回的响应，会触发 Dubbo Channel 中各个 ChannelHandler 的 received() 方法。

- 当响应传递到 **HeaderExchangeHandler** 的时候，会通过调用 `handleResponse()` 方法进行处理，其中调用了 `DefaultFuture.received()` 方法，该方法会找到响应关联的 DefaultFuture 对象（根据请求 ID 从 FUTURES 集合查找）并调用 `doReceived()` 方法，将 DefaultFuture 设置为完成状态。

### HeaderExchangeHandler

HeaderExchangeHandler 是 ExchangeHandler 的装饰器，其中维护了一个 ExchangeHandler 对象，ExchangeHandler 接口是 Exchange 层与上层交互的接口之一，上层调用方可以实现该接口完成自身的功能。然后再由 HeaderExchangeHandler 修饰，具备 Exchange 层处理 Request-Response 的能力。最后再由 Transport ChannelHandler 修饰，具备 Transport 层的能力。

`received()` 方法处理的消息分类：

- 只读请求会由 `handlerEvent()` 方法进行处理，它会在 Channel 上设置 channel.readonly 标志，上层调用中会读取该值。
- 双向请求由 `handleRequest()` 方法进行处理，会先对解码失败的请求进行处理，返回异常响应。然后将正常解码的请求交给上层实现的 ExchangeHandler 进行处理，并添加回调。上层 ExchangeHandler 处理完请求后，会触发回调，根据处理结果填充响应结果和响应码，并向对端发送。
- 单向请求直接委托给上层 ExchangeHandler 实现的 `received()` 方法进行处理，由于不需要响应，HeaderExchangeHandler 不会关注处理结果。
- 对于 Response ，HeaderExchangeHandler 会通过 `handleResponse()` 方法将关联的 DefaultFuture 设置为完成状态（或是异常完成状态）。

### HeaderExchangeClient

HeaderExchangeClient 是 Client 装饰器，主要为其装饰的 Client 添加两个功能：

- 维持与 Server 的长连状态，这是通过**定时发送心跳消息**实现的。
- 在因故障掉线之后，进行重连，这是通过**定时检查连接状态**实现的。

### HeaderExchangeServer

HeaderExchangeServer 是 RemotingServer 的装饰器，实现自 RemotingServer 接口的大部分方法都委托给了所修饰的 RemotingServer 对象。在 HeaderExchangeServer 的构造方法中，会启动一个 CloseTimerTask 定时任务，定期关闭长时间空闲的连接。



## Transporter层

### Transporter

Transporter 接口上有 @SPI 注解，它是一个扩展接口，默认使用 netty 这个扩展名，@Adaptive 注解的出现表示动态生成适配器类，会先后根据 server 和 transporter 的值确定 RemotingServer 的扩展实现类，先后根据 client 和 transporter 的值确定 Client 接口的扩展实现。

可以通过 Dubbo SPI 修改使用的具体 Transporter 扩展实现，从而切换到不同的 Client 和 RemotingServer 实现，达到底层 NIO 库切换的目的，而且无须修改任何代码，符合开放-封闭原则。

Transporters 是门面类，封装了 Transporter 对象的创建（通过 Dubbo SPI）以及 ChannelHandler 的处理。

### NettyServer

NettyServer 实现的 `doOpen()` 方法是启动一个 Netty 的 Server 端基本流程。包括初始化 ServerBootstrap、创建 Boss EventLoopGroup 和 Worker EventLoopGroup、创建 ChannelInitializer 指定如何初始化 Channel 上的 ChannelHandler 等一系列 Netty 使用的标准化流程。

- InternalDecoder 和 InternalEncoder

  继承 Netty 中的 MessageToByteEncoder 和 ByteToMessageDecoder。

  将真正的编解码功能委托给 NettyServer 关联的这个 Codec2 对象去处理。

- IdleStateHandler

  Netty 提供的一个工具型 ChannelHandler，用于定时心跳请求的功能或是自动关闭长时间空闲连接的功能。

- NettyServerHandler

  继承了 ChannelDuplexHandler，这是 Netty 提供的一个同时处理 Inbound 数据和 Outbound 数据的 ChannelHandler。

  创建 NettyServerHandler，第二个参数传入的是 NettyServer 这个对象，会将数据委托给NettyServer 。

```java
@Override
protected void doOpen() throws Throwable {
    // 创建ServerBootstrap
    bootstrap = new ServerBootstrap();
    // 创建boss EventLoopGroup
    bossGroup = NettyEventLoopFactory.eventLoopGroup(1, "NettyServerBoss");
    // 创建worker EventLoopGroup
    workerGroup = NettyEventLoopFactory.eventLoopGroup(
        getUrl().getPositiveParameter(IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
        "NettyServerWorker");

    // 创建NettyServerHandler，Netty中的ChannelHandler实现，
    final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);

    // 获取当前NettyServer创建的所有Channel，这里的channels集合中的 Channel不是Netty中的Channel对象，而是Dubbo Remoting层的Channel对象
    channels = nettyServerHandler.getChannels();

    // 初始化ServerBootstrap，指定boss和worker EventLoopGroup
    bootstrap.group(bossGroup, workerGroup)
        .channel(NettyEventLoopFactory.serverSocketChannelClass())
        .option(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
        .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
        .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
                // 连接空闲超时时间
                int idleTimeout = UrlUtils.getIdleTimeout(getUrl());
                // NettyCodecAdapter中会创建Decoder和Encoder
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                if (getUrl().getParameter(SSL_ENABLED_KEY, false)) {
                    ch.pipeline().addLast("negotiation", 
                    	SslHandlerInitializer.sslServerHandler(getUrl(), nettyServerHandler));
                }
                ch.pipeline()
                    // 注册Decoder和Encoder
                    .addLast("decoder", adapter.getDecoder()) // ChannelInboundHandlerAdapter
                    .addLast("encoder", adapter.getEncoder()) // ChannelOutboundHandlerAdapter
                    // 注册IdleStateHandler
                    .addLast("server-idle-handler", 
                             new IdleStateHandler(0, 0, idleTimeout, MILLISECONDS)) // ChannelDuplexHandler
                    // 注册NettyServerHandler
                    .addLast("handler", nettyServerHandler); // ChannelDuplexHandler
            }
        });

    // 绑定指定的地址和端口
    ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
    channelFuture.syncUninterruptibly(); // 等待bind操作完成
    channel = channelFuture.channel();
}
```

在 NettyCodecAdapter 里面定义内部类 InternalEncoder（继承 Netty 中的 MessageToByteEncoder）实现 Dubbo 的自定义编码器，定义内部类 ByteToMessageDecoder（继承 Netty 中的 ByteToMessageDecoder）实现 Dubbo 自定义解码器。


### NettyClient

在 NettyClient 的 `doOpen()` 方法中会通过 Bootstrap 构建客户端，其中会完成连接超时时间、keepalive 等参数的设置，以及 ChannelHandler 的创建和注册。

NettyClientHandler 实现了 Netty 中的 ChannelDuplexHandler，将所有方法委托给 NettyClient 关联的 ChannelHandler 对象进行处理。

### NettyChannel

send() 方法会通过底层关联的 Netty 框架 Channel，将数据发送到对端。

### ChannelHandler

AbstractServer、AbstractClient 以及 Channel 实现，都是通过 AbstractPeer 实现了 ChannelHandler 接口，但只是做了一层简单的委托（也可以说成是装饰器），将全部方法委托给了其底层关联的 ChannelHandler 对象。



## Serialize层

Dubbo 为了支持多种序列化算法，单独抽象了一层 Serialize 层，在整个 Dubbo 架构中处于最底层，对应的模块是 dubbo-serialization 模块。

### Serialization

dubbo-serialization-api 模块中定义了 Dubbo 序列化层的核心接口，其中最核心的是 Serialization 这个接口，它是一个扩展接口，被 @SPI 接口修饰，默认扩展实现是 Hessian2Serialization。

> 序列化算法详见 Java 基础。



## 其它

### ExecutorRepository

ExecutorRepository 负责创建并管理 Dubbo 中的线程池，该接口虽然是个 SPI 扩展点，但是只有一个默认实现 DefaultExecutorRepository。

在 createExecutor() 方法中，会通过 Dubbo SPI 查找 ThreadPool 接口的扩展实现，并调用其 getExecutor() 方法创建线程池。ThreadPool 接口被 @SPI 注解修饰，默认使用 FixedThreadPool 实现，但是 ThreadPool 接口中的 getExecutor() 方法被 @Adaptive 注解修饰，动态生成的适配器类会优先根据 URL 中的 threadpool 参数选择 ThreadPool 的扩展实现。

- CacheThreadPool：可以指定核心线程数、最大线程数以及缓冲队列长度。

- LimitedThreadPool：可以指定核心线程数、最大线程数以及缓冲队列长度，非核心线程不会被回收。

- FixedThreadPool：核心线程数和最大线程数一致，且不会被回收。

- EagerThreadPool：

  创建的线程池是 EagerThreadPoolExecutor（继承了 JDK 提供的 ThreadPoolExecutor），使用的队列是 TaskQueue（继承了LinkedBlockingQueue）。

  该线程池与 ThreadPoolExecutor 不同的是：

  - 在线程数没有达到最大线程数，EagerThreadPoolExecutor 会优先创建线程来执行任务，而不是放到缓冲队列中。
  - 当线程数达到最大值时，EagerThreadPoolExecutor 会将任务放入缓冲队列，等待空闲线程。

  EagerThreadPoolExecutor 覆盖了 ThreadPoolExecutor 中的两个方法：`execute()` 方法和 `afterExecute()` 方法。维护了一个AtomicInteger 类型的 submittedTaskCount 字段，用来记录当前在线程池中的任务总数。

  关联的 TaskQueue 实现中覆盖了 `LinkedBlockingQueue.offer()` 方法，会判断线程池的 submittedTaskCount 值是否已经达到最大线程数，如果未超过，则会返回 false，迫使线程池创建新线程来执行任务。

### Codec

实现RPC调用的 Request 和 Response 对象的编码和解码类，RPC调用实现的核心传输也就是这两个类对象。

- TransportCodec 中根据 `getSerialization()` 方法选择的序列化方法对传入消息或 ChannelBuffer 进行序列化或反序列化。

- TelnetCodec 继承了 TransportCodec 序列化和反序列化的基本能力，同时还提供了对 Telnet 命令处理的能力。

- ExchangeCodec 在 TelnetCodec 的基础之上，添加了处理协议头的能力。

- DubboCodec 在 ExchangeCodec 基础之上，添加了解析 Dubbo 消息体的功能。



# 附录

## Dubbo 示例

- dubbo-demo-interface 模块

  定义业务接口：

  ```java
  public interface DemoService { 
      String sayHello(String name);
  } 
  ```

- dubbo-demo-provider 模块

  依赖 DemoService 公共接口：

  ```xml
  <dependency> 
      <groupId>org.apache.dubbo</groupId> 
      <artifactId>dubbo-demo-interface</artifactId> 
      <version>${version}</version> 
  </dependency> 
  ```

  初始化Spring容器，指定注册中心地址（ZooKeeper 地址），Dubbo 把暴露的 DemoService 服务注册到 ZooKeeper 中。

  DemoServiceImpl 实现了 DemoService 接口，并且在 org.apache.dubbo.demo.provider 目录下被扫描到，暴露为 Dubbo 服务。

  ```java
  public class Application { 
      public static void main(String[] args) throws Exception { 
  	    // 初始化Spring容器，从ProviderConfiguration这个类的注解上获取相关配置信息 
          AnnotationConfigApplicationContext context =  new AnnotationConfigApplicationContext( 
              ProviderConfiguration.class); 
          context.start(); 
      } 
  
      @Configuration
      @EnableDubbo(scanBasePackages = "org.apache.dubbo.demo.provider") // @EnableDubbo注解指定包下的Bean都会被扫描，并做Dubbo服务暴露出去   
      @PropertySource("classpath:/spring/dubbo-provider.properties") // @PropertySource注解指定了其他配置信息 
      static class ProviderConfiguration { 
          @Bean 
          public RegistryConfig registryConfig() { 
              RegistryConfig registryConfig = new RegistryConfig(); 
              registryConfig.setAddress("zookeeper://127.0.0.1:2181"); 
              return registryConfig; 
          } 
      } 
  } 
  ```

- dubbo-demo-consumer 模块

  依赖 DemoService 公共接口：

  ```xml
  <dependency> 
      <groupId>org.apache.dubbo</groupId> 
      <artifactId>dubbo-demo-interface</artifactId> 
      <version>${version}</version> 
  </dependency> 
  ```

  通过 AnnotationConfigApplicationContext 初始化 Spring 容器，也会扫描指定目录下的 Bean，会扫到 DemoServiceComponent 这个 Bean，其中通过 @Reference 注解注入了 Dubbo 服务相关的 Bean。

  ```java
  @Component("demoServiceComponent") 
  public class DemoServiceComponent implements DemoService { 
      @Reference // 注入Dubbo服务 
      private DemoService demoService; 
  
      @Override 
      public String sayHello(String name) { 
          return demoService.sayHello(name); 
      } 
  } 
  ```



## Curator

Dubbo Provider 在启动时会将自身的服务信息整理成 URL 注册到注册中心，Dubbo Consumer 在启动时会向注册中心订阅感兴趣的 Provider 信息，之后 Provider 和 Consumer 才能建立连接。

Dubbo 目前支持 Consul、etcd、Nacos、ZooKeeper、Redis 等多种开源组件作为注册中心。官方推荐使用 ZooKeeper 作为注册中心，它是在实际生产中最常用的注册中心实现。

Apache Curator 是 Apache 基金会提供的一款 ZooKeeper 客户端，它提供了一套易用性和可读性非常强的 Fluent 风格的客户端 API ，可以帮助我们快速搭建稳定可靠的 ZooKeeper 客户端程序。



## 时间轮 HashedWheelTimer

![image-20240729205653598](https://gitee.com/fushengshi/image/raw/master/image-20240729205653598.png)

时间轮是一种高效的、批量管理定时任务的调度模型。时间轮一般会实现成一个环形结构，类似一个时钟，分为很多槽，一个槽代表一个时间间隔，每个槽使用双向链表存储定时任务。指针周期性地跳动，跳动到一个槽位，就执行该槽位的定时任务。

> JDK 提供的 java.util.Timer 和 DelayedQueue 等工具类，可以帮助我们实现简单的定时任务管理，其底层实现使用的是**堆**这种数据结构，插入和删除的时间复杂度都是 O(logn)，无法支持大量的定时任务。

Dubbo 中时间轮的应用：

- 失败重试

  例如 Provider 向注册中心进行注册失败时的重试操作，或是 Consumer 向注册中心订阅时的失败重试等。

- 周期性定时任务

  例如定期发送心跳请求，请求超时的处理，或是网络连接断开后的重连机制。

Dubbo 的时间轮实现位于 dubbo-common 模块的 org.apache.dubbo.common.timer 包中。

```java
private static final class HashedWheelBucket {
	// 双向链表的首尾两个节点   
    private HashedWheelTimeout head;
    private HashedWheelTimeout tail;
    ...
}
```

HashedWheelBucket 是时间轮中的一个槽，时间轮中的槽实际上就是一个用于缓存和管理双向链表的容器，双向链表中的每一个节点就是一个 HashedWheelTimeout 对象，也就关联了一个 TimerTask 定时任务。

```java
private final class Worker implements Runnable {
    private final Set<Timeout> unprocessedTimeouts = new HashSet<Timeout>();

    private long tick;

    @Override
    public void run() {
        startTime = System.nanoTime();
        if (startTime == 0) {
            startTime = 1;
        }

        startTimeInitialized.countDown();
        do {
            // sleep直到指针转动
            final long deadline = waitForNextTick();
            if (deadline > 0) {
                int idx = (int) (tick & mask);
                processCancelledTasks();
                
                // 根据当前指针定位对应槽，处理该槽位的双向链表中的定时任务。
                HashedWheelBucket bucket = wheel[idx];
                transferTimeoutsToBuckets();
                
                // 执行TimerTask的run()
                bucket.expireTimeouts(deadline);
                tick++;
            }
        } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);
    }
    ...
}
```

HashedWheelTimer 是 Timer 接口的实现，它通过时间轮算法实现了一个定时器。HashedWheelTimer 会根据当前时间轮指针选定对应的槽（HashedWheelBucket），从双向链表的头部开始迭代，对每个定时任务（HashedWheelTimeout）进行计算，属于当前时钟周期则取出运行，不属于则将其剩余的时钟周期数减一操作。



## 服务调用

![image-20240729205013461](https://gitee.com/fushengshi/image/raw/master/image-20240729205013461.png)

一个远程调用请求的发送与接收过程：

- 服务消费者通过代理对象 Proxy 发起远程调用，接着通过网络客户端 Client 将编码后的请求发送给服务提供方的网络层上（Server）。

- Server 在收到请求后：
  - 对数据包进行解码。
  - 将解码后的请求发送至分发器 Dispatcher，再由分发器将请求派发到指定的线程池上，最后由线程池调用具体的服务。

### Customer 代理生成

Consumer 端 RPC 服务接口生成动态代理类。

在 `createProxy()` 方法中，通过 **Protocol 的 `refer()` 方法**根据接口类型（Class<T> type）返回一个 Invoker 对象。这个 Invoker 对象作为 `getProxy()` 的参数，Consumer 端可以通过这个 Invoker 请求到 Provider 端的服务。

```java
@Override
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    return new AsyncToSyncInvoker<>(protocolBindingRefer(type, url));
}
```

服务接口生成动态代理类反编译结果：

```java
public class proxy0 implements ClassGenerator.DC, EchoService, DemoService {
    // 方法数组。
    public static Method[] methods;
    private InvocationHandler handler;

    public proxy0(InvocationHandler invocationHandler) {
        this.handler = invocationHandler;
    }

    public proxy0() {
    }

    public String sayHello(String string) {
        // 将参数存储到 Object 数组中。
        Object[] arrobject = new Object[]{string};
        // 调用 InvocationHandler 实现类的 invoke 方法得到调用结果。
        Object object = this.handler.invoke(this, methods[0], arrobject);
        // 返回调用结果。
        return (String)object;
    }
}
```



## Spring 整合

- 当 Dubbo 和 Spring 整合时，会往容器中注入 2 个 BeanPostProcessor，ServiceAnnotationBeanPostProcessor 将 @Service 注解的类封装成 ServiceBean 注入容器 ReferenceAnnotationBeanPostProcessor，将 @Reference 注解的接口封装成 ReferenceBean 注入容器。

  ```java
  // 注册ReferenceAnnotationBeanPostProcessor
  registerInfrastructureBean(registry, ReferenceAnnotationBeanPostProcessor.BEAN_NAME,
                             ReferenceAnnotationBeanPostProcessor.class);
  
  public class ReferenceAnnotationBeanPostProcessor extends AbstractAnnotationBeanPostProcessor implements
      ApplicationContextAware, ApplicationListener<ServiceBeanExportedEvent> {
      ...
  }
  
  public abstract class AbstractAnnotationBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter implements MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware, BeanClassLoaderAware, EnvironmentAware, DisposableBean {
      // 对属性值进行修改
      public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
          InjectionMetadata metadata = this.findInjectionMetadata(beanName, bean.getClass(), pvs);
  
          try {
              // 调用doGetInjectedBean创建ReferenceBean
              metadata.inject(bean, beanName, pvs);
              return pvs;
          } catch (BeanCreationException var7) {
              throw var7;
          } catch (Throwable var8) {
              throw new BeanCreationException(beanName, "Injection of @" + this.getAnnotationType().getSimpleName() + " dependencies is failed", var8);
          }
      }
  }
  ```

- ReferenceBean 实现 FactoryBean，bean加载过程完成 `createProxy()`。

  ```java
  public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean,
  ApplicationContextAware, InitializingBean, DisposableBean {
      @Override
      public Object getObject() {
          return get();
      }
  }
  
  public synchronized T get() {
      if (destroyed) {
          throw new IllegalStateException("");
      }
      if (ref == null) {
          init();
      }
      return ref;
  }
  
  public synchronized void init() {
      ...
      // 调用createProxy()方法，创建代理
      ref = createProxy(map);
  }
  
  private T createProxy(Map<String, String> map) {
    // 通过Protocol的适配器选择对应的Protocol实现创建Invoker对象
      invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
      ...
  
      // 通过ProxyFactory适配器选择合适的ProxyFactory扩展实现，创建代理对象
      return (T) PROXY_FACTORY.getProxy(invoker, ProtocolUtils.isGeneric(generic));
  }
  ```
  
  > 自定义 FactoryBean 原理详见 Spring 源码。
  
  














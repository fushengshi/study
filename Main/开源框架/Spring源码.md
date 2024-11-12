# Spring 源码

![image-20240803171746819](https://gitee.com/fushengshi/image/raw/master/image-20240803171746819.png)

## Spring IOC

IoC（Inversion of Control，控制反转） 是一种设计思想。IoC 的思想就是将原本在程序中手动创建对象的控制权，交由 Spring 框架来管理。

在 Spring 中 IoC 容器是 Spring 用来实现 IoC 的载体， IoC 容器实际上就是个 Map（key，value），Map 中存放的是各种对象。

- BeanFactory

  BeanFactory 是 IOC 容器或对象工厂，所有的 Bean 都是由 BeanFactory 管理的，职责包括实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。

  ApplicationContext 是 BeanFactory 的扩展，功能得到了进一步增强，比如更易与 Spring AOP 集成、消息资源处理（国际化处理）、事件传递及各种不同应用层的 context 实现（如针对 web 应用的 WebApplicationContext）。

- BeanDefinition

  BeanDefinition 中保存了 Bean 信息，比如这个 Bean 指向的是哪个类、是否是单例的、是否懒加载、这个 Bean 依赖哪些 Bean 等等。

- FactoryBean 

  FactoryBean 是一个能生产或修饰对象生成的工厂 Bean，类似于设计模式中的工厂模式和装饰器模式。

  Bean A 如果实现了 FactoryBean 接口，那么 A 就变成了一个工厂，根据 A 的名称获取到的实际上是工厂调用 `getObject()` 返回的对象，而不是A本身。



### refresh()

```java
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
    throws BeansException {
    super(parent);
    setConfigLocations(configLocations);

    if (refresh) {
        refresh(); // 核心方法
    }
}

@Override
public void refresh() throws BeansException, IllegalStateException {
    // 准备工作。
    prepareRefresh();

    // 重点
    // 初始化 BeanFactory、加载 Bean、注册 Bean 等等（Bean 还没有初始化）。
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // 设置 BeanFactory 的类加载器，添加几个BeanPostProcessor，手动注册几个特殊的Bean。
    prepareBeanFactory(beanFactory);
    
    // 子类的扩展点，此时所有的 Bean 都加载、注册完成，子类可以添加一些特殊的 BeanFactoryPostProcessor 实现类。
    postProcessBeanFactory(beanFactory);

    // 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory 方法。
    invokeBeanFactoryPostProcessors(beanFactory);

    // 注册 BeanPostProcessor的实现类。
    // 此接口方法 postProcessBeforeInitialization 和 postProcessAfterInitialization 分别在 Bean 初始化之前和初始化之后执行。
    registerBeanPostProcessors(beanFactory);

    // 初始化当前 ApplicationContext 的 MessageSource。
    initMessageSource();

    // 初始化当前 ApplicationContext 的事件广播器。
    initApplicationEventMulticaster();

    // 模板方法(钩子方法)，子类可以初始化一些特殊的Bean（在初始化Bean之前）
    onRefresh();

    // 注册事件监听器，监听器需要实现 ApplicationListener 接口。
    registerListeners();

    // 重点
    // 初始化Bean（lazy-init的除外），doCreateBean。
    finishBeanFactoryInitialization(beanFactory);

    // 广播事件，ApplicationContext初始化完成。
    finishRefresh();
    
    ...
}
```



### doCreateBean()

![image-20240803171927319](https://gitee.com/fushengshi/image/raw/master/image-20240803171927319.png)

- Bean 实例化

  Bean 容器首先会找到配置文件中的 Bean 定义，然后使用 Java 反射 API 来创建 Bean 的实例。

- 属性赋值

  为 Bean 设置相关属性和依赖，例如 @Autowired 等注解注入的对象、@Value 注入的值、setter 方法或构造函数注入依赖和值、@Resource 注入的各种资源。

- Bean 初始化

  - 如果 Bean 实现了 BeanNameAware 接口，调用 `setBeanName()` 方法，传入 Bean 的名字。
  - 如果 Bean 实现了 BeanClassLoaderAware 接口，调用 `setBeanClassLoader()` 方法，传入 ClassLoader 对象的实例。
  - 如果 Bean 实现了 BeanFactoryAware 接口，调用 `setBeanFactory()` 方法，传入 BeanFactory 对象的实例。
  - 与上面的类似，如果实现了其他 *.Aware 接口，就调用相应的方法。
  - 如果有和加载这个 Bean 的 Spring 容器相关的 BeanPostProcessor 对象，执行 `postProcessBeforeInitialization()` 方法
  - 如果 Bean 实现了 **InitializingBean** 接口，执行 `afterPropertiesSet()` 方法。
  - 如果 Bean 在配置文件中的定义包含 `init-method` 属性，执行指定的方法。
  - 如果有和加载这个 Bean 的 Spring 容器相关的 BeanPostProcessor 对象，执行 `postProcessAfterInitialization()` 方法。

- 销毁 Bean

  销毁并不是立马把 Bean 销毁掉，而是把 Bean 的销毁方法先记录下来，将来需要销毁 Bean 或者销毁容器的时候，就调用这些方法去释放 Bean 所持有的资源。

```java
protected Object doCreateBean(String beanName, RootBeanDefinition rootBeanDefinition, @Nullable Object[] args) throws BeanCreationException {
    ...
    if (instanceWrapper == null) {
        // Bean实例化
        instanceWrapper = createBeanInstance(beanName, rootBeanDefinition, args);
    }

    // 是否需要提前曝光：单例，允许循环依赖，当前bean正在创建
    boolean earlySingletonExposure = (rootBeanDefinition.isSingleton() && this.allowCircularReferences && isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        // 创建 objectFactory 加入三级缓存
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, rootBeanDefinition, bean));
    }

    Object exposedObject = bean;
    try {
        // 对bean进行填充，将各个属性值注入
        populateBean(beanName, rootBeanDefinition, instanceWrapper);

        // 调用初始化方法，比如init-method（在bean实例化前调用指定方法根据用户业务进行相应的实例化）
        exposedObject = initializeBean(beanName, exposedObject, rootBeanDefinition);
    } catch (Throwable ex) {
    }

    if (earlySingletonExposure) {
        // 从缓存中获取，第二个参数是false表示只能从一级缓存、二级缓存中获取，此时还未放入一级缓存，故只能从二级缓存中获取
        Object earlySingletonReference = getSingleton(beanName, false);

        // earlySingletonReference只有在检测到循环依赖的情况不会为空
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            } else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                ...
            }
        }
    }
    ...
    return exposedObject;
}
```



### 循环依赖

SpringBoot 2.6.x 以后官方不再推荐编写存在循环依赖的代码，建议开发者自己写代码的时候去减少不必要的互相依赖。

如果不重构循环依赖的代码：

- 在全局配置文件中设置允许循环依赖存在 `spring.main.allow-circular-references=true`，不推荐。

- 在导致循环依赖的 Bean 上添加 @Lazy 注解。

  如果一个 Bean 被标记为懒加载，那么它不会在 Spring IoC 容器启动时立即实例化，而是在第一次被请求时才创建。

  A 的构造器上添加 @Lazy 注解，Spring 创建 A 的 Bean，创建时需要注入 B 的属性，由于在 A 上标注了 @Lazy 注解，因此 Spring 会去创建一个 B 的代理对象，将这个代理对象注入到 A 中。

#### Spring 的三级缓存

- 一级缓存

  ```java
  private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
  ```

  用来存放就绪状态的 Bean。保存在该缓存中的 Bean 所实现 Aware 子接口的方法已经回调完毕，自定义初始化方法已经执行完毕，已经过 BeanPostProcessor 实现类的 `postProcessorBeforeInitialization()`、`postProcessorAfterInitialization()` 方法处理。

- 二级缓存

  ```java
  private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
  ```

  用来存放早期曝光的 Bean，一般只有处于循环引用状态的Bean才会被保存在该缓存中。

  保存在该缓存中的 Bean 所实现 Aware 子接口的方法还未回调，自定义初始化方法未执行，也未经过 BeanPostProcessor 实现类的`postProcessorBeforeInitialization()`、`postProcessorAfterInitialization()` 方法处理。

  如果启用了Spring AOP，并且处于切点表达式处理范围之内，会被增强创建代理对象。

  > 普通Bean被增强的时机是 AbstractAutoProxyCreator 实现的 BeanPostProcessor 的 `postProcessorAfterInitialization()` 方法中。
  >
  > 处于循环引用状态的Bean被增强的时机是在 AbstractAutoProxyCreator 实现的 SmartInstantiationAwareBeanPostProcessor的 `getEarlyBeanReference()` 方法中。

- 三级缓存

  ```java
  private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
  ```

  用来存放创建用于获取 Bean 的工厂类 ObjectFactory 实例，BeanFactory 实现了 ObjectFactory 接口。



#### 循环依赖执行过程

- 实例化 A

  为 A 创建一个 ObjectFactory，**放入到三级缓存中**，工厂生产对象A 的逻辑为回调函数 `getEarlyBeanReference()`。

  ```java
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, rootBeanDefinition, bean));
  ```
  
  > 此时 A 还未完成属性填充和初始化方法（@PostConstruct）的执行，A 只是一个半成品。

- 发现 A 需要注入 B 对象

  一级、二级、三级缓存均未发现对象 B。

- 实例化 B

  为 B 创建一个 ObjectFactory，放入到三级缓存中。

  > 此时 B 还未完成属性填充和初始化方法（@PostConstruct）的执行，B 只是一个半成品。

- 发现 B 需要注入 A 对象

  此时在一级、二级未发现对象 A，但是在三级缓存中发现了对象 A。

- 从三级缓存中工厂生产对象 A

  ```java
  Object sharedInstance = getSingleton(beanName);
  ```

  执行回调函数 `getEarlyBeanReference()` 的逻辑，如果被 AOP 增强则生成代理对象。

  将对象 A 放入二级缓存中，同时删除三级缓存中的对象 A。

  > 此时的 A 还是一个半成品，并没有完成属性填充和执行初始化方法。

- 对象 A 注入到对象 B 中。

- 对象 B 完成属性填充，执行初始化方法。

  B 放入到一级缓存中，同时删除二级缓存中的对象 B。

  > 此时对象 B 已经是一个成品。

- 对象 B 注入到对象 A 中。

  对象 A 得到的是一个完整的对象 B。

- 对象 A 完成属性填充，执行初始化方法。

  A 放入到一级缓存中，同时删除二级缓存中的对象 A。



#### 只用两级缓存

没有 AOP 的情况下，可以只使用一级和三级缓存来解决循环依赖问题。但是涉及到 AOP 时，二级缓存确保了即使在 Bean 的创建过程中有多次对早期引用的请求，也始终只返回同一个代理对象，从而避免了同一个 Bean 有多个代理对象的问题。

如果只用两级缓存来解决循环依赖的话，所有 Bean 都需要在实例化完成之后就立即为其创建代理。而 Spring 的设计原则是在 Bean 初始化完成之后才为其创建代理，所以 Spring 选择了三级缓存。

> Spring 的设计原则是在 Bean 初始化完成之后为其创建代理。
>
> 因为循环依赖的出现，导致了 Spring 不得不提前创建代理，如果不提前创建代理对象注入的就是原始对象，会产生错误。



#### @Async 循环依赖报错

@Async 注解可以被标注在方法上（也可以标注在类上代表所有方法都异步调用），异步地调用该方法。调用者将在调用时立即返回，方法的实际执行将提交给 Spring TaskExecutor 的任务中，由指定的线程池中的线程执行。

使用了 @Async 注解方法后，循环依赖报错。

```java
protected Object doCreateBean(String beanName, RootBeanDefinition rootBeanDefinition, @Nullable Object[] args) throws BeanCreationException {
    ...

    // 是否需要提前曝光：单例，允许循环依赖，当前bean正在创建
    boolean earlySingletonExposure = (rootBeanDefinition.isSingleton() && this.allowCircularReferences && isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        // 创建 objectFactory 加入三级缓存
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, rootBeanDefinition, bean));
    }
        
    Object exposedObject = bean;
    try {
        // 对bean进行填充，将各个属性值注入
        populateBean(beanName, rootBeanDefinition, instanceWrapper);

        // 在这里进行初始化后的操作，如果已经进行过切面代理，则直接将原对象返回
		// 但是方法如果被 @Async 修饰，则会创建代理对象并将代理对象返回，exposedObject更改为代理对象，导致后面exposedObject != bean
        exposedObject = initializeBean(beanName, exposedObject, rootBeanDefinition);
    } catch (Throwable ex) {
    }

    if (earlySingletonExposure) {
        // 从缓存中获取，第二个参数是false表示只能从一级缓存、二级缓存中获取，此时还未放入一级缓存，故只能从二级缓存中获取
        Object earlySingletonReference = getSingleton(beanName, false);

        // earlySingletonReference只有在检测到循环依赖的情况不会为空
        if (earlySingletonReference != null) {
            // initializeBean如果已经进行过切面代理，则直接将原对象返回，故此时exposedObject == bean, 
            // 将exposedObject修改为二级缓存保存的切面代理后的对象作为结果返回。
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            } 
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                // 接下来就会判断这个Bean是否有其它Bean进行依赖，如果有则说明注入到其它Bean的依赖不是最终包装过后的Bean 
                String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName, "");
					}
            }
        }
    }
    ...
    return exposedObject;
}
```

addSingletonFactory 创建 objectFactory 加入三级缓存，工厂生产对象 的逻辑为回调函数 `getEarlyBeanReference()`。如果有 SmartInstantiationAwareBeanPostProcessor 子类的 beanPostProcessor 则创建代理类。AOP 增强AnnotationAwareAspectJAutoProxyCreator 继承自 SmartInstantiationAwareBeanPostProcessor，**循环依赖时** AOP 增强的类通过三级缓存执行该回调函数的逻辑，生成代理对象并添加 earlyProxyReferences 缓存。

对于 @Async 注解，AsyncAnnotationBeanPostProcessor 并不继承 SmartInstantiationAwareBeanPostProcessor，因此不会在`getEarlyBeanReference()` 中执行。

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition rootBeanDefinition, Object bean) {
    Object exposedObject = bean;
    if (!rootBeanDefinition.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor beanPostProcessor : getBeanPostProcessors()) {
            // SmartInstantiationAwareBeanPostProcessor
            if (beanPostProcessor instanceof SmartInstantiationAwareBeanPostProcessor) {
                SmartInstantiationAwareBeanPostProcessor smartInstantiationAwareBeanPostProcessor = (SmartInstantiationAwareBeanPostProcessor) beanPostProcessor;
                exposedObject = smartInstantiationAwareBeanPostProcessor.getEarlyBeanReference(exposedObject, beanName);
            }
        }
    }
    return exposedObject;
}

@Override
public Object getEarlyBeanReference(Object bean, String beanName) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    // 添加 earlyProxyReferences 缓存
    this.earlyProxyReferences.put(cacheKey, bean);
    // 生成代理对象
    return wrapIfNecessary(bean, beanName, cacheKey);
}
```

完成属性注入后，循环依赖的 AOP 增强类已经通过 `getEarlyBeanReference()` 生成了代理对象，@Async 类没有执行对应的beanPostProcessor。代码执行 initializeBean，执行所有 BeanPostProcessor，如果这些后处理是SmartInstantiationAwareBeanPostProcessor 的子类，则会因为存在 earlyProxyReferences 缓存直接返回原对象。

对于 @Async 注解，AsyncAnnotationBeanPostProcessor 逻辑将在该处执行，返回一个代理对象，导致后面 `exposedObject != bean`。

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition rootBeanDefinition) {
    Object wrappedBean = bean;
	
    ...
    
    if (rootBeanDefinition == null || !rootBeanDefinition.isSynthetic()) {
        // 调用后处理器，尽可能保证所有bean初始化后都会调用注册BeanPostProcessor的postProcessAfterInitialization方法进行处理
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}

@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
    throws BeansException {
    Object result = existingBean;
    for (BeanPostProcessor beanPostProcessor : getBeanPostProcessors()) {
        Object current = beanPostProcessor.postProcessAfterInitialization(result, beanName);
        if (current == null) {
            return result;
        }
        result = current;
    }
    return result;
}
```

通过 `getSingleton()` 如果找到二级缓存，则说明该类存在循环依赖，判断 exposedObject 和 bean 是否相同，如果相同说明 initializeBean 没有变更对象，即所有的增强已经执行，即可将 exposedObject 更改为二级缓存中的对象返回（注意 exposedObject 没有在 initializeBean 中执行逻辑增强，不是可返回的对象）。

对于 @Async 注解，执行了 AsyncAnnotationBeanPostProcessor 逻辑，`exposedObject != bean`，程序报错。



## Spring AOP

AOP 面向切面编程，可以将公共的代码抽离出来，动态的织入到目标类、目标方法中，提高编程的效率，也使程序变得更加优雅。事务、操作日志等都可以使用 AOP 实现。

对于 AOP 的实现，基本上都是通过 AnnotationAwareAspectJAutoProxyCreator 完成，可以根据 @Point 注解定义的切点自动代理相匹配的 Bean。通过自定义配置完成了 AnnotationAwareAspectJAutoProxyCreator 类型的自动注册，AnnotationAwareAspectJAutoProxyCreator 实现了 BeanPostProcessor（SmartInstantiationAwareBeanPostProcessor、InstantiationAwareBeanPostProcessor子类）接口。

### Spring AOP 代理时机

- Bean 初始化后

  `postProcessAfterInitialization()`

  Spring 的设计原则是在 Bean 初始化完成之后为其创建代理。

- Bean 实例化前

  对于 InstantiationAwareBeanPostProcessor，如果自定义了 targetSource，则在 bean 实例化前完成 Spring AOP 代理并且直接发生短路操作返回 bean。

  `postProcessBeforeInstantiation()`

  在 Bean 还没有实例化的时候就提前创建一个代理对象（创建了则不会继续后续的 Bean 的创建过程），例如 RPC 远程调用的实现，因为本地类没有远程能力，可以通过这种方式进行拦截。

  ```java
  @Override
  protected Object createBean(String beanName, RootBeanDefinition rootBeanDefinition, @Nullable Object[] args) throws BeanCreationException {
      ...
      
      Object bean = resolveBeforeInstantiation(beanName, rootBeanDefinitionToUse);
      // 短路判断，经过前置处理后返回的结果不为空，直接略过后续Bean的创建
      if (bean != null) {
          return bean;
      }
      
      Object beanInstance = doCreateBean(beanName, rootBeanDefinitionToUse, args);
  }
  
  @Nullable
  protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition rootBeanDefinition) {
      Object bean = null;
      if (!Boolean.FALSE.equals(rootBeanDefinition.beforeInstantiationResolved)) {
          if (!rootBeanDefinition.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
              Class<?> targetType = determineTargetType(beanName, rootBeanDefinition);
              if (targetType != null) {
                  // postProcessBeforeInstantiation（注意Instantiation）
                  bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                  if (bean != null) {
                      // 如果bean不为空，不会进行普通bean的创建过程，执行后处理器的 postProcessAfterInitialization 方法。
                      // 尽可能保证所有bean初始化后都会调用注册 BeanPostProcessor 的 postProcessAfterInitialization 方法进行处理。
                      bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                  }
              }
          }
          rootBeanDefinition.beforeInstantiationResolved = (bean != null);
      }
      return bean;
  }
  
  @Nullable
  protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
      for (BeanPostProcessor beanPostProcessor : getBeanPostProcessors()) {
          if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor) {
              // InstantiationAwareBeanPostProcessor
              InstantiationAwareBeanPostProcessor instantiationAwareBeanPostProcessor = (InstantiationAwareBeanPostProcessor) beanPostProcessor;
              
              // 判断是否可获取targetSource，如果可以返回代理对象。
              // TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
              Object result = instantiationAwareBeanPostProcessor.postProcessBeforeInstantiation(beanClass, beanName);
              if (result != null) {
                  return result;
              }
          }
      }
      return null;
  }
  ```

- 循环依赖时提前曝光代理

  为了解决 Bean 循环依赖的问题，Bean 仅实例化还未初始化，但是出现了循环依赖，不得不在此时创建一个代理对象。

  `getEarlyBeanReference()`



### 创建代理对象

找到所有带有 @Aspect 注解的类，并获取其中没有 @Pointcut 注解的方法，循环创建切面，创建切面需要**切点**和**增强**两个元素，其中切点可理解为表达式，增强则是根据 @Before、@Around、@After 等注解创建的对应的 Advice 类。

切面创建后则循环判断哪些切面能对当前的 Bean 实例的方法进行增强并排序，最后通过 ProxyFactory 创建代理对象。

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName, @Nullable Object[] specificInterceptors, TargetSource targetSource) {
    ...
    
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    // 加入增强器
    proxyFactory.addAdvisors(advisors);
    // 设置要代理的类
    proxyFactory.setTargetSource(targetSource);

    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    // 创建代理，具体实现 JdkDynamicAopProxy 或 CglibAopProxy
    return proxyFactory.getProxy(getProxyClassLoader());
}
```

### JdkDynamicAopProxy

JDK 动态代理，代理对象调用方法时会进入到 InvocationHandler 对象的 `invoke()` 方法。

```java
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {
    @Override
    public Object getProxy(@Nullable ClassLoader classLoader) {
        Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
        findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
        // java.lang.reflect.Proxy 生成代理对象。
        return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
    }

    @Override
	@Nullable
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		Object oldProxy = null;
		boolean setProxyContext = false;
		TargetSource targetSource = this.advised.targetSource;
		Object target = null;
		try {
			...
            
			Object returnVal;
			// 目标对象自我调用无法使用切面，通过此属性暴露代理
			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			target = targetSource.getTarget();
			Class<?> targetClass = (target != null ? target.getClass() : null);

			// 获取当前方法的拦截器链
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
			if (chain.isEmpty()) {
				// 没有拦截器直接调用切点方法
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
				returnVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			} else {
				// 将拦截器封装在ReflectiveMethodInvocation
				MethodInvocation invocation =
						new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// 执行拦截器链
				returnVal = invocation.proceed();
			}
		} finally {
		}
	}
}
```

### 拦截器链式调用

通过 `getInterceptorsAndDynamicInterceptionAdvice()`  生成拦截器链，获取到切面 advisor 中的 advice，并且包装成 MethodInterceptor 类型的对象。

```java
MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
```

> 责任链模式。
>
> 通过将 MethodInvocation 对象作为参数传入 `invoke()`，拦截器中执行 `methodInvocation.proceed()`，由 methodInvocation 移动指针到下一个节点。

如果拦截器链不为空，通过 ReflectiveMethodInvocation 类进行链式调用，关键方法就是 `proceed()`。

```java
@Override
@Nullable
public Object proceed() throws Throwable {
    // 拦截器已经到末尾，执行被增强的方法。
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
       return invokeJoinpoint();
    }

    // 获取下一个要执行的拦截器
    // ++this.currentInterceptorIndex移动指针到下一个位置（责任链）
    Object interceptorOrInterceptionAdvice =
          this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
       // 动态匹配
       InterceptorAndDynamicMethodMatcher interceptorAndDynamicMethodMatcher =
             (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
       Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
       if (interceptorAndDynamicMethodMatcher.methodMatcher.matches(this.method, targetClass, this.arguments)) {
          return interceptorAndDynamicMethodMatcher.interceptor.invoke(this);
       } else {
          // 不匹配则不执行拦截器
          return proceed();
       }
    }
    else {
       // 普通拦截器直接调用，如ExposeInvocationInterceptor、AspectJAfterAdvice、MethodBeforeAdviceInterceptor、AspectJAroundAdvice
       return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    }
}
```

以 AspectJAfterAdvice 为例：

```java
public class AspectJAfterAdvice extends AbstractAspectJAdvice implements MethodInterceptor, AfterAdvice, Serializable {
    // 通过深度优先遍历（后序），调用链返回再执行增强方法，实现AOP的@After逻辑。
    @Override
	public Object invoke(MethodInvocation methodInvocation) throws Throwable {
		try {
            // 执行调用链，在methodInvocation通过++this.currentInterceptorIndex移动到下一个拦截器执行invoke
			return methodInvocation.proceed();
		} finally {
			// 执行增强方法
			invokeAdviceMethod(getJoinPointMatch(), null, null);
		}
	}
}
```



## Spring 事务

Spring 事务提供一套抽象的事务管理，结合 Spring IOC 和 Spring AOP，简化了应用程序使用数据库事务，通过声明式事务，可以做到应用程序无侵入的实现事务功能。

### TransactionInterceptor 

TransactionInterceptor 实现了 MethodInterceptor，通过 AOP 拦截器链实现事务功能。

```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {
	@Override
	@Nullable
	public Object invoke(MethodInvocation invocation) throws Throwable {
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
		return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
	}
}

@Nullable
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass, final InvocationCallback invocation) throws Throwable {
    ...

    // 声明式事务处理
    if (transactionAttribute == null || !(platformTransactionManager instanceof CallbackPreferringPlatformTransactionManager)) {
        // 创建事务
        TransactionInfo transactionInfo = createTransactionIfNecessary(platformTransactionManager,  transactionAttribute, joinpointIdentification);

        Object retVal;
        try {
            // 执行被增强方法
            retVal = invocation.proceedWithInvocation();
        } catch (Throwable ex) {
            // 异常回滚
            completeTransactionAfterThrowing(transactionInfo, ex);
            throw ex;
        } finally {
            // 清除信息
            cleanupTransactionInfo(transactionInfo);
        }

        if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
            TransactionStatus status = transactionInfo.getTransactionStatus();
            if (status != null && transactionAttribute != null) {
                retVal = VavrDelegate.evaluateTryFailure(retVal, transactionAttribute, status);
            }
        }

        // 提交事务
        commitTransactionAfterReturning(transactionInfo);
        return retVal;
    }  else {
        Object result;
        final ThrowableHolder throwableHolder = new ThrowableHolder();
        // 编程式事务处理
        ...
        return result;
    }
}
```

### Spring 事务传播

- 保证同一个事务中

  - PROPAGATION_REQUIRED（默认）

    支持当前事务，如果不存在，新建一个事务。

  - PROPAGATION_SUPPORTS

    支持当前事务，如果不存在，不使用事务。

  - PROPAGATION_MANDATORY

    支持当前事务，如果不存在，抛出异常。

- 保证没有在同一个事务中

  - PROPAGATION_REQUIRES_NEW

    如果有事务存在，挂起当前事务，创建一个新的事务。

  - PROPAGATION_NOT_SUPPORTED

    以非事务方式运行，如果有事务存在，挂起当前事务。

  - PROPAGATION_NEVER

    以非事务方式运行，如果有事务存在，抛出异常。

  - PROPAGATION_NESTED

    如果当前事务存在，则嵌套事务执行。

```java
@Override
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
    throws TransactionException {
    TransactionDefinition transactionDefinition = (definition != null ? definition : TransactionDefinition.withDefaults());

    // 获取事务（ThreadLocal获取）
    Object transaction = doGetTransaction();
    boolean debugEnabled = logger.isDebugEnabled();

    // 判断当前线程是否存在事务
    if (isExistingTransaction(transaction)) {
        // 当前线程已经存在事务, 进行嵌套事务处理
        // PROPAGATION_REQUIRES_NEW、PROPAGATION_NESTED...
        return handleExistingTransaction(transactionDefinition, transaction, debugEnabled);
    }

    // 事务超时设置验证
    if (transactionDefinition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", transactionDefinition.getTimeout());
    }

    // PROPAGATION_MANDATORY 不存在事务则抛出异常
    if (transactionDefinition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
            "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    // PROPAGATION_REQUIRED、PROPAGATION_REQUIRES_NEW、PROPAGATION_NESTED 不存在事务则创建事务
    else if (transactionDefinition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
             transactionDefinition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
             transactionDefinition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        SuspendedResourcesHolder suspendedResources = suspend(null);
        try {
            // 开启事务
            return startTransaction(transactionDefinition, transaction, debugEnabled, suspendedResources);
        } catch (RuntimeException | Error ex) {
            resume(null, suspendedResources);
            throw ex;
        }
    } 
    // PROPAGATION_SUPPORTS、PROPAGATION_NOT_SUPPORTS、PROPAGATION_NEVER 不创建事务
    else {
        if (transactionDefinition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
            logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                        "isolation level will effectively be ignored: " + transactionDefinition);
        }
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(transactionDefinition, null, true, newSynchronization, debugEnabled, null);
    }
}


@Nullable
private static Object doGetResource(Object actualKey) {
    // ThreadLocal<Map<Object, Object>> resources
    // 如果线程中存在dataSource的连接，直接使用
    Map<Object, Object> map = resources.get();
    if (map == null) {
        return null;
    }
    Object value = map.get(actualKey);
    if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
        map.remove(actualKey);
        if (map.isEmpty()) {
            resources.remove();
        }
        value = null;
    }
    return value;
}
```

`doGetResource()` 从 TreadLocal 中获取，因此 Spring 事务不能跨线程。



## Spring MVC

![image-20240803172146473](https://gitee.com/fushengshi/image/raw/master/image-20240803172146473.png)

- DispatcherServlet：核心的中央处理器，负责接收请求、分发，并给予客户端响应。

- HandlerMapping：处理器映射器，根据 URL 去匹配查找能处理的 Handler ，并会将请求涉及到的拦截器和 Handler 一起封装。

- HandlerAdapter：处理器适配器，根据 HandlerMapping 找到的 Handler ，适配执行对应的 Handler。

- Handler：请求处理器，处理实际请求的处理器。

- ViewResolver：视图解析器，根据 Handler 返回的逻辑视图 / 视图，解析并渲染真正的视图，并传递给 DispatcherServlet 响应客户端。





# 附录

## Spring 的设计模式

- 单例模式

  Spring 中的 Bean 默认都是单例的。

  ```java
  private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(64);
  ```

- 工厂模式 

  Spring 使用工厂模式通过 BeanFactory、ApplicationContext 创建 Bean 对象。

- 代理模式

  Spring AOP 功能的实现。

- 模板方法模式

  Spring 中 jdbcTemplate、hibernateTemplate 等以 Template 结尾的对数据库操作的类，它们就使用到了模板模式。

- 观察者模式

  Spring 事件驱动模型就是观察者模式很经典的一个应用。

  ApplicationListener 充当了事件监听者角色，它是一个接口，里面只定义了一个 `onApplicationEvent()` 方法来处理ApplicationEvent，在 Spring 中实现 ApplicationListener 接口的 `onApplicationEvent()` 方法即可完成监听事件。

  ```java
  // 定义一个事件，继承自ApplicationEvent并且写相应的构造函数
  public class DemoEvent extends ApplicationEvent {
      private static final long serialVersionUID = 1L;
  
      private String message;
  
      public DemoEvent(Object source,String message){
          super(source);
          this.message = message;
      }
  
      public String getMessage() {
           return message;
      }
  }
  
  // 定义一个事件监听者，实现 ApplicationListener 接口，重写 onApplicationEvent() 方法；
  @Component
  public class DemoListener implements ApplicationListener<DemoEvent>{
      // 使用onApplicationEvent接收消息
      @Override
      public void onApplicationEvent(DemoEvent event) {
          String msg = event.getMessage();
          System.out.println("接收到的信息是："+ msg);
      }
  }
  
  // 发布事件，可以通过 ApplicationEventPublisher 的 publishEvent() 方法发布消息。
  @Component
  public class DemoPublisher {
      @Autowired
      ApplicationContext applicationContext;
  
      public void publish(String message){
          // 发布事件
          applicationContext.publishEvent(new DemoEvent(this, message));
      }
  }
  ```

- 适配器模式

  通过将一个类的接口转换成客户端所期望的另一个接口，使得原本由于接口不兼容而无法一起工作的类能够协同工作。
  
  - Spring AOP 的增强或通知（Advice）使用到了适配器模式。
  
    AspectJAfterReturningAdvice 和 AspectJMethodBeforeAdvice 没有实现 MethodInterceptor 接口，而 Spring Aop 的方法拦截器却必须是实现了 MethodInterceptor 的，所以 Spring 提供了对应的适配器来适配这个问题，分别是 MethodBeforeAdviceAdapter、AfterReturningAdviceAdapter 和 ThrowsAdviceAdapter。
  
  - Spring MVC 中也是用到了适配器模式适配 Controller。
  
    Spring MVC 中的 Controller 种类众多，不同类型的 Controller 通过不同的方法来对请求进行处理。
  
    ```java
    // 对于每种类型的 Controller，提供相应的 HandlerAdapter 实现类。
    public class HttpRequestHandlerAdapter implements HandlerAdapter {
        @Override
        public boolean supports(Object handler) {
            return (handler instanceof HttpRequestHandler);
        }
    
        @Override
        @Nullable
        public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
            ((HttpRequestHandler) handler).handleRequest(request, response);
            return null;
        }
    
        @Override
        @SuppressWarnings("deprecation")
        public long getLastModified(HttpServletRequest request, Object handler) {
            if (handler instanceof LastModified) {
                return ((LastModified) handler).getLastModified(request);
            }
            return -1L;
        }
    }
    
    // 当请求到达时，DispatcherServlet会根据 Handler 的类型找到对应的 HandlerAdapter。这个过程通常通过遍历 HandlerAdapter列表并使用 supports() 方法来确定。
    protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
        if (this.handlerAdapters != null) {
            for (HandlerAdapter adapter : this.handlerAdapters) {
                if (adapter.supports(handler)) {
                    return adapter;
                }
            }
        }
        throw new ServletException("");
    }
    ```
  
- 装饰器模式

  装饰者模式可以动态地给对象添加一些额外的属性或行为。

  项目需要连接多个数据库，而且不同的客户在每次访问中根据需要会去访问不同的数据库。这种模式可以根据客户的需求能够动态切换不同的数据源。
  
- 责任链模式



## InitializingBean

Bean 实现了 InitializingBean 接口时，Spring 容器会在所有必需的属性（也就是依赖注入完成后）被填充后调用 afterPropertiesSet() 方法。这个方法主要用于执行一些初始化操作，比如初始化数据、打开资源连接等。

执行时机：

- Bean 实例化

  Spring 首先通过反射创建 Bean 实例。

- 依赖注入

  Spring 会根据配置完成对该 Bean 依赖的其他 Bean 的注入。

- Aware接口回调

  如果 Bean 实现了如 BeanNameAware、ApplicationContextAware 等 Aware 接口，Spring 会依次调用这些接口的方法，传递相应的信息给 Bean。

- @PostConstruct 注解的方法调用

  如果Bean中有标注了 @PostConstruct 的方法，接下来会调用这些方法。

- `afterPropertiesSet()` 调用

  如果 Bean 实现了 InitializingBean 接口，Spring 容器会调用 `afterPropertiesSet()` 方法。

- 自定义初始化方法

  如果在配置文件或 @Bean 注解中指定了 init-method 属性，Spring 也会在 `afterPropertiesSet()` 之后调用指定的初始化方法。



## 自定义 FactoryBean

FactoryBean 接口是 Spring IOC 容器 Bean 实例化逻辑的一个扩展点，如果有复杂的初始化 Bean 逻辑，则可以选择创建自定义FactoryBean，在该类中编写初始化逻辑，然后把自定义 FactoryBean 注入到容器中。

> 第三方框架可以写一个 FactoryBean 类，提供一个简单的入口和 Spring 框架进行整合。
>
> 如 Mybatis 注入代理对象，Dubbo 注入代理对象。

```java
protected <T> T doGetBean(
    String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
    throws BeansException {

    String beanName = transformedBeanName(name);
    Object bean;

    // 直接尝试从一级缓存 singletonObjects 或者三级缓存 singletonFactories 获取
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // 处理FactoryBean的方法
        // 当bean本身创建完成之后，就会调用这个方法，如果缓存中没有会创建对象实例，创建完依然会调用这个方法。
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }
    ...
}

protected Object getObjectForBeanInstance(
    Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition rootBeanDefinition) {
    ...
    if (!(beanInstance instanceof FactoryBean)) {
        return beanInstance;
    }
    ...
    if (object == null) {
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
        if (rootBeanDefinition == null && containsBeanDefinition(beanName)) {
            rootBeanDefinition = getMergedLocalBeanDefinition(beanName);
        }
        boolean synthetic = (rootBeanDefinition != null && rootBeanDefinition.isSynthetic());
        // 处理FactoryBean
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}

private Object doGetObjectFromFactoryBean(FactoryBean<?> factory, String beanName) throws BeanCreationException {
    Object object;
    try {
        if (System.getSecurityManager() != null) {
            AccessControlContext acc = getAccessControlContext();
            try {
                object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
            } catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        } else {
            // 直接调用 getObject() 方法
            object = factory.getObject();
        }
    }
    ...
}
```






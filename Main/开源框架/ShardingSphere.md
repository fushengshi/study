# ShardingSphere

ShardingSphere 的定位是一种关系型数据库中间件。

![image-20240817201847065](https://gitee.com/fushengshi/image/raw/master/image-20240817201847065.png)

- Sharding-JDBC

  Sharding-JDBC 的定位是一个轻量级 Java 框架，在 JDBC 层提供了扩展性服务。Sharding-JDBC 以 JAR 包的形式提供服务。开发人员可以使用这个 JAR 包直连数据库，无需额外的部署和依赖管理。

  由于 Sharding-JDBC 提供了一套与 JDBC 规范完全一致的 API，所以它可以很方便地与遵循 JDBC 规范的各种组件和框架进行无缝集成。例如，用于提供数据库连接的 DBCP、C3P0、Durid 等数据库连接池组件，以及用于提供对象-关系映射的 Hibernate、MyBatis 等 ORM 框架。作为一款支持多数据库的开源框架，Sharding-JDBC 支持 MySQL、Oracle、SQLServer 等主流关系型数据库。

- Sharding-Proxy

  ShardingSphere 中的 Sharding-Proxy 组件定位为一个透明化的数据库代理端，所以它是代理服务器分片方案的一种具体实现方式。

  可以直接把 Sharding-Proxy 视为一个数据库，用来代理后面分库分表的多个数据库，它屏蔽了后端多个数据库的复杂性。

- Sharding-Sidecar

  Sidecar 设计模式受到了越来越多的关注和采用，这个模式的目标是把系统中各种异构的服务组件串联起来，并进行高效的服务治理。

ShardingSphere 的整体功能拆分成四大部分，基础设施、分片引擎、分布式事务和治理与集成。

- 基础设施

  ShardingSphere 在架构上也提供了很多基础设施类的组件，这些组件更多与它的内部实现机制有关。

  - 微内核架构

    ShardingSphere 在设计上采用了微内核（MicroKernel）架构模式，来确保系统具有高度可扩展性。

  - 分布式主键

    ShardingSphere 提供了分布式主键的实现机制，默认采用的是 SnowFlake（雪花）算法。

- 分片引擎

  - 数据分片

    数据分片是 ShardingSphere 的核心功能，基于垂直拆分和水平拆分的分库分表操作都支持。ShardingSphere 预留了分片扩展点，开发人员也可以基于需要实现分片策略的定制化开发。

  - 读写分离

    在分库分表的基础上，ShardingSphere 也实现了基于数据库主从架构的读写分离机制。

- 分布式事务

  - 标准化事务处理接口

    ShardingSphere 抽象了一组标准化的事务处理接口，并通过分片事务管理器 ShardingTransactionManager 进行统一管理。

  - 强一致性事务与柔性事务

    ShardingSphere 内置了一组分布式事务的实现方案。

    强一致性事务内置集成了 Atomikos、Narayana 和 Bitronix 等技术来实现 XA 事务管理器。

    ShardingSphere 内部整合了 Seata 来提供柔性事务功能。

- 治理与集成

  对于分布式数据库而言，治理的范畴可以很广，ShardingSphere 也提供了注册中心、配置中心等一系列功能来支持数据库治理。另一方面，ShardingSphere 作为一款支持快速开发的开源框架，也完成了与其他主流框架的无缝集成。

  - 数据脱敏

    ShardingSphere 将数据脱敏机制内嵌到了 SQL 的执行过程中，只要通过简单的配置就能实现数据的自动脱敏。

  - 配置中心

    ShardingSphere 可以基于 YAML 格式或 XML 格式的配置文件完成配置信息的维护。还提供了配置信息动态化的管理机制，可以支持数据源、表与分片及读写分离策略的动态切换。

  - 注册中心

    ShardingSphere 中的注册中心提供了基于 Nacos 和 ZooKeeper 的两种实现方式。

    在应用场景上可以基于注册中心完成数据库实例管理、数据库熔断禁用等治理功能。

  - 链路跟踪

    SQL 解析与 SQL 执行是数据分片的最核心步骤，ShardingSphere 会将运行时的数据通过标准协议提交到链路跟踪系统。ShardingSphere 使用 OpenTracing API 发送性能追踪数据，SkyWalking、Zipkin 和 Jaeger 等面向 OpenTracing 协议的具体产品都可以和 ShardingSphere 自动完成对接。

  - 系统集成

    ShardingSphere 和 Spring 系列框架的集成。







# 附录

## JDBC 规范

![image-20240817202847574](https://gitee.com/fushengshi/image/raw/master/image-20240817202847574.png)

JDBC（Java Database Connectivity）的设计初衷是提供一套用于各种数据库的统一标准，而不同的数据库厂家共同遵守这套标准，并提供各自的实现方案供应用程序调用。

JDBC 架构中的 Driver Manager 负责加载各种不同的驱动程序（Driver），并根据不同的请求，向调用者返回相应的数据库连接（Connection）。JDBC API 是访问数据库的主要途径，也是 ShardingSphere 重写 JDBC 规范并添加分片功能的入口。

```java
// 创建池化的数据源
PooledDataSource dataSource = new PooledDataSource ();
// 设置MySQL Driver
dataSource.setDriver ("com.mysql.jdbc.Driver");
// 设置数据库URL、用户名和密码
dataSource.setUrl ("jdbc:mysql://localhost:3306/xxx");
dataSource.setUsername ("root");
dataSource.setPassword ("pwd");
// 获取连接
Connection connection = dataSource.getConnection();
// 执行查询
PreparedStatement statement = connection.prepareStatement ("select * from user");
// 获取查询结果应该处理
ResultSet resultSet = statement.executeQuery();
while (resultSet.next()) {
	...
}
// 关闭资源
statement.close();
resultSet.close();
connection.close();
```

- DataSource

  DataSource 在 JDBC 规范中代表的是一种数据源，核心作用是获取数据库连接对象 Connection。在 JDBC 规范中，可以直接通过 DriverManager 获取 Connection，获取 Connection 的过程需要建立与数据库之间的连接，会产生较大的系统开销。

  为了提高性能，通常会建立一个中间层，该中间层将 DriverManager 生成的 Connection 存放到连接池中，然后从池中获取 Connection。在 ShardingSphere 中，暴露给业务开发人员的同样是一个经过增强的 DataSource 对象。

- Connection

  DataSource 的目的是获取 Connection 对象，可以把 Connection 理解为一种会话（Session）机制。Connection 代表一个数据库连接，负责完成与数据库之间的通信。

- Statement 

  JDBC 规范中的 Statement 存在两种类型，一种是普通的 Statement，一种是支持预编译的 PreparedStatement。

  > 预编译是指数据库的编译器会对 SQL 语句提前编译，然后将预编译的结果缓存到数据库中，这样下次执行时就可以替换参数并直接使用编译过的语句，从而提高 SQL 的执行效率。

- ResultSet

  通过 Statement 或 PreparedStatement 执行了 SQL 语句并获得了 ResultSet 对象后，那么就可以通过调用 Resulset 对象中的 `next()` 方法遍历整个结果集。



### 基于适配器模式的 JDBC 重写

适配器模式通常被用作连接两个不兼容接口之间的桥梁，涉及为某一个接口加入独立的或不兼容的功能。作为一套适配 JDBC 规范的实现方案，ShardingSphere 需要对 JDBC API 中的 DataSource、Connection、Statement 及 ResultSet 等核心对象都完成重写。

- ShardingConnection

  ShardingConnection 是对 JDBC 中 Connection 的适配和包装，所以它需要提供 Connection 接口中定义的方法，包括 createConnection、getMetaData、各种重载的 prepareStatement 和 createStatement 以及针对事务的 setAutoCommit、commit 和 rollback 方法等。

  ```java
  // dataSource为真实数据源
  private Connection createConnection(final String dataSourceName, final DataSource dataSource) throws SQLException {
      Connection result = isInShardingTransaction() ? 
          shardingTransactionManager.getConnection(dataSourceName) : dataSource.getConnection();
      // 根据方法和参数通过反射技术进行执行
      replayMethodsInvocation(result);
      return result;
  }
  ```



## 数据库连接池

ShardingSphere 支持一批主流的第三方数据库连接池，包括 DBCP、C3P0、BoneCP、Druid 和 HikariCP 等。在应用 ShardingSphere 时，可以通过创建 DataSource 来使用数据库连接池。

在 Spring Boot 中，可以在 .properties 配置文件中使用 DruidDataSource 类，初始化基于 Druid 数据库连接池的 DataSource：

```properties
spring.shardingsphere.datasource.names=test_datasource
spring.shardingsphere.datasource.test_datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.test_datasource.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.test_datasource.jdbc-url=jdbc:mysql://localhost:3306/test_database
spring.shardingsphere.datasource.test_datasource.username=root
spring.shardingsphere.datasource.test_datasource.password=root
```



## 分库分表

为了解决由于数据量过大而导致的数据库性能降低的问题，将原来独立的数据库拆分成若干数据库，把原来数据量大的表拆分成若干数据表，使得单一数据库、单一数据表的数据量变得足够小，从而达到提升数据库性能的效果。

分库分表包括分库和分表两个维度，在开发过程中，对于每个维度都可以采用两种拆分思路，即**垂直拆分**和水平拆分。

- 垂直分库

  按照业务将表进行分类，然后分布到不同的数据库上。每个库可以位于不同的服务器上，其核心理念是专库专用。

  从实现上讲，垂直分库很大程度上取决于业务的规划和系统边界的划分。比如用户数据的独立拆分就需要考虑到系统用户体系与其他业务模块之间的关联关系，而不是简单地创建一个用户库。

- 垂直分表

  将一个表按照字段分成多张表，每个表存储其中一部分字段。

- 水平分库

  水平分库是把同一个表的数据按一定规则拆分到不同的数据库中，每个库同样可以位于不同的服务器上。

  - 取模算法

    取模的方式有很多，比如按照用户 ID 进行取模，当然也可以通过表的一列或多列字段进行 hash 求值来取模。

  - 范围限定算法

    比如可以采用按年份、按时间等策略路由到目标数据库或表。

  - 预定义算法

    事先规划好具体库或表的数量，然后直接路由到指定库或表中。

- 水平分表

  水平分表是在同一个数据库内，把同一个表的数据按一定规则拆到多个表中。

### 代码实现

```java
public DataSource createDataSource() throws SQLException {
    Map<String, DataSource> dataSourceMap = new HashMap<>();
    dataSourceMap.put("ds0", createDataSource("ds0"));
    dataSourceMap.put("ds1", createDataSource("ds1"));

    // 创建分片规则配置类
    ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();

    // 分库策略：根据性别分库，一共分为2个库
    shardingRuleConfig.setDefaultDatabaseShardingStrategyConfig(
        new InlineShardingStrategyConfiguration("sex", "ds${sex % 2}"));
    // 分表策略
    shardingRuleConfig.setDefaultTableShardingStrategyConfig(new NoneShardingStrategyConfiguration());

    Properties properties = new Properties();
    properties.put("sql-show", "false");
    // 通过工厂类创建具体的 DataSource
    return ShardingDataSourceFactory.createDataSource(dataSourceMap, shardingRuleConfig, properties);
}

public DataSource createDataSource(final String dataSourceName) {
    // Druid连接池
    DruidDataSource result = new DruidDataSource();
    result.setDriverClassName(com.mysql.jdbc.Driver.class.getName());
    result.setUrl(String.format("jdbc:mysql://%s:%s/%s", "localhost", 3306, dataSourceName));
    result.setUsername("root");
    result.setPassword("pwd");
    return result;
}
```



## 分片（Sharding）

无论是分库还是分表，都是把数据划分成不同的数据片，并存储在不同的目标对象中。而具体的分片方式涉及实现分库分表的不同解决方案。

- 客户端分片

  客户端分片相当于在数据库的客户端就实现了分片规则。最为简单的方式就是应用层分片，在应用程序中直接维护着分片规则和分片逻辑。

  客户端分片在实现上通常会进一步抽象，把分片规则的管理工作从业务代码中剥离出来，形成单独演进的一套体系。这方面典型的设计思路是重写 JDBC 协议，在 JDBC 协议层面嵌入分片规则。

  > 客户端分片典型的中间件包括阿里巴巴的 TDDL 以及 ShardingSphere。

- 代理服务器分片

  在应用层和数据库层之间添加一个代理层，把分片规则集中维护在这个代理层中，对外提供与 JDBC 兼容的 API 给应用层。

  > 常见的开源框架有阿里的 Cobar 以及开源社区的 MyCat。在 ShardingSphere 3.X 版本中也添加了 Sharding-Proxy 模块来实现代理服务器分片。

- 分布式数据库

  分布式数据库中数据分片及分布式事务将是其内置的基础功能，开发人员只需要使用框架对外提供的 JDBC 接口。

  > 分布式数据库代表有 TiDB。



实际应用

```java
protected ShardingSphereDataSource makeShardingDataSource(ShardDBConfig shardDBConfig,
    RoutingDataSource tenantDefaultDataSource,
    List<DivisionDataSourcePackage> divisionDataSourcePackages) throws SQLException {
    // 这里用linked hash map，确保没使用分库分表时，首先路由到默认数据源
    Map<String, DataSource> dataSources = new LinkedHashMap<>();
    // 所有的规则集合
    List<RuleConfiguration> ruleConfigurations = new ArrayList<>();

    // 1.设置shardRuleConfig策略
    ShardingRuleConfiguration shardingRuleConfig = buildShardingRuleConfig();
    ruleConfigurations.add(shardingRuleConfig);

    // 2.生成租户主库的数据源
    String defaultDataSourceName = buildTenantDefaultDataSource(shardDBConfig, tenantDefaultDataSource,
        dataSources);
    shardingRuleConfig.setDefaultDataSourceName(defaultDataSourceName);
    // 3.设置分表分库数据源，暂时无需求

    // 4.设置租户分库的数据源
    Map<String, TenantDivisionDB> tenantDivisionDBMap = getStringTenantDivisionDBMap(divisionDataSourcePackages,
        dataSources);

    // 构建sharding table rule
    // 根据租户主库和分库配置创建 tableRules
    List<ShardingTableRuleConfiguration> tableRules = buildShardingTableRuleConfigurations(
        shardingRuleConfig.getDefaultDataSourceName(), tenantDivisionDBMap, shardDBConfig.getTenantId());

    shardingRuleConfig.setTables(tableRules);

    Properties properties = new Properties();
    properties.put("sql-show", "false");
    properties.put("max-connections-size-per-query", 5);
    properties.put("load-table-metadata-enabled", false);
    
    // 创建数据源
    return (ShardingSphereDataSource) ShardingSphereDataSourceFactory.createDataSource(dataSources,
        Collections.singleton(shardingRuleConfig),
        properties);
}
```









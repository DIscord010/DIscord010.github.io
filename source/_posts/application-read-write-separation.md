---
title: 数据库读写分离实现方案
date: 2020-02-13 14:06:15
tags: [SpringBoot, 读写分离]
categories: 后端开发
---

数据库主从集群搭建完毕后，我们就可以进行数据库读写分离。数据库读写分离可以在应用层自行实现，也可以借助中间件。中间件的实现又分为两类，一种是以JAR的形式提供服务，应用导入之后，进行配置使用。而另一种则是需要单独部署，应用连接数据库一样连接中间件服务即可。

# 静态多数据源

这种方式就是简单地配置多数据源进行数据库读写分离。实现起来比较简单，配置两个数据源，一个进行读相关操作，另一个进行写相关操作即可。备库的数据源配置提供的账号需要限制为只读账号。

SpringBoot配置多数据源也比较简单，这里整合Mybatis进行实现：

```
# 主库配置
spring.datasource.master.jdbc-url=jdbc:mysql://123.207.89.169:3306/max?\
  serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
spring.datasource.master.username=root
spring.datasource.master.password=******
# 备库配置
spring.datasource.slave.jdbc-url=jdbc:mysql://123.207.89.169:3307/max?\
  serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
spring.datasource.slave.username=root
spring.datasource.slave.password=******
```

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.master")
    DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slave")
    DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

```java
@Configuration
@MapperScan(basePackages = "club.csiqu.mysqlmasterslave.dao.master",
        sqlSessionFactoryRef = "masterSqlSessionFactory")
public class MasterMybatisConfig {

    @Bean(name = "masterSqlSessionFactory")
    public SqlSessionFactory masterSqlSessionFactory(@Qualifier("masterDataSource") DataSource datasource)
            throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(datasource);
        // 配置 Mybatis mapper文件的配置
        bean.setMapperLocations(
                new PathMatchingResourcePatternResolver()
                        .getResources("classpath*:mybatis/mapper/master/*.xml"));
        return bean.getObject();
    }

    @Primary
    @Bean("masterSqlSessionTemplate")
    public SqlSessionTemplate masterSqlSessionTemplate(
            @Qualifier("masterSqlSessionFactory") SqlSessionFactory sessionfactory) {
        return new SqlSessionTemplate(sessionfactory);
    }
}
```

```java
@Configuration
@MapperScan(basePackages = "club.csiqu.mysqlmasterslave.dao.slave",
        sqlSessionFactoryRef = "slaveSqlSessionFactory")
public class SlaveMybatisConfig {

    @Bean(name = "slaveSqlSessionFactory")
    public SqlSessionFactory slaveSqlSessionFactory(@Qualifier("slaveDataSource") DataSource datasource)
            throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(datasource);
        // 配置 Mybatis mapper文件的配置
        bean.setMapperLocations(
                new PathMatchingResourcePatternResolver()
                        .getResources("classpath*:mybatis/mapper/slave/*.xml"));
        return bean.getObject();
    }

    @Bean("slaveSqlSessionTemplate")
    public SqlSessionTemplate slaveSqlSessionTemplate(
            @Qualifier("slaveSqlSessionFactory") SqlSessionFactory sessionfactory) {
        return new SqlSessionTemplate(sessionfactory);
    }
}
```

基本不会采用通过配置多数据源的方式来实现数据库读写分离。实际应用场景为应用确实使用多个不同的数据源。

# 多数据源动态切换

我们可以基于`AbstractRoutingDataSource`实现动态数据源切换来完成对数据库的读写分离。

## 原理

我们查看`AbstractRoutingDataSource`的源码：

```java
public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean {

	@Nullable
	private Map<Object, Object> targetDataSources;

	@Nullable
	private Object defaultTargetDataSource;

	private boolean lenientFallback = true;

	private DataSourceLookup dataSourceLookup = new JndiDataSourceLookup();

	@Nullable
	private Map<Object, DataSource> resolvedDataSources;

	@Nullable
	private DataSource resolvedDefaultDataSource;
```

就可以发现其实现`DataSource`接口，且内部通过一个Map来维护多个数据源。

```java
	@Override
	public Connection getConnection() throws SQLException {
		return determineTargetDataSource().getConnection();
	}
```

通过模板设计模式，让子类实现获取key的逻辑：

```java
	protected DataSource determineTargetDataSource() {
		Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
		// 这个方法是抽象方法，由子类实现具体逻辑获取 key
        Object lookupKey = determineCurrentLookupKey();
		DataSource dataSource = this.resolvedDataSources.get(lookupKey);
		if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
			dataSource = this.resolvedDefaultDataSource;
		}
		if (dataSource == null) {
			throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
		}
		return dataSource;
	}
```

明白原理之后，我们就可以很轻松的通过继承这个类来实现数据源的切换。

## 实现

使用枚举作为key：

```java
public enum DataSourceRoutingKey {

    /**
     * 写操作
     */
    WRITE,

    /**
     * 读操作
     */
    READ
}
```

通过线程本地变量来维护切换key：

```java
public class DataSourceContextHolder {

    private static final Logger LOGGER = LoggerFactory.getLogger(DataSourceContextHolder.class);

    private static final ThreadLocal<DataSourceRoutingKey> ROUTING_KEY = new ThreadLocal<>();

    public static void setDataSourceRoutingKey(DataSourceRoutingKey key) {
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("修改数据源路由标识为：{}", key);
        }
        ROUTING_KEY.set(key);
    }

    public static DataSourceRoutingKey getDataSourceRoutingKey() {
        return ROUTING_KEY.get();
    }

    public static void clearDataSourceRoutingKey() {
        ROUTING_KEY.remove();
    }
}
```

编写类继承`AbstractRoutingDataSource`类实现获取key的逻辑：

```java
public class RoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceContextHolder.getDataSourceRoutingKey();
    }
}
```

配置多个数据源，设置`routingDataSource`的`targetDataSource`属性：

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.master")
    DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slave1")
    DataSource slave1DataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "routingDataSource")
    DataSource dataSource(@Qualifier("masterDataSource") DataSource masterDataSource,
                          @Qualifier("slave1DataSource") DataSource slave1DataSource) {
        Map<Object, Object> targetDataSource = new HashMap<>(8);
        targetDataSource.put(DataSourceRoutingKey.WRITE, masterDataSource);
        targetDataSource.put(DataSourceRoutingKey.READ, slave1DataSource);
        AbstractRoutingDataSource dataSource = new RoutingDataSource();
        // 设置 targetDataSource
        dataSource.setTargetDataSources(targetDataSource);
        dataSource.setDefaultTargetDataSource(masterDataSource);
        return dataSource;
    }
}
```

将我们注册的`routingDataSource`作为`sqlSessionFactory`的数据源：

```java
@Configuration
@MapperScan(basePackages = "club.csiqu.mysqlmasterslave.dao",
        sqlSessionFactoryRef = "sqlSessionFactory")
public class MybatisConfig {

    @Bean(name = "sqlSessionFactory")
    public SqlSessionFactory masterSqlSessionFactory(@Qualifier("routingDataSource") DataSource datasource)
            throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(datasource);
        // 配置 Mybatis mapper文件的配置
        bean.setMapperLocations(
                new PathMatchingResourcePatternResolver()
                        .getResources("classpath*:mybatis/mapper/*.xml"));
        return bean.getObject();
    }

    @Bean("sqlSessionTemplate")
    public SqlSessionTemplate masterSqlSessionTemplate(
            @Qualifier("sqlSessionFactory") SqlSessionFactory sessionfactory) {
        return new SqlSessionTemplate(sessionfactory);
    }
}
```

通过自定义注解和切面编程实现根据注解切换使用的数据源：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
@Documented
public @interface DataSourceRouting {

    DataSourceRoutingKey value() default DataSourceRoutingKey.WRITE;
}
```

```java
@Aspect
@Component
public class DataSourceRoutingAspect {

    @Before("@annotation(DataSourceRouting)")
    public void before(JoinPoint point) {
        // 获取切点方法
        Method method = ((MethodSignature)point.getSignature()).getMethod();
        DataSourceRouting annotation = method.getAnnotation(DataSourceRouting.class);
        DataSourceContextHolder.setDataSourceRoutingKey(annotation.value());
    }

    @After("@annotation(DataSourceRouting)")
    public void after() {
        DataSourceContextHolder.clearDataSourceRoutingKey();
    }
}
```

在方法上使用我们的注解，并进行测试：

```java
@Service
public class UserServiceImpl implements UserService {

    final UserMapper userMapper;

    public UserServiceImpl(UserMapper userMapper) {this.userMapper = userMapper;}

    @Override
    @DataSourceRouting
    public void insert(User user) {
        userMapper.insert(user);
    }

    @Override
    @DataSourceRouting(DataSourceRoutingKey.READ)
    public User getByUserName(String userName) {
        return userMapper.getByUserName(userName);
    }
}
```

```java
class UserServiceTest extends MysqlMasterSlaveApplicationTests {

    @Autowired
    UserService userService;

    @Test
    void insert() {
        User user = new User();
        user.setUserName("chen");
        user.setPassword("123456");
        userService.insert(user);
    }

    @Test
    void getByUserName() {
        Assertions.assertEquals(userService.getByUserName("chen").getPassword(), "123456");
    }
}
```

以上是一主一从的实现。

## 多从库

在多个从库的情况，我们稍微修改一下代码。

在类`RoutingDataSource`中维护一个读库的key的列表，并使用随机的负载均衡方式获取读库的key：

```java
public class RoutingDataSource extends AbstractRoutingDataSource {

    private List<String> slaveDataSourceKeys;

    @Override
    protected Object determineCurrentLookupKey() {
        DataSourceRoutingKey dataSourceRoutingKey = DataSourceContextHolder.getDataSourceRoutingKey();
        if (dataSourceRoutingKey.equals(DataSourceRoutingKey.WRITE)) {
            return dataSourceRoutingKey;
        }
        return getReadKeyBalancing();
    }

    private String getReadKeyBalancing() {
        Random random = new Random();
        // 随机在列表中获取从库的 key
        return slaveDataSourceKeys.get(random.nextInt(slaveDataSourceKeys.size()));
    }
    
    public void setSlaveDataSourceKeys(List<String> slaveDataSourceKeys) {
        this.slaveDataSourceKeys = slaveDataSourceKeys;
    }
}
```

修改配置类设置`slaveDataSourceKeys`（需要在配置文件中增加一个数据源的配置）：

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.master")
    DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slave1")
    DataSource slave1DataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slave2")
    DataSource slave2DataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "routingDataSource")
    DataSource dataSource(@Qualifier("masterDataSource") DataSource masterDataSource,
                          @Qualifier("slave1DataSource") DataSource slave1DataSource,
                          @Qualifier("slave2DataSource") DataSource slave2DataSource) {
        Map<Object, Object> targetDataSource = new HashMap<>(8);
        targetDataSource.put(DataSourceRoutingKey.WRITE, masterDataSource);
        targetDataSource.put("slave1DataSource", slave1DataSource);
        targetDataSource.put("slave2DataSource", slave2DataSource);
        List<String> slaveKeys = new ArrayList<>(2);
        slaveKeys.add("slave1DataSource");
        slaveKeys.add("slave2DataSource");
        RoutingDataSource dataSource = new RoutingDataSource();
        dataSource.setTargetDataSources(targetDataSource);
        // 设置从库的 key列表
        dataSource.setSlaveDataSourceKeys(slaveKeys);
        dataSource.setDefaultTargetDataSource(masterDataSource);
        return dataSource;
    }
}
```

运行结果：

```java
2020-02-21 16:46:59.783  INFO 12932 --- [           main] c.c.m.c.routing.DataSourceContextHolder  : 修改数据源路由标识为：WRITE
2020-02-21 16:46:59.807  INFO 12932 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2020-02-21 16:47:00.180  INFO 12932 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2020-02-21 16:47:00.252  INFO 12932 --- [           main] c.c.m.c.routing.DataSourceContextHolder  : 修改数据源路由标识为：READ
2020-02-21 16:47:00.254  INFO 12932 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-2 - Starting...
2020-02-21 16:47:00.442  INFO 12932 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-2 - Start completed.
2020-02-21 16:47:00.476  INFO 12932 --- [           main] c.c.m.c.routing.DataSourceContextHolder  : 修改数据源路由标识为：READ
2020-02-21 16:47:00.477  INFO 12932 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-3 - Starting...
2020-02-21 16:47:00.689  INFO 12932 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-3 - Start completed.
2020-02-21 16:47:00.709  INFO 12932 --- [           main] c.c.m.c.routing.DataSourceContextHolder  : 修改数据源路由标识为：READ
2020-02-21 16:47:00.728  INFO 12932 --- [           main] c.c.m.c.routing.DataSourceContextHolder  : 修改数据源路由标识为：READ
```

我们可以看到三个数据源被初始化和使用。

# ShardingSphere

使用ShardingSphere实现对数据库的读写分离。

## Sharding-JDBC

以JAR的形式提供服务。

Maven依赖：

```xml
        <dependency>
            <groupId>org.apache.shardingsphere</groupId>
            <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
            <version>4.0.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.shardingsphere</groupId>
            <artifactId>sharding-jdbc-spring-namespace</artifactId>
            <version>4.0.0</version>
        </dependency>
```

配置文件如下：

```
spring.shardingsphere.datasource.names=master,slave0
# 主库配置
spring.shardingsphere.datasource.master.type=com.zaxxer.hikari.HikariDataSource
spring.shardingsphere.datasource.master.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.master.jdbc-url=jdbc:mysql://123.207.89.169:3306/max?\
  serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
spring.shardingsphere.datasource.master.username=root
spring.shardingsphere.datasource.master.password=******
# 备库配置
spring.shardingsphere.datasource.slave0.type=com.zaxxer.hikari.HikariDataSource
spring.shardingsphere.datasource.slave0.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.slave0.jdbc-url=jdbc:mysql://123.207.89.169:3307/max?\
  serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
spring.shardingsphere.datasource.slave0.username=root
spring.shardingsphere.datasource.slave0.password=******

spring.shardingsphere.masterslave.name=ms
spring.shardingsphere.masterslave.master-data-source-name=master
spring.shardingsphere.masterslave.slave-data-source-names=slave0
spring.shardingsphere.props.sql.show=true
```

Mybatis配置类：

```java
@Configuration
@MapperScan(basePackages = "club.csiqu.mysqlmasterslave.dao",
        sqlSessionFactoryRef = "sqlSessionFactory")
public class MybatisConfig {

    @Bean(name = "sqlSessionFactory")
    @Resource
    public SqlSessionFactory masterSqlSessionFactory(DataSource dataSource)
            throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        // 配置 Mybatis mapper文件的配置
        bean.setMapperLocations(
                new PathMatchingResourcePatternResolver()
                        .getResources("classpath*:mybatis/mapper/*.xml"));
        return bean.getObject();
    }

    @Bean("sqlSessionTemplate")
    public SqlSessionTemplate masterSqlSessionTemplate(
            @Qualifier("sqlSessionFactory") SqlSessionFactory sessionfactory) {
        return new SqlSessionTemplate(sessionfactory);
    }
}
```

![](image-20200224145939402.png)

可以看出`dataSource`是类`MasterSlaveDataSource`的实例对象。这种方式实际上就是实现`java.sql.Connection`接口，对其进行扩展。

开启打印语句功能后，可以看到路由模式和相关的SQL信息：

```verilog
2020-02-24 15:14:03.681  INFO 5736 --- [           main] c.c.m.service.UserServiceTest            : Started UserServiceTest in 27.679 seconds (JVM running for 28.92)

2020-02-24 15:14:04.511  INFO 5736 --- [           main] ShardingSphere-SQL                       : Rule Type: master-slave
2020-02-24 15:14:04.511  INFO 5736 --- [           main] ShardingSphere-SQL                       : SQL: select *
        from t_user
        where username = ? ::: DataSources: slave0
```

## Sharding-Proxy

这种方式下，我们需要下载Sharding-Proxy修改配置后运行，而应用连接MySQL一样连接Sharding-Proxy即可。

下载后，修改配置文件*config-master_slave.yaml*进行读写分离的配置：

```yaml
schemaName: master_slave_db

dataSources:
  ds_master:
    url: jdbc:jdbc:mysql://123.207.89.169:3306/max?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
    username: root
    password: ******
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 65
  ds_slave0:
    url: jdbc:mysql://123.207.89.169:3307/max?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
    username: root
    password: ******
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 65

masterSlaveRule:
  name: ds_ms
  masterDataSourceName: ds_master
  slaveDataSourceNames: 
    - ds_slave0
```

配置*server.yaml*进行认证和属性配置：

```
authentication:
  users:
    root:
      password: root
    sharding:
      password: sharding 
      authorizedSchemas: sharding_db
      
props:
  sql.show: true
```

启动：

```verilog
[INFO ] 16:17:11.034 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.default_max_wait_time_on_shutdown = 9223372036854775807
[INFO ] 16:17:11.035 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.allow_subtransactions = true
[INFO ] 16:17:11.036 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.recovery_delay = 300000
[INFO ] 16:17:11.036 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.automatic_resource_registration = false
[INFO ] 16:17:11.037 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.oltp_max_retries = 5
[INFO ] 16:17:11.037 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.client_demarcation = false
[INFO ] 16:17:11.038 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.threaded_2pc = false
[INFO ] 16:17:11.038 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.serial_jta_transactions = false
[INFO ] 16:17:11.039 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.log_base_dir = ./logs
[INFO ] 16:17:11.039 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.rmi_export_class = none
[INFO ] 16:17:11.039 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.max_actives = 10000
[INFO ] 16:17:11.040 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.checkpoint_interval = 50000
[INFO ] 16:17:11.040 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.enable_logging = true
[INFO ] 16:17:11.040 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.log_base_name = xa_tx
[INFO ] 16:17:11.041 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.max_timeout = 300000
[INFO ] 16:17:11.041 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.trust_client_tm = false
[INFO ] 16:17:11.042 [main] c.a.icatch.provider.imp.AssemblerImp - USING: java.naming.factory.initial = com.sun.jndi.rmi.registry.RegistryContextFactory
[INFO ] 16:17:11.042 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.tm_unique_name = 10.0.75.1.tm
[INFO ] 16:17:11.043 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.forget_orphaned_log_entries_delay = 86400000
[INFO ] 16:17:11.043 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.oltp_retry_interval = 10000
[INFO ] 16:17:11.044 [main] c.a.icatch.provider.imp.AssemblerImp - USING: java.naming.provider.url = rmi://localhost:1099
[INFO ] 16:17:11.045 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.force_shutdown_on_vm_exit = false
[INFO ] 16:17:11.045 [main] c.a.icatch.provider.imp.AssemblerImp - USING: com.atomikos.icatch.default_jta_timeout = 300000
[INFO ] 16:17:11.047 [main] c.a.icatch.provider.imp.AssemblerImp - Using default (local) logging and recovery...
[INFO ] 16:17:11.247 [main] c.a.d.xa.XATransactionalResource - resource-1-ds_master: refreshed XAResource
[INFO ] 16:17:11.433 [main] c.a.d.xa.XATransactionalResource - resource-2-ds_slave0: refreshed XAResource
[INFO ] 16:17:12.957 [nioEventLoopGroup-2-1] i.n.handler.logging.LoggingHandler - [id: 0x1d357822] REGISTERED
[INFO ] 16:17:12.959 [nioEventLoopGroup-2-1] i.n.handler.logging.LoggingHandler - [id: 0x1d357822] BIND: 0.0.0.0/0.0.0.0:3307
[INFO ] 16:17:12.960 [nioEventLoopGroup-2-1] i.n.handler.logging.LoggingHandler - [id: 0x1d357822, L:/0:0:0:0:0:0:0:0:3307] ACTIVE
```

可以看出Sharding-Proxy服务已经成功运行。

应用连接Sharding-Proxy：

```
spring.datasource.url=jdbc:mysql://127.0.0.1:3307/master_slave_db?\
  serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
spring.datasource.username=root
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.password=root
```

这里有一点需要注意的是应用MySQL的驱动版本不能过高。

应用调用测试方法后，可以在Sharding-Proxy的日志中看到具体信息：

```verilog
[INFO ] 16:20:47.020 [ShardingSphere-Command-0] ShardingSphere-SQL - Rule Type: master-slave
[INFO ] 16:20:47.020 [ShardingSphere-Command-0] ShardingSphere-SQL - SQL: /* mysql-connector-java-5.1.46 ( Revision: 9cc87a48e75c2d2e87c1a293b2862ce651cb256e ) */SELECT  @@session.auto_increment_increment AS auto_increment_increment, @@character_set_client AS character_set_client, @@character_set_connection AS character_set_connection, @@character_set_results AS character_set_results, @@character_set_server AS character_set_server, @@collation_server AS collation_server, @@init_connect AS init_connect, @@interactive_timeout AS interactive_timeout, @@license AS license, @@lower_case_table_names AS lower_case_table_names, @@max_allowed_packet AS max_allowed_packet, @@net_buffer_length AS net_buffer_length, @@net_write_timeout AS net_write_timeout, @@query_cache_size AS query_cache_size, @@query_cache_type AS query_cache_type, @@sql_mode AS sql_mode, @@system_time_zone AS system_time_zone, @@time_zone AS time_zone, @@tx_isolation AS transaction_isolation, @@wait_timeout AS wait_timeout ::: DataSources: ds_slave0
[INFO ] 16:20:47.132 [ShardingSphere-Command-1] ShardingSphere-SQL - Rule Type: master-slave
[INFO ] 16:20:47.132 [ShardingSphere-Command-1] ShardingSphere-SQL - SQL: SET NAMES utf8 ::: DataSources: ds_master
[INFO ] 16:20:47.212 [ShardingSphere-Command-4] ShardingSphere-SQL - Rule Type: master-slave
[INFO ] 16:20:47.212 [ShardingSphere-Command-4] ShardingSphere-SQL - SQL: SELECT @@session.tx_isolation ::: DataSources: ds_slave0
[INFO ] 16:20:47.300 [ShardingSphere-Command-5] ShardingSphere-SQL - Rule Type: master-slave
[INFO ] 16:20:47.300 [ShardingSphere-Command-5] ShardingSphere-SQL - SQL: select *
        from t_user
        where username = 'chen' ::: DataSources: ds_slave0
[INFO ] 16:20:47.338 [nioEventLoopGroup-2-1] i.n.handler.logging.LoggingHandler - [id: 0x1d357822, L:/0:0:0:0:0:0:0:0:3307] READ: [id: 0xf76b7367, L:/127.0.0.1:3307 - R:/127.0.0.1:61134]
[INFO ] 16:20:47.340 [nioEventLoopGroup-2-1] i.n.handler.logging.LoggingHandler - [id: 0x1d357822, L:/0:0:0:0:0:0:0:0:3307] READ COMPLETE
[INFO ] 16:20:47.474 [ShardingSphere-Command-6] ShardingSphere-SQL - Rule Type: master-slave
[INFO ] 16:20:47.474 [ShardingSphere-Command-6] ShardingSphere-SQL - SQL: /* mysql-connector-java-5.1.46 ( Revision: 9cc87a48e75c2d2e87c1a293b2862ce651cb256e ) */SELECT  @@session.auto_increment_increment AS auto_increment_increment, @@character_set_client AS character_set_client, @@character_set_connection AS character_set_connection, @@character_set_results AS character_set_results, @@character_set_server AS character_set_server, @@collation_server AS collation_server, @@init_connect AS init_connect, @@interactive_timeout AS interactive_timeout, @@license AS license, @@lower_case_table_names AS lower_case_table_names, @@max_allowed_packet AS max_allowed_packet, @@net_buffer_length AS net_buffer_length, @@net_write_timeout AS net_write_timeout, @@query_cache_size AS query_cache_size, @@query_cache_type AS query_cache_type, @@sql_mode AS sql_mode, @@system_time_zone AS system_time_zone, @@time_zone AS time_zone, @@tx_isolation AS transaction_isolation, @@wait_timeout AS wait_timeout ::: DataSources: ds_slave0
[INFO ] 16:20:47.501 [ShardingSphere-Command-8] ShardingSphere-SQL - Rule Type: master-slave
[INFO ] 16:20:47.502 [ShardingSphere-Command-8] ShardingSphere-SQL - SQL: SET NAMES utf8 ::: DataSources: ds_master
```

## 参考

这里简单使用ShardingSphere实现了对数据库的读写分离。

官方文档：https://shardingsphere.apache.org/document/current/en/overview/

# 总结

这篇博客只是简单的介绍几种数据库读写分离的方式。

自己编码实现的话，需要完善从库的熔断禁用等功能。而[ShardingSphere](https://shardingsphere.apache.org/document/current/cn/)拥用编排治理的功能，提供了配置动态化、数据库熔断禁用等功能，但是相应的需要增加对ShardingSphere的了解。


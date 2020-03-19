---
title: Spring整合Mybatis分析
date: 2020-02-17 15:50:22
tags: [Mybatis, Spring]
categories: 后端开发
---

# 前言

在未整合Spring时，我们获取Mybatis的代理对象通过这样：

```java
        // 配置文件文件路径
        String resource = "mybatis/mybatis-config.xml";
        // 配置文件文件流
        InputStream inputStream = Resources.getResourceAsStream(resource);
        // 通过配置文件，初始化 SqlSessionFactory
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        // 获取 SqlSession，包含了面向数据库执行 SQL命令所需的所有方法，通过 SqlSession实例来直接执行已映射的 SQL语句
        // 简洁的方式，使用正确描述每个语句的参数和返回值的接口
        // 可以执行更清晰和类型安全的代码，而且还不用担心易错的字符串字面值以及强制类型转换
        try (SqlSession session = sqlSessionFactory.openSession()) {
            UserMapper mapper = session.getMapper(UserMapper.class);
            User user = mapper.getByUserName("admin");
            System.out.println(user.getPassword());
        }
```

而整合Spring之后，生成的接口代理对象已经注册在Spring的IoC容器中，我们可以通过`getBean()`方法获取：

```java
        ApplicationContext context = new ClassPathXmlApplicationContext(
                "/spring/mybatis/spring-mybatis.xml");
        UserMapper userMapper = (UserMapper)context.getBean("userMapper");
        System.out.println(userMapper.getByUserName("admin").getPassword());
```

# Bean注册分析

而在Spring的配置文件中，我们只注册两个Mybaits相关的bean：

```xml
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="configLocation" value="mybatis/mybatis-config.xml"/>
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="org.chensiqu.framework.mybatis.demo.mapper"/>
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
    </bean>
```

查看一下`MapperScannerConfigurer`的源码：

```java
public class MapperScannerConfigurer
    implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware {
```

可以发现其实现`BeanDefinitionRegistryPostProcessor`接口（Spring预留的扩展接口，三方可以借此来完成自定义的`BeanDefinition`的注册）：

```java
  @Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }
    // 映射接口扫描
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.setMapperFactoryBeanClass(this.mapperFactoryBeanClass);
    if (StringUtils.hasText(lazyInitialization)) {
      scanner.setLazyInitialization(Boolean.valueOf(lazyInitialization));
    }
    scanner.registerFilters();
    // 注册 BeanDefinition
    scanner.scan(
        StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }
```

可以发现`ClassPathMapperScanner`继承类`ClassPathBeanDefinitionScanner`并使用`super.doScan(basePackages)`调用类`ClassPathBeanDefinitionScanner`的`doScan()`方法：

```java
  @Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages)
          + "' package. Please check your configuration.");
    } else {
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }
```

我们来查看一下类`ClassPathBeanDefinitionScanner`的`doScan()`方法：

```java
	/**
	 * Perform a scan within the specified base packages,
	 * returning the registered bean definitions.
	 * <p>This method does <i>not</i> register an annotation config processor
	 * but rather leaves this up to the caller.
	 * @param basePackages the packages to check for annotated classes
	 * @return set of beans registered if any for tooling registration purposes (never {@code null})
	 */
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		for (String basePackage : basePackages) {
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```

也就说这个方法会扫描指定的包路径，生成`beanDefinition`对象并注册在`beanDefinitionMap`中，并将这些各属性为`ClassPathBeanDefinitionScanner`赋予的`beanDefinition`进行返回。mybatis-spring借助`ClassPathBeanDefinitionScanner`类进行路径的扫描和`beanDefinition`的生成，并在后续中进行`beanDefinition`的各项属性重新赋值：

![](image-20200219145130306.png)

在类`ClassPathMapperScanner`的`processBeanDefinitions()`方法中可以看到：

```java
  private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();
      String beanClassName = definition.getBeanClassName();
      LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName() + "' and '" + beanClassName
          + "' mapperInterface");

      // the mapper interface is the original class of the bean
      // but, the actual class of the bean is MapperFactoryBean
      definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
      // 将 definition的 beanClasss设为 MapperFactoryBean.class
      definition.setBeanClass(this.mapperFactoryBeanClass);

      definition.getPropertyValues().add("addToConfig", this.addToConfig);

      boolean explicitFactoryUsed = false;
      // 设置 sqlSessionFactory
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory",
            new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }

      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          LOGGER.warn(
              () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate",
            new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          LOGGER.warn(
              () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }

      if (!explicitFactoryUsed) {
        LOGGER.debug(() -> "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
      definition.setLazyInit(lazyInitialization);
    }
  }
```

在这里我们看到`definition`的`beanClass`设为`MapperFactoryBean.class`，也就是说在Spring实际上实例化的类是`MapperFactoryBean`：

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
```

而其实现`FactoryBean`接口，查看`getobject()`方法：

```java
  @Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }
```

看到了熟悉的`getMapper()`方法，就是Mybatis中获取代理对象的方法。Spring解析`beanDefinition`对象进行bean实例化时，如果其实现`FactoryBean`接口，则会调用`getobject()`方法，方法的返回结果对象是真正注册在IoC容器中的对象。

# 事务管理替换

在Mybatis中，一个`sqlSession`对象持有相同的数据库连接对象，而整合Spring后我们可以发现各个代理对象实际上是不同的`sqlSession`创建出来的，那是如何进行事务处理的呢？

我们知道Mybatis在执行查询语句时，会通过`transaction`的`getConnection()`方法获取连接对象，而在Spring整合Mybatis时，实际上`sqlSession`持有的事务管理器对象被替换为mybatis-spring整合包中实现的事务管理器：

```java
public class SpringManagedTransaction implements Transaction {
```

我们来查看其的`getConnection()`方法：

```java
  @Override
  public Connection getConnection() throws SQLException {
    if (this.connection == null) {
      openConnection();
    }
    return this.connection;
  }

  /**
   * Gets a connection from Spring transaction manager and discovers if this {@code Transaction} should manage
   * connection or let it to Spring.
   * <p>
   * It also reads autocommit setting because when using Spring Transaction MyBatis thinks that autocommit is always
   * false and will always call commit/rollback so we need to no-op that calls.
   */
  private void openConnection() throws SQLException {
    this.connection = DataSourceUtils.getConnection(this.dataSource);
    this.autoCommit = this.connection.getAutoCommit();
    this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);

    LOGGER.debug(() -> "JDBC Connection [" + this.connection + "] will"
        + (this.isConnectionTransactional ? " " : " not ") + "be managed by Spring");
  }
```

最终我们可以发现：

```java
public abstract class TransactionSynchronizationManager {

	private static final Log logger = LogFactory.getLog(TransactionSynchronizationManager.class);

	private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<>("Transactional resources");

	private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
			new NamedThreadLocal<>("Transaction synchronizations");

	private static final ThreadLocal<String> currentTransactionName =
			new NamedThreadLocal<>("Current transaction name");

	private static final ThreadLocal<Boolean> currentTransactionReadOnly =
			new NamedThreadLocal<>("Current transaction read-only status");

	private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
			new NamedThreadLocal<>("Current transaction isolation level");

	private static final ThreadLocal<Boolean> actualTransactionActive =
			new NamedThreadLocal<>("Actual transaction active");
```

通过线程本地变量来控制事务，也就是说即使代理对象持有的`sqlSession`不同，但是获取到的数据库连接对象还是相同的。

# 扩展Spring XML

修改配置文件：

```xml
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="configLocation" value="mybatis/mybatis-config.xml"/>
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <mybatis:scan base-package="org.chensiqu.framework.mybatis.demo.mapper"/>
```

没有在配置文件中配置注册`MapperScannerConfigurer`类，程序也能正常运行。

这是由于在mybatis-spring-2.0.3.jar的META-INF文件夹中存在spring.handlers文件和spring.schemas文件：

```
http\://mybatis.org/schema/mybatis-spring=org.mybatis.spring.config.NamespaceHandler
```

```
http\://mybatis.org/schema/mybatis-spring-1.2.xsd=org/mybatis/spring/config/mybatis-spring.xsd
http\://mybatis.org/schema/mybatis-spring.xsd=org/mybatis/spring/config/mybatis-spring.xsdxml
```

通过扫描该文件，Spring会将XML配置文件中指定命名空间的元素交由指定的类进行处理：

```java
public class NamespaceHandler extends NamespaceHandlerSupport {

  /**
   * {@inheritDoc}
   */
  @Override
  public void init() {
    registerBeanDefinitionParser("scan", new MapperScannerBeanDefinitionParser());
  }

}
```

类`MapperScannerBeanDefinitionParser`会解析scan元素中的内容，并注册`beanClass`为`MapperScannerConfigurer.class`的`beanDefinition`，剩下的步骤就由`MapperScannerConfigurer`完成。Dubbo整合Spring时也是通过这种扩展机制进行扩展识别特定的标签元素。

```java
  @Override
  protected AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);

    ClassLoader classLoader = ClassUtils.getDefaultClassLoader();

    builder.addPropertyValue("processPropertyPlaceHolders", true);
    try {
      String annotationClassName = element.getAttribute(ATTRIBUTE_ANNOTATION);
      if (StringUtils.hasText(annotationClassName)) {
        @SuppressWarnings("unchecked")
        Class<? extends Annotation> annotationClass = (Class<? extends Annotation>) classLoader
            .loadClass(annotationClassName);
        builder.addPropertyValue("annotationClass", annotationClass);
      }
      String markerInterfaceClassName = element.getAttribute(ATTRIBUTE_MARKER_INTERFACE);
      if (StringUtils.hasText(markerInterfaceClassName)) {
        Class<?> markerInterface = classLoader.loadClass(markerInterfaceClassName);
        builder.addPropertyValue("markerInterface", markerInterface);
      }
      String nameGeneratorClassName = element.getAttribute(ATTRIBUTE_NAME_GENERATOR);
      if (StringUtils.hasText(nameGeneratorClassName)) {
        Class<?> nameGeneratorClass = classLoader.loadClass(nameGeneratorClassName);
        BeanNameGenerator nameGenerator = BeanUtils.instantiateClass(nameGeneratorClass, BeanNameGenerator.class);
        builder.addPropertyValue("nameGenerator", nameGenerator);
      }
      String mapperFactoryBeanClassName = element.getAttribute(ATTRIBUTE_MAPPER_FACTORY_BEAN_CLASS);
      if (StringUtils.hasText(mapperFactoryBeanClassName)) {
        @SuppressWarnings("unchecked")
        Class<? extends MapperFactoryBean> mapperFactoryBeanClass = (Class<? extends MapperFactoryBean>) classLoader
            .loadClass(mapperFactoryBeanClassName);
        builder.addPropertyValue("mapperFactoryBeanClass", mapperFactoryBeanClass);
      }
    } catch (Exception ex) {
      XmlReaderContext readerContext = parserContext.getReaderContext();
      readerContext.error(ex.getMessage(), readerContext.extractSource(element), ex.getCause());
    }

    builder.addPropertyValue("sqlSessionTemplateBeanName", element.getAttribute(ATTRIBUTE_TEMPLATE_REF));
    builder.addPropertyValue("sqlSessionFactoryBeanName", element.getAttribute(ATTRIBUTE_FACTORY_REF));
    builder.addPropertyValue("lazyInitialization", element.getAttribute(ATTRIBUTE_LAZY_INITIALIZATION));
    builder.addPropertyValue("basePackage", element.getAttribute(ATTRIBUTE_BASE_PACKAGE));

    return builder.getBeanDefinition();
  }
```

# 自定义扩展Spring XML

通过这种机制我们也可以自己扩展Spring XML。

1. 创建xsd文件：

   ```xml
   <?xml version="1.0" encoding="UTF-8" standalone="no"?>
   <xsd:schema xmlns="http://www.csiqu.club/schema/my"
               xmlns:xsd="http://www.w3.org/2001/XMLSchema"
               xmlns:beans="http://www.springframework.org/schema/beans"
               targetNamespace="http://www.csiqu.club/schema/my">
   
       <xsd:import namespace="http://www.springframework.org/schema/beans"/>
       <xsd:annotation>
           <xsd:documentation>
               <![CDATA[ Namespace support for the my demo. ]]></xsd:documentation>
       </xsd:annotation>
   
       <xsd:element name="user">
           <xsd:complexType>
               <xsd:complexContent>
                   <xsd:extension base="beans:identifiedType">
                       <xsd:attribute name="name" type="xsd:string" default="chen">
                           <xsd:annotation>
                               <xsd:documentation><![CDATA[ The name of user. ]]></xsd:documentation>
                           </xsd:annotation>
                       </xsd:attribute>
   
                   </xsd:extension>
               </xsd:complexContent>
           </xsd:complexType>
       </xsd:element>
   </xsd:schema>
   ```

2. 编写`NamespaceHandler`和`BeanDefinitionParser`

   ```java
   public class MyBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {
   
       @Override
       protected Class<User> getBeanClass(Element element) {
           return User.class;
       }
   
       @Override
       protected void doParse(Element element, BeanDefinitionBuilder bean) {
           bean.setLazyInit(false);
           bean.addPropertyValue("name", element.getAttribute("name"));
       }
   }
   ```

   ```java
   public class MyNamespaceHandler extends NamespaceHandlerSupport {
   
       @Override
       public void init() {
           registerBeanDefinitionParser("user", new MyBeanDefinitionParser());
       }
   }
   ```

3. 在META-INF中加入spring.handlers和spring.schemas文件：

   ```
   http\://www.csiqu.club/schema/my=org.chensiqu.framework.spring.handler.MyNamespaceHandler
   ```

   ```
   http\://www.csiqu.club/schema/my.xsd=META-INF/my.xsd
   ```

4. 进行测试：

   自定义扩展的作用就是在IoC容器中注册`User`的实例对象（忽略getter和setter）：

   ```java
   public class User {
   
       private String name;
   
       private Integer age;
   
   }
   ```

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns:my="http://www.csiqu.club/schema/my"
          xsi:schemaLocation="http://www.springframework.org/schema/beans
   	http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
       http://www.csiqu.club/schema/my
       http://www.csiqu.club/schema/my.xsd">
   
       <my:user id="user" name="chensiqu"/>
   </beans>
   ```

   ```java
       public static void main(String[] args) {
           ApplicationContext context = new ClassPathXmlApplicationContext(
                   "/spring/handler/spring-handler.xml");
           // 配置文件的方式进行注入
           User user = (User)context.getBean("user");
           System.out.println(user.getName());
       }
   ```

   运行结果可以发现已经成功在IoC容器中注册。

# 其他

自定义XML扩展参考官方文档 9.2节：

https://docs.spring.io/spring/docs/5.1.5.RELEASE/spring-framework-reference/core.html#xsd-custom-schema

| 框架           | 版本          |
| -------------- | ------------- |
| Spring         | 5.1.5.RELEASE |
| Mybatis        | 3.5.2         |
| mybatis-spring | 2.0.3         |

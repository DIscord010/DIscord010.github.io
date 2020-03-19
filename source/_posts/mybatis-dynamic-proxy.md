---
title: Mybatis代理方法执行流程
date: 2019-11-13 09:09:42
tags: [Mybatis]
categories: 后端开发
---

在不整合Spring的情况下，我们获取Mybatis的代理对象通过如下的方式：

```java
UserMapper mapper = session.getMapper(UserMapper.class);
```

查看源码可以发现使用的是JDK动态代理的方式（在类`MapperProxyFactory<T>`中)，`getMapper()`方法最终会调用此方法来获取实例：

```java
protected T newInstance(MapperProxy<T> mapperProxy) {
    // 获取代理对象
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), 
            new Class[] { mapperInterface }, mapperProxy);
  }
```

为了更好地了解这个过程，我们来简单使用JDK动态代理生成接口的代理实例对象：

1、定义一个接口

```java
public interface Person {

    void speak(String words);
}
```

2、编写实现`InvocationHandler`接口的代理处理类

```java
public class MyProxy implements InvocationHandler {

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) {
        // 直接打印方法的第一个参数
        System.out.println(args[0]);
        return null;
    }
}
```

3、测试类

```java
public class ProxyTest {

    public static void main(String[] args) {
        Person instance = (Person) Proxy.newProxyInstance(
                ClassLoader.getSystemClassLoader(),
                new Class[]{Person.class}, new MyProxy());
        instance.speak("hello world");
    }
}
```

4、执行结果

```bash
hello world
```

了解JDK动态代理之后，我们来查看Mybatis代理处理类的`invoke()`方法（类`MapperProxy<T>`中）：

```java
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (method.isDefault()) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

可以看到，首先先排除继承自`Object`类的方法，比如`toString()`方法，是不需要进行代理的；接口的默认方法也是不需要进行代理的（Java8新增）。通过`cachedMapperMethod()`方法获取`MapperMethod`类的实例：

```java
  private MapperMethod cachedMapperMethod(Method method) {
    // 缓存中不存在则创建
    return methodCache.computeIfAbsent(method, k -> new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
  }
```

然后执行类`MapperMethod`的`execute()`方法中：

部分代码如下：

```java
switch (command.getType()) {
      case INSERT: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
          if (method.returnsOptional()
              && (result == null || !method.getReturnType().equals(result.getClass()))) {
            result = Optional.ofNullable(result);
          }
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
```

先调用`convertArgsToSqlCommandParam()`方法对参数进行转换，方便后面真正执行SQL查询时使用。可以看到我们执行insert，update，delete语句的时候返回的都是`rowCount()`方法的返回对象也就是操作命中的行数。我们来查看一下`SqlSession`接口的部分源码：

```java
  /**
   * Retrieve a single row mapped from the statement key and parameter.
   * @param <T> the returned object type
   * @param statement Unique identifier matching the statement to use.
   * @param parameter A parameter object to pass to the statement.
   * @return Mapped object
   */
  <T> T selectOne(String statement, Object parameter);
```

接口的注释很全，可以看出此方法的作用就是使用给定的参数执行查询语句并返回结果对象。查看`SqlSession`的一个默认实现类`DefaultSqlSession`的源码：

```java
public class DefaultSqlSession implements SqlSession {

  private final Configuration configuration;
  private final Executor executor;

  private final boolean autoCommit;
  private boolean dirty;
  private List<Cursor<?>> cursorList;
```

其持有实现`Executor`接口的实例对象也就是真正执行SQL语句的执行器。`sqlSession`会将`MappedStatement`的实例对象传递给`executor`。而继承`Executor`接口的抽象类`BaseExecutor`使用模板设计模式，具体的方法由子类实现。其中的一个默认子类`SimpleExecutor`，我们查看它的`doQuery()`方法：

```java
  @Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
```

看到了熟悉的`Statement`接口。在`prepareStatement()`方法中调用了`getConnection()`方法获取到了数据库连接对象并获得`Statement`实例对象：

```java
  protected Connection getConnection(Log statementLog) throws SQLException {
    Connection connection = transaction.getConnection();
    if (statementLog.isDebugEnabled()) {
      return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    } else {
      return connection;
    }
  }
```

而`getConnection()`实际调用的是`Transaction#getConnection()`,在`JdbcTransaction`这个实现类中先判断`connection`是否为空，为空则调用的`openConnection()`方法：

```java
  protected void openConnection() throws SQLException {
    if (log.isDebugEnabled()) {
      log.debug("Opening JDBC Connection");
    }
    connection = dataSource.getConnection();
    if (level != null) {
      connection.setTransactionIsolation(level.getLevel());
    }
    setDesiredAutoCommit(autoCommit);
  }
```

在Mybatis中，`sqlsession`持有`executor`，`executor`持有`transaction`，`transaction`持有`dataSource`以及`connection`。只有在第一次真正执行查询语句时，才会从使用 `DataSource#getConnection()`获取数据库连接对象。

而`handler.query(stmt, resultHandler);`我们查看其中的一个实现：

```java
  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    String sql = boundSql.getSql();
    statement.execute(sql);
    // 进行结果集的处理
    return resultSetHandler.handleResultSets(statement);
  }
```

可以发现就是调用JDBC执行语句后再对结果集进行对象映射。而结果集的映射处理，大体上就是遍历结果集，通过反射调用对应的setter方法进行属性的赋值。具体的实现也是比较复杂。

# 小结

- Mybatis使用JDK动态代理来生成接口的实例对象。
- Mybatis在执行代理方法时会生成`mapperMethod`对象并进行缓存。
- Mybatis只有在第一次执行查询语句时，才会从连接源获取连接对象。

# 版本

| ArtifactID | 版本  |
| ---------- | :---- |
| mybatis    | 3.5.2 |


---
title: 模板方法模式
date: 2020-03-06 11:28:17
tags: [设计模式]
categories: 设计模式
---

Spring和Mybatis中都使用到了模板方法设计模式。

Mybatis：

在抽象类`BaseExecutor`定义了四个抽象方法（基本方法）：

```java
  protected abstract int doUpdate(MappedStatement ms, Object parameter)
      throws SQLException;

  protected abstract List<BatchResult> doFlushStatements(boolean isRollback)
      throws SQLException;

  protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
      throws SQLException;

  protected abstract <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql)
      throws SQLException;
```

并且在模板方法`queryFromDatabase`中实现对基本方法的调度：

```java
  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```

Spring：

抽象类`AbstractBeanDefinitionParser`中`parse(Element element, ParserContext parserContext)`方法即为模板方法，基本方法`parseInternal()`由子类实现，来完成`BeanDefinition`的生成逻辑。

```java
	/**
	 * Central template method to actually parse the supplied {@link Element}
	 * into one or more {@link BeanDefinition BeanDefinitions}.
	 * @param element the element that is to be parsed into one or more {@link BeanDefinition BeanDefinitions}
	 * @param parserContext the object encapsulating the current state of the parsing process;
	 * provides access to a {@link org.springframework.beans.factory.support.BeanDefinitionRegistry}
	 * @return the primary {@link BeanDefinition} resulting from the parsing of the supplied {@link Element}
	 * @see #parse(org.w3c.dom.Element, ParserContext)
	 * @see #postProcessComponentDefinition(org.springframework.beans.factory.parsing.BeanComponentDefinition)
	 */
	@Nullable
	protected abstract AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext);
```

# 定义

> The template method is a method in a superclass, usually an abstract superclass, and defines the skeleton of an operation in terms of a number of high-level steps. These steps are themselves implemented by additional helper methods in the same class as the template method.  The helper methods may be either abstract methods, for which case subclasses are required to provide concrete implementations, or hook methods, which have empty bodies in the superclass. Subclasses can (but are not required to) customize the operation by overriding the hook methods. The intent of the template method is to define the overall structure of the operation, while allowing subclasses to refine, or redefine, certain steps.

模板方法是超类中的方法（通常是抽象类），模板方法定义一个操作的步骤框架，而这些步骤由基本方法实现。

基本方法可以是抽象方法也可以是钩子方法。

模板方法的目的是定义操作的整体结构，同时允许子类细化或重新定义某些步骤。

# 类图

![](image-20200309104817539.png)

# 使用场景

- 多个子类有公用的方法，且逻辑基本相同时。
- 重要、复杂的算法，可以把核心算法设计为模板方法，相关细节功能由子类实现。

# 简单实践

```java
public abstract class BaseTeaProvider {

    public final void tea() {
        getTea();
        preparingTeaSet();
        makeTea();
    }

    /** 获取茶叶 */
    protected abstract void getTea();

    /** 准备茶具 */
    protected abstract void preparingTeaSet();

    /** 泡茶 */
    protected abstract void makeTea();
}
```

```java
public class GreenTeaProvider extends BaseTeaProvider {

    @Override
    protected void getTea() {
        System.out.println("获取绿茶");
    }

    @Override
    protected void preparingTeaSet() {
        System.out.println("准备并清洗专业茶具");
    }

    @Override
    protected void makeTea() {
        System.out.println("绿茶泡五分钟");
    }
}
```

```java
public class RedTeaProvider extends BaseTeaProvider {

    @Override
    protected void getTea() {
        System.out.println("获取红茶");
    }

    @Override
    protected void preparingTeaSet() {
        System.out.println("清洗普通茶具");
    }

    @Override
    protected void makeTea() {
        System.out.println("红茶泡十分钟");
    }
}
```

父类定义模板方法`tea()`规定上茶的几个步骤，由子类实现具体的步骤行为。

# 参考

《设计模式之禅(第2版)》


---
title: Maven多环境配置
date: 2020-04-16 17:31:02
tags: [Maven]
categories: 后端开发
---

通常生产环境和测试环境的相关配置是完全不一样的，我们可以通过Maven的`filters`针对不同的环境进行打包，配合持续部署。

# 目录结构

![](image-20200416174909204.png)

product.properties：

```properties
port=8085
```

# Maven配置

```xml
    <profiles>
        <profile>
            <id>product</id>
            <properties>
                <env>product</env>
            </properties>
            <activation>
                <!-- 默认激活配置 -->
                <activeByDefault>true</activeByDefault>
            </activation>
        </profile>
        <profile>
            <id>test</id>
            <properties>
                <env>test</env>
            </properties>
        </profile>
    </profiles>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>

        <filters>
            <filter>src/main/filter/${env}.properties</filter>
        </filters>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
            </resource>
        </resources>
    </build>
```

# 项目配置文件

项目配置文件中使用占位符：

```properties
port=${port}
```

![](image-20200416175430211.png)

这时候使用`mvn clean package -P [profile id]` 命令就可以进行不同环境下的打包。


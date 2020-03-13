---
title: Docker MySQL容器异常退出排查
date: 2020-02-10 15:58:07
tags: [Docker]
categories: Linux
---

云上的MySQL 5.6（使用Docker搭建）突然无法连接。

# 问题排查

查看容器状态：

```shell
[root@centos ~]# docker ps -a
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS                         PORTS                       NAMES
6d7f491dcb2b        mysql:5.6                   "docker-entrypoint.s…"   About an hour ago   Exited (1) About an hour ago                               mysql

```

发现容器异常退出，查看容器日志：

```shell
docker logs mysql
```

```shell
MySQL init process in progress...
MySQL init process in progress...
MySQL init process in progress...
MySQL init process in progress...
MySQL init process failed.
```

未能找到有效的异常信息。

搜索之后，可以通过查看Linux的日志来分析进程异常挂掉：

```shell
dmesg | grep mysql
```

```shell
[14859486.695566] Out of memory: Kill process 14229 (mysqld) score 212 or sacrifice child
[14859486.698149] Killed process 14229 (mysqld) total-vm:440344kB, anon-rss:399816kB, file-rss:4kB, shmem-rss:0kB
```

很明显的找到了原因：由于内存不足，Linux内核选择杀掉MySQL进程。

> Apparently all modern Linux kernels have a built-in mechanism called “Out Of Memory killer” which can annihilate your processes under extremely low memory conditions. When such a condition is detected, the killer is activated and picks a process to kill.

# 解决方案

找到了原因之后，需要进行解决。最简单的方法就是加大机器的内存。

在这里我选择调整MySQL的配置文件，降低MySQL的内存占用来防止MySQL进程再被“Out Of Memory killer“杀掉。先重启容器，查看在调整前的MySQL内存占用和配置参数：

```java
[root@centos ~]# top
```

键入M来依照内存占用排序：

```shell
15094 polkitd   20   0 1322072 466768   1852 S  0.0 24.8   0:06.74 mysqld
```

机器内存大小为2G，MySQL占24.8%也就是500M左右，这也是MySQL的默认配置内存占用大小。

进入容器内部并修改MySQL的默认配置：

```shell
docker exec -it mysql bash
```

修改MySQL的配置文件（/etc/mysql/mysql.conf.d/mysqld.cnf），添加如下内容：

```shell
performance_schema_max_table_instances=400
table_definition_cache=400
table_open_cache=256
```

相关配置的具体信息可以查看官方文档：

https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html

重启容器后，再查看MySQL的内存占用：

```shell
20352 polkitd   20   0  903372  57464   6752 S  0.0  3.1   0:00.21 mysqld
```

内存占用为3.1%，有效降低了MySQL的内存占用。
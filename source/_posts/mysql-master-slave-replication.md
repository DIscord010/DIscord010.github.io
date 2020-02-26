---
title: MySQL复制集群搭建
date: 2020-02-12 10:44:36
tags: [主从复制]
categories: MySQL
---

MySQL复制功能即让一台MySQL服务器的数据与其他服务器保持同步。典型的应用场景即一主多从，应用进行读写分离和负载均衡。在读操作远大于写操作的场景下可以保证数据的查询效率。

# 复制实现

## 基本步骤

1、当主库数据发生更改时，主库将数据更改记录到二进制日志（Binary Log）

2、备库通过I/O线程将二进制日志复制到自己的中继日志（Relay Log）

3、备库读取中继日志中的事件，重放到备库数据

# 复制拓扑

## 一主一备

![](image-20200213145924780.png)

## 一主多备

![](image-20200213150113176.png)

## 双向复制

![](image-20200213150209195.png)

## 环形复制

![](image-20200213151215298.png)

## 树形复制

![](image-20200213150254936.png)

双向复制和环形复制通常不会采用。

# 复制配置

在这里简单的实现一主一从拓扑的MySQL主从复制集群。

我们需要有两台MySQL服务器，版本最好一致，MySQL复制大部分是向后兼容的，在不一致的情况低版本的作为主库。这里我使用Docker搭建了两个5.6版本的MySQL服务：

```shell
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS                                                                                        NAMES
549f4b23ec34        mysql:5.6                "docker-entrypoint.s…"   21 hours ago        Up 19 hours         0.0.0.0:3307->3306/tcp                                                                       mysql-slave
d848a7e92edc        wurstmeister/zookeeper   "/bin/sh -c '/usr/sb…"   4 months ago        Up 4 months         22/tcp, 2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp                                           zookeeper
d0ec64cedb85        mysql:5.6                "docker-entrypoint.s…"   4 months ago        Up 26 hours         0.0.0.0:3306->3306/tcp                                                                       mysql

```

## 创建复制账号

复制账户实际上拥有主库的REPLICATION SLAVE权限即可进行复制，这里也增加REPLICATION CLIENT权限进行监控和管理复制：

```mysql
mysql> GRANT REPLICATION SLAVE,REPLICATION CLIENT ON *.*
    -> TO repl@'%' IDENTIFIED BY 'password'
    -> ;
Query OK, 0 rows affected (0.02 sec)
```

创建可以在任意主机登录的用户名为`repl`，密码为`password`并拥有`REPLICATION SLAVE,REPLICATION CLIENT`权限的账户。

## 主库配置

服务器需要打开二进制日志并制定一个独一无二的服务器ID

在主库的配置文件中增加：

```shell
log_bin=mysql-bin
server_id=3306
```

验证二进制日志文件是否创建成功：

```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      120 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

## 备库配置

备库的配置文件中增加：

```shell
# 打开二进制日志
log_bin=mysql-bin
# 指定独一无二的服务器 ID
server_id=3307
# 指定中继日志的位置
relay_log=/var/lib/mysql/mysql-relay-bin
# 允许备库重放的事件也记录到自身的二进制日志中
log_slave_updates=1
# 阻止没有特殊权限的线程修改本库的数据
read_only=1
```

## 启动复制

在备库上执行基本命令：

```mysql
mysql> change master to master_host='123.207.89.169',
    -> master_user='repl',
    -> master_password='password',
    -> master_log_file='mysql-bin.000001',
    -> master_log_pos=0;
Query OK, 0 rows affected, 2 warnings (0.06 sec)
```

出现了两个警告，进行查看：

```mysql

mysql> mysql> show warnings;
+-------+------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message                                                                                                                                                                                                                                                                              |
+-------+------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note  | 1759 | Sending passwords in plain text without SSL/TLS is extremely insecure.                                                                                                                                                                                                               |
| Note  | 1760 | Storing MySQL user name or password information in the master info repository is not secure and is therefore not recommended. Please consider using the USER and PASSWORD connection options for START SLAVE; see the 'START SLAVE Syntax' in the MySQL Manual for more information. |
+-------+------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
2 rows in set (0.00 sec)
```

可以发现是安全问题，先不进行处理。

检查复制是否正确执行：

```mysql
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State:
                  Master_Host: 123.207.89.169
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 4
               Relay_Log_File: mysql-relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: No
            Slave_SQL_Running: No
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 4
              Relay_Log_Space: 120
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 0
                  Master_UUID:
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State:
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
1 row in set (0.00 sec)
```

可以看出复制功能还未运行，执行命令开始复制：

```mysql
mysql> start slave;
Query OK, 0 rows affected (0.01 sec)
```

命令没有显示错误，再输入`show slave status`命令进行检查：

```mysql
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 123.207.89.169
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 120
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 283
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 120
              Relay_Log_Space: 456
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 3306
                  Master_UUID: 33155f8f-e334-11e9-ac6a-0242ac110002
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
1 row in set (0.00 sec)

```

可以看出复制功能已经启动，状态也变为了等待主库发送事件。

也能使用`show  processlist`命令查看线程信息：

```mysql

mysql> show processlist;
+----+-------------+-----------+------+---------+------+-----------------------------------------------------------------------------+------------------+
| Id | User        | Host      | db   | Command | Time | State                                                                       | Info             |
+----+-------------+-----------+------+---------+------+-----------------------------------------------------------------------------+------------------+
|  1 | root        | localhost | NULL | Query   |    0 | init                                                                        | show processlist |
|  2 | system user |           | NULL | Connect |  264 | Waiting for master to send event                                            | NULL             |
|  3 | system user |           | NULL | Connect |  263 | Slave has read all relay log; waiting for the slave I/O thread to update it | NULL             |
+----+-------------+-----------+------+---------+------+-----------------------------------------------------------------------------+------------------+
3 rows in set (0.00 sec)
```

也可以看出I/O复制线程在等待主库发送事件。

## 验证

复制功能配置完成后，我们来进行验证是否能够成功进行复制：

主库新建一数据库并插入一些数据：

![](image-20200212155524979.png)

查看备库：

![](image-20200212155622871.png)

复制功能正常运行。

# 其他

这篇博文只是做个记录，大部分内容参考《高性能mysql(第3版)》。


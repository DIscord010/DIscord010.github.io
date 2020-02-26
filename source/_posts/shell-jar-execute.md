---
title: Shell脚本执行JAR
date: 2019-10-30 15:19:52
tags: [Shell]
categories: Linux
---

最近做的项目使用SpringBoot并打包成可执行JAR在Linux环境下进行部署。每次进行更新上传JAR文件至Linux服务器后，都需要手动输入命令进行重新部署，于是参照网上资料，编写了脚本进行程序的执行和重启。

## Linux下命令执行JAR

1、打印进程信息

```bash
ps -ef|grep xxx.jar
```

2、手动kill进程

```bash
kill pid
```

3、重新执行项目

```bash
nohup java -jar xxx.jar >log.log 2>&1 &
```

## 编写Shell脚本执行JAR

效果如下：

```bash
./service.sh restart
```

```bash
>>> api PID = 1001 begin kill 1001 <<<
>>> xxx.jar is not running <<<
>>> start xxx.jar successed PID=2472 <<<
```

完整脚本：

```bash
#!/bin/sh
# java env
#export JAVA_HOME=/usr/local/jdk/jdk1.8.0_231
#export JRE_HOME=$JAVA_HOME/jre

# JAR文件名
APPLICATION_NAME=directory-push-receive-0.0.1-SNAPSHOT
JAR_NAME=$APPLICATION_NAME\.jar
# PID代表的是 PID文件
PID=$APPLICATION_NAME\.pid

# 使用说明，用来提示输入参数
usage() {
  echo "Usage: sh 执行脚本.sh [start|stop|restart|status]"
  exit 1
}

# 检查程序是否在运行
is_exist() {
  pid=$(pgrep -f $JAR_NAME)
  # 如果不存在返回 1，存在返回 0
  # True of the length if "STRING" is zero.
  if [ -z "${pid}" ]; then
    return 1
  else
    return 0
  fi
}

# 启动方法
start() {
  # if判断的不是返回值，而是调用是否成功。不要理解为 if 0。与以下代码效果相同：
  #
  #  is_exist
  #  if [ $? -eq "0" ]; then
  #
  # 此处是判断函数是否正常执行，在 linux中函数正常执行的返回值即为 0
  if is_exist; then
    echo ">>> ${JAR_NAME} is already running PID=${pid} <<<"
  else
    # 此处可以配置 JVM参数
    nohup "$JRE_HOME"/bin/java -Xms512m -Xmx512m -jar $JAR_NAME >logs.log 2>&1 &
    # 将 pid输出到 PID文件
    echo $! >$PID
    echo ">>> start $JAR_NAME successed PID=$! <<<"
  fi
}

# 停止方法
stop() {
  # 读取 PID文件获取 pid
  pidf=$(cat $PID)
  echo ">>> application PID = $pidf begin kill $pidf <<<"
  kill "$pidf"
  rm -rf $PID
  sleep 2

  if is_exist; then
    echo ">>> application 2 PID = $pid begin kill -9 $pid <<<"
    kill -9 "$pid"
    sleep 2
    echo ">>> $JAR_NAME process stopped <<<"
  else
    echo ">>> ${JAR_NAME} is not running <<<"
  fi
}

# 输出运行状态
status() {
  if is_exist; then
    echo ">>> ${JAR_NAME} is running PID is ${pid} <<<"
  else
    echo ">>> ${JAR_NAME} is not running <<<"
  fi
}

# 重启
restart() {
  stop
  start
}

# 根据输入参数，选择执行对应方法，不输入则执行使用说明
case "$1" in
"start")
  start
  ;;
"stop")
  stop
  ;;
"status")
  status
  ;;
"restart")
  restart
  ;;
*)
  usage
  ;;
esac
exit 0

```

脚本的编写参照了这篇博文： https://www.cnblogs.com/aizj/p/10020310.html 进行了简单的修改。

## 格式转换问题

将脚本上传至服务器后，增加可执行权限：

```bash
 chmod a+x ./service.sh
```

执行出现如下提示：

```bash
./service.sh start
-bash: ./service.sh: /bin/sh^M: bad interpreter: No such file or directory
```

由于脚本是在windows环境下编写的需要更改文件的格式：

```bash
vi service.sh
# 进入文本编辑后执行
# 设置格式
:set ff=unix
# 保存并退出
:wq
```

更改格式后即可正常执行。
---
title: Dubbo异步转同步原理
date: 2020-01-02 17:20:23
tags: [Dubbo]
categories: 后端开发
---

Dubbo发送数据至服务方后，在通信层面是异步的，通信线程并不会等待结果数据返回。而我们在使用Dubbo进行RPC调用缺省就是同步的，这其中就涉及到了异步转同步的操作。

# 2.6.x

在Dubbo 2.6.x的版本中，通过断点我们可以发现调用PRC方法阻塞在`DefaultFuture#get()`上：

```java
    @Override
    public Object get() throws RemotingException {
        return get(timeout);
    }

    @Override
    public Object get(int timeout) throws RemotingException {
        if (timeout <= 0) {
            timeout = Constants.DEFAULT_TIMEOUT;
        }
        if (!isDone()) {
            long start = System.currentTimeMillis();
            lock.lock();
            try {
                // 循环判断
                while (!isDone()) {
                    done.await(timeout, TimeUnit.MILLISECONDS);
                    // 判断结果是否返回
                    if (isDone() || System.currentTimeMillis() - start > timeout) {
                        break;
                    }
                }
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } finally {
                lock.unlock();
            }
            if (!isDone()) {
                throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
            }
        }
        return returnFromResponse();
    }
```

在调用`get()`方法时，`done.await(timeout, TimeUnit.MILLISECONDS);`阻塞调用线程。而在通信层获取到结果数据时，就会调用`DefaultFuture#doReceived(Response res)`方法进行唤醒：

```java
    private void doReceived(Response res) {
        lock.lock();
        try {
            // 赋值
            response = res;
            if (done != null) {
                // 唤醒阻塞在该条件上的线程
                done.signal();
            }
        } finally {
            lock.unlock();
        }
        if (callback != null) {
            invokeCallback(callback);
        }
    }
```

从上诉的两段代码中，可以看出Dubbo RPC调用的流程：客户端调用RPC方法 -> 生成该调用的唯一ID -> 序列化信息后发送至服务端，同时判断此次调用类型是否为同步调用，是则进行阻塞操作 -> 通信层得到服务端返回结果后反序列，根据ID获取到`DefaultFuture`对象，进行唤醒操作。

# 2.7.x

而在2.7.x版本中，这种自实现的异步转同步操作进行了修改。新的`DefaultFuture`继承了`CompletableFuture`，新的`doReceived(Response res)`方法如下：

```java
    private void doReceived(Response res) {
        if (res == null) {
            throw new IllegalStateException("response cannot be null");
        }
        if (res.getStatus() == Response.OK) {
            this.complete(res.getResult());
        } else if (res.getStatus() == Response.CLIENT_TIMEOUT || res.getStatus() == Response.SERVER_TIMEOUT) {
            this.completeExceptionally(new TimeoutException(res.getStatus() == Response.SERVER_TIMEOUT, channel, res.getErrorMessage()));
        } else {
            this.completeExceptionally(new RemotingException(channel, res.getErrorMessage()));
        }
    }
```

通过`CompletableFuture#complete`方法来设置异步的返回结果，且删除旧的`get()`方法，使用`CompletableFuture#get()`方法：

```java
    public T get() throws InterruptedException, ExecutionException {
        Object r;
        return reportGet((r = result) == null ? waitingGet(true) : r);
    }
```

使用`CompletableFuture`完成了异步转同步的操作。

# 小结

使用`CompletableFuture`可以很方便的简单实现异步转同步的操作：

```java
public class Asyc2SyncMain {

    private static final Logger LOG = LoggerFactory.getLogger(Asyc2SyncMain.class);

    public static final ThreadFactory THREAD_NAME_FACTORY =
            new ThreadFactoryBuilder().setNameFormat("异步获取数据线程池:%d").build();

    public static ThreadPoolExecutor executor = new ThreadPoolExecutor(
            1,
            1,
            0,
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(100),
            THREAD_NAME_FACTORY);


    public static void main(String[] args) throws ExecutionException, InterruptedException {
        CompletableFuture<String> completableFuture = new CompletableFuture<>();
        executor.execute(() -> {
            try {
                TimeUnit.SECONDS.sleep(10);
            } catch (InterruptedException e) {
                LOG.info("线程休眠中断异常");
            }
            completableFuture.complete("complete");
        });
        LOG.info("结果为：{}", completableFuture.get());
    }
}
```

通过控制台结果可以看出主线程阻塞在`get()`方法上直到我们调用`CompletableFuture#complete`方法设置返回结果。
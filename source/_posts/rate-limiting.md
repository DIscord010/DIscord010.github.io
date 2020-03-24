---

title: 服务限流实现
date: 2020-03-16 16:54:04
tags: [限流]
categories: 后端开发
---

限流是服务降级的一种方式。为保证系统的稳定运行，防止突发的流量洪峰把系统打垮，需要对服务进行限流处理。

# guava `RateLimiter`

## 限流注解

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Limiter {

    /**
     * 限流器的 key
     *
     * @return key
     */
    String key();

    /**
     * 每秒许可
     *
     * @return 每秒许可
     */
    double permitsPerSecond();
}
```

## 切面编程

```java
@Component
@Aspect
@Order(1)
@SuppressWarnings("UnstableApiUsage")
public class LimiterAspect {

    /** 限流器容器 */
    private static final ConcurrentHashMap<String, RateLimiter> LIMITER_MAP = new ConcurrentHashMap<>(16);

    @Pointcut("@annotation(club.csiqu.limitdemo.common.spring.limter.Limiter)")
    public void lockAspect() {}

    @Around("lockAspect()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature signature = (MethodSignature)joinPoint.getSignature();
        Method method = signature.getMethod();
        Limiter limiterAnnotation = method.getAnnotation(Limiter.class);
        String key = limiterAnnotation.key();
        RateLimiter limiter = LIMITER_MAP.putIfAbsent(key,
                RateLimiter.create(limiterAnnotation.permitsPerSecond()));
        if (limiter == null) {
            limiter = LIMITER_MAP.get(key);
        }
        boolean flag = limiter.tryAcquire();
        if (!flag) {
            return "limited";
        }
        return joinPoint.proceed();
    }
}
```

## 服务限流

```java
@Service
public class HelloServiceImpl implements HelloService {
    
    @Override
    @Limiter(key = "hello", permitsPerSecond = 10)
    public String hello() {
        return "hello";
    }
}
```

## 验证

使用JMeter进行测试，同时发送500个请求，进行赛选后可以发现只通过10+个请求：

![](image-20200324140346600.png)

# Redisson `RRateLimiter`

如果服务进行集群化，guava工具包提供的`RateLimiter`只能针对单个应用节点的服务进行限制，无法统一对整个集群进行流量限制。使用Redisson实现的`RRateLimiter`来完成分布式限流：

## 切面编程

```java
@Component
@Aspect
@Order(1)
public class LimiterAspect {

    private final RedissonClient redisson;

    private static final String REDIS_LIMITER_PREFIX = "redis-limiter";

    public LimiterAspect(RedissonClient redissonClient) {
        this.redisson = redissonClient;
    }

    @Pointcut("@annotation(club.csiqu.limitdemo.common.spring.limter.Limiter)")
    public void lockAspect() {}

    @Around("lockAspect()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature signature = (MethodSignature)joinPoint.getSignature();
        Method method = signature.getMethod();
        Limiter limiterAnnotation = method.getAnnotation(Limiter.class);
        String key = limiterAnnotation.key();
        RRateLimiter limiter = redisson.getRateLimiter(REDIS_LIMITER_PREFIX + key);
        // trySetRate方法只会执行一次，目前暂时没有提供方法重新设置速率，只能删除后重建 issue#2322
        limiter.trySetRate(RateType.OVERALL, 10, 1, RateIntervalUnit.SECONDS);
        boolean allow = limiter.tryAcquire();
        if (!allow) {
            return "limited";
        }
        return joinPoint.proceed();
    }
}
```

## 动态修改限流速率

目前官方没有提供覆盖限流配置的方法，官方的建议是删除该对象后进行重建：

> You can delete it and then set config again.

```java
@Component
public class LimiterUtil {

    private final RedissonClient redisson;

    private static final Logger LOGGER = LoggerFactory.getLogger(LimiterUtil.class);

    @Autowired
    private LimiterUtil(RedissonClient redissonClient) {
        this.redisson = redissonClient;
    }

    public void updateRedissonLimiterConfig(String key, long rate, long rateInterval, RateIntervalUnit rateIntervalUnit) {
        RRateLimiter limiter = redisson.getRateLimiter(REDIS_LIMITER_PREFIX + key);
        if (!limiter.isExists()) {
            LOGGER.warn("key:{}对应的限流器不存在.", REDIS_LIMITER_PREFIX + key);
        }
        // 循环直到重新配置成功
        while (!limiter.trySetRate(RateType.OVERALL, rate, rateInterval, rateIntervalUnit)) {
            limiter.delete();
            limiter = redisson.getRateLimiter(REDIS_LIMITER_PREFIX + key);
        }
    }
}
```

测试一下是否可行：

```java
limiter.trySetRate(RateType.OVERALL, 1, 1, RateIntervalUnit.SECONDS);
```

可以发现50个请求只通过一个：

![](image-20200324160904393.png)

调用方法修改限流速率后：

```java
limiterUtil.updateRedissonLimiterConfig("hello", 50, 1, RateIntervalUnit.SECONDS);
```

![](image-20200324161003215.png)

50个请求全部通过。

# 参考

https://github.com/redisson/redisson/wiki/6.-distributed-objects

https://github.com/redisson/redisson/issues/2322
这三个方法不会直接跳过
equals
hashCode
toString

Feign的负载俊航怎么实现的？
1. RoundRobin 使用 AtomicInteger % 实例的size
2. ThreadLocalRandom 进行随机

Eureka 启动时会拉取注册信息，之后每 30秒 拉取注册中心的注册信息

Feign 重试机制：出现异常时进行重试，重试多少次，下次重试时间是上次的 1.5 倍

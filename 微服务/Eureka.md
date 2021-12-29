# 服务端

使用 Maven 导入包：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

然后在 SpringBoot 程序上加入`@EnableEurekaServer`注解启动即可。


客户端会定时拉取注册中心的注册信息

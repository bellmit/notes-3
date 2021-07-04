# SpringBoot 自动生成 PID 文件

并不是在 application.yml 文件中配置了 `spring.pid.file=/Users/wu/web01.pid` 就可以了，**还得将监听器给加入进来**。

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.ApplicationPidFileWriter;

@SpringBootApplication
public class Web01Application {

    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(Web01Application.class);
        application.addListeners(new ApplicationPidFileWriter());
        application.run(args);
    }

}
```


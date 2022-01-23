# 一、什么是 SPI

SPI ，全称为 Service Provider Interface，是一种服务发现机制。它通过在ClassPath路径下的 META-INF/services 文件夹查找文件，自动加载文件里所定义的类。

这一机制为很多框架扩展提供了可能，比如在Dubbo、JDBC中都使用到了SPI机制。我们先通过一个很简单的例子来看下它是怎么用的。

# 二、使用

建立一个接口，并有两个实现类，如下：

```java
// 接口
public interface SpiService {
    void say();
}

// 实现类
public class CatSpiService implements SpiService {

    {
        System.out.println("实例化Cat");
    }

    @Override
    public void say() {
        System.out.println("Cat Say");
    }
}
public class DogSpiService implements SpiService {

    {
        System.out.println("实例化Dog");
    }

    @Override
    public void say() {
        System.out.println("Dog Say");
    }
}
```

然后在 resources 目录下新建 MATE-INF/services 目录，并新建一个文本文件，命名为 SpiService 这个接口的全类名，如：com.snailwu.spi.services.SpiService，然后打开该文件，并将两个实现类的全类名写在这里，每个实现类占一行，如：


```
com.snailwu.spi.services.impl.CatSpiService
com.snailwu.spi.services.impl.DogSpiService
```

测试代码：
```java
import com.snailwu.spi.services.SpiService;
import sun.misc.Service;

import java.util.Iterator;
import java.util.ServiceLoader;

/**
 * Hello world!
 * @author wu
 */
public class Main {
    public static void main(String[] args) {
        Iterator<SpiService> providers = Service.providers(SpiService.class);
        while (providers.hasNext()) {
            SpiService service = providers.next();
            System.out.println("next后");
            service.say();
        }

        ServiceLoader<SpiService> services = ServiceLoader.load(SpiService.class);
        Iterator<SpiService> iter = services.iterator();
        while (iter.hasNext()) {
            SpiService service = iter.next();
            System.out.println("获取后");
            service.say();
        }
    }
}
// 输出
实例化Cat
next后
猫叫
实例化Dog
next后
狗叫
实例化Cat
获取后
猫叫
实例化Dog
获取后
狗叫
```
可以发现，在调用 next() 方法获取实例对象的时候，会自动进行实例化操作，如果一个类的实例化操作很耗时，就比较麻烦，Dubbo 也是为了避免这种情况，开发了自己的 SPI 机制。

可以参考一下 next() 的源码，位置是：sun.misc.Service.LazyIterator#next。

# 应用场景

JDBC 数据库连接也是使用了 SPI 的机制，所以我们可以不写 Class.forName(com.mysql.cj.jdbc.Driver)。
可以参考源码：java.sql.DriverManager#loadInitialDrivers，以及 JDBC 包的 MATE-INF/services 目录。

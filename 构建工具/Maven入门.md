# 一、Maven配置阿里云源

阿里云源文档地址：[https://developer.aliyun.com/mvn/guide](https://developer.aliyun.com/mvn/guide)

编辑 `~/.m2/settings.xml` 文件：

```xml
<settings>
    <mirrors>
        <mirror>
            <id>aliyunmaven</id>
            <mirrorOf>central,jcenter</mirrorOf>
            <name>阿里云公共仓库</name>
            <url>https://maven.aliyun.com/repository/public</url>
        </mirror>
    </mirrors>
</settings>

```

这个 url 是 central 仓和 jcenter 仓的聚合仓，所以只需要拦截这两个仓库即可，其它仓库的包还是去 maven 仓库中心下载。

这是配置全局的拦截器，mirrorOf 是配置拦截哪些镜像。

# 二、配置私服地址

首先配置私服的账号和密码，这个必须在 `settings.xml` 中配置：

```xml
<settings>
    <servers>
        <server>
            <id>snailwu-demo-releases</id>
            <username>releases-1637456160565</username>
            <password>xxxxxxxxxx</password>
        </server>
        <server>
            <id>snailwu-demo-snapshots</id>
            <username>snapshots-1637456200873</username>
            <password>xxxxxxxxxx</password>
        </server>
    </servers>
</settings>

```

## 1、在项目中配置

```xml
<project>
    <!-- 配置拉取-->
    <repositories>
        <!-- 按照顺序进行包的拉取 -->
        <repository>
            <id>aliyun-public</id>
            <name>aliyun-maven</name>
            <url>https://maven.aliyun.com/repository/public</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>snailwu-demo-releases</id>
            <name>releases</name>
            <url>https://snailwu-maven.pkg.coding.net/repository/demo/releases/</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>snailwu-demo-snapshots</id>
            <name>snapshots</name>
            <url>https://snailwu-maven.pkg.coding.net/repository/demo/snapshots/</url>
            <releases>
                <enabled>false</enabled>
            </releases>
        </repository>
    </repositories>
</project>

```

## 2、配置在 `settings.xml` 中

配置在这里需要先声明一个 profile：

```xml
<settings>
    <profiles>
        <profile>
            <id>snailwu-demo</id>
            <repositories>
                <!-- 按照顺序进行包的拉取 -->
                <repository>
                    <id>aliyun-public</id>
                    <name>aliyun-maven</name>
                    <url>https://maven.aliyun.com/repository/public</url>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                </repository>
                <repository>
                    <id>snailwu-demo-releases</id>
                    <name>releases</name>
                    <url>https://snailwu-maven.pkg.coding.net/repository/demo/releases/</url>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                </repository>
                <repository>
                    <id>snailwu-demo-snapshots</id>
                    <name>snapshots</name>
                    <url>https://snailwu-maven.pkg.coding.net/repository/demo/snapshots/</url>
                    <releases>
                        <enabled>false</enabled>
                    </releases>
                </repository>
            </repositories>
        </profile>
    </profiles>
</settings>
```

# 三、发布配置

在项目中的 pom.xml 文件中进行配置：

```xml
<project>
    <!-- 配置 deploy -->
    <distributionManagement>
        <repository>
            <!--必须与 settings.xml 的 id 一致-->
            <id>snailwu-demo-releases</id>
            <name>releases</name>
            <url>https://snailwu-maven.pkg.coding.net/repository/demo/releases/</url>
        </repository>
        <snapshotRepository>
            <!--必须与 settings.xml 的 id 一致-->
            <id>snailwu-demo-snapshots</id>
            <name>snapshots</name>
            <url>https://snailwu-maven.pkg.coding.net/repository/demo/snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
</project>
```

Maven 会自动根据包的名字是否带有 SNAPSHOT 来区分发布到哪个仓库。

# 四、生成 javadoc 和 sources jar包

```xml
<project>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <!-- 生成 xxxxxx-sources.jar 包 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <executions>
                    <execution>
                        <id>attach-sources</id>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <!-- 生成 xxxxx-javadoc.jar 包 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-javadoc-plugin</artifactId>
                <executions>
                    <execution>
                        <id>attach-javadocs</id>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

# 五、命令

如果是 SpringBoot 项目，会自动给你生成 maven 的 jar 和命令。

此时，你的机器不需要安装 maven，直接使用 mvnw 这个脚本即可运行，比如 `./mvnw clean compile install deploy`。

**如何跳过测试？**

1. 在pom.xml 的 properties 下配置 `<maven.test.skip>true</maven.test.skip>`
2. 在执行 ./mvnw 命令时，添加 `-Dmaven.test.skip=true`，实际就是执行时动态传一个变量，1中是静态配置

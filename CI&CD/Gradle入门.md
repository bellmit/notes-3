配置文档：[https://docs.gradle.org/current/dsl/org.gradle.api.Project.html](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html)

插件文档：[https://docs.gradle.org/current/userguide/plugin_reference.html#plugin_reference](https://docs.gradle.org/current/userguide/plugin_reference.html#plugin_reference)

常用命令：

1. `./gradlew clean`
2. `./gradlew build`
3. `./gradlew publish`

# 一、切换阿里源

```Groovy
allprojects {
    repositories {
        maven {
            url 'https://maven.aliyun.com/repository/public/'
        }
        mavenLocal()
        mavenCentral()
    }
}
```

# 二、添加依赖 & 排除依赖

```Groovy
dependencies {
    implementation('org.springframework.boot:spring-boot-starter-web') {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-logging'
    }
    implementation 'cn.hutool:hutool-all:5.7.16'
    implementation 'org.springframework.boot:spring-boot-starter-log4j2'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

命令查看依赖项：gradle -q dependencies --configuration=compileClasspath

# 三、idea自动下载javadoc 包和 source 包

```Groovy
plugins {
    id 'idea'
}

idea {
    module {
        downloadJavadoc=true
        downloadSources=true
    }
}
```

# 四、生成本项目的 javadoc 包和source 包

```Bash
java {
    withJavadocJar()
    withSourcesJar()
}
```

# 五、发布 jar 都私服

```Bash
plugins {
    id 'maven-publish'
}

publishing {
    // 配置要发布的仓库
    repositories {
        mavenLocal()
        maven {
            // 判断是否是测试版本
            if (!version.endsWith('SNAPSHOT')) {
                // 此处直接使用 gradle.properties 中的 key 即可
                url = releasesRepoUrl
                // 私服认证需要
                credentials {
                    username = releasesGradleUsername
                    password = releasesGradlePassword
                }
            } else {
                url = snapshotsRepoUrl
                credentials {
                    username = snapshotsGradleUsername
                    password = snapshotsGradlePassword
                }
            }
        }
    }
    // 配置发布的项目信息
    publications {
        bootJava(MavenPublication) {
            artifact bootJar
            artifact sourcesJar
            artifact javadocJar
            pom {
                name = 'GradleDemoName'
                description = 'A concise description of my library'
                url = 'http://www.example.com/library'
                licenses {
                    license {
                        name = 'The Apache License, Version 2.0'
                        url = 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                    }
                }
                developers {
                    developer {
                        id = 'wql'
                        name = 'WuQinglong'
                        email = 'snail.wu@foxmail.com'
                    }
                }
            }
        }
    }
}

```

# 六、跳过测试

方法一：

```Bash
test {
    enabled = false
    useJUnitPlatform()
}
```

方法二：

`./gradlew build -x test`

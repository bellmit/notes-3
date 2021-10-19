官方Docker命令参考文档：https://docs.docker.com/engine/reference/commandline/cli/

查看机器上安装的 Docker 信息：docker info

# 一、仓库

仓库就是存放 Dcoker 镜像的地方。类似于 GitHub，Docker 有 DockerHub。也可以搭建私有的 DockerHub。

创建账号即可使用：https://registry.hub.docker.com/

# 二、镜像

官方Doc：https://docs.docker.com/engine/reference/commandline/images/

使用方式：docker image [OPTIONS] [REPOSITORY[:TAG]]

镜像分为顶层镜像和用户镜像：

顶层镜像：由 Docker 公司和由选定的能提供优质基础镜像的厂商（如 Fefora 团队提供了 fedora 镜像）管理，镜像名只包含仓库名，如：redis

用户镜像：基于顶层镜像的基础上构件的镜像，镜像名由用户名和仓库名组成，如：snailwu/redis

相关命令：

1. 列出本地所有镜像：docker image ls
2. 拉取镜像：docker image pull NAME[:TAG]
3. 删除本地镜像：docker image rm NAME[:TAG] | IMAGE_ID
4. 查看镜像的构建步骤：docker image history NAME[:TAG]
5. 给镜像重新打标签：docker image tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
6. 推送镜像到远程仓库：docker image push NAME[:TAG]

# 三、容器

容器是通过镜像构建出来的，一个容器可以看做是一个基于 Linux 的操作系统。

使用方式：docker ps [OPTIONS]

相关命令：

1. 列出运行时的容器：docker ps
2. 列出所有容器：docker ps -a
3. 启动容器：docker start CONTAINER [CONTAINER...]
4. 停止容器：docker stop CONTAINER [CONTAINER...]
5. 删除容器：docker rm CONTAINER [CONTAINER...]
6. 查看容器日志：docker logs -f -t CONTAINER
7. 查看容器的进程：docker top CONTAINER

# 四、Dockerfile

它是一个文本文件，有自己的语法，通过 Dockerfile 可以构建自定义的镜像。

构件镜像的两种方式：

1. 通过基础镜像运行一个容器，然后通过 docker commit 进行构建。（docker commit --author='Wu' --message='Msg' CONTAINER NAME[:TAG]）
2. 通过 Dockerfile 进行构建。

Dockerfile 由 docker build 进行使用，进行镜像的构建。

规则：Dockerfile 由一系列的指令和参数组成。每条指令，如FROM，都必须为大写字母，且后面要跟随一个参数：FROM ubuntu:18.04。Dockerfile 会按照顺序从上到下执行，所以应该根据需要合理安排指令的顺序。Dockerfile 支持注解，以 # 开头的行都会被认为是注解。

每条指令都会创建一个新的镜像层并对镜像进行提交。

（1）、FROM

每个 Dockerfile 的第一条指令都应该是 FROM。FROM 指令指定一个已经存在的镜像，后续的指令都将基于该镜像进行，这个镜像被称为基础镜像（base image）。

FROM “ubuntu:18.04”

（2）、MAINTAINER

这条指令告诉 Docker 该镜像的作者是谁，以及作者的电子邮件地址。

MAINTAINER WuQinglong “[snail.wu@foxmail.com](mailto:snail.wu@foxmail.com)”

（3）、RUN

这条指令会在当前的镜像中运行指定的命令。每条 RUN 指令都会创建一个新的镜像层，为了减少镜像的层数，通常将指令合并为一条指令，如：RUN apt update && apt install wget

该指令默认使用命令包装器 /bin/sh -c 来执行，这属于 shell 格式。也可以使用 exec 格式进行运行 RUN 指令，如：RUN [“apt”, “update”]。

RUN apt update && apt install wget

（4）、EXPOSE

这条指令只是告诉 Docker 容器内的应用程序将会使用容器的指定端口。Docker 并不会自动打开该端口，而是要在 docker run 的时候进行指定。

EXPOSE 8080

（5）、CMD

用于指定一个容器启动时要运行的命令。类似于 RUN 指令，不同的是 RUN 指令是指定构建时要运行的命令，而 CMD 是指定容器被启动时要运行的命令，与 docker run 命令启动容器时指定要运行的命令非常相似。

使用 docker run 命令可以覆盖 CMD 命令。

注意：Dockerfile 中只能指定一个 CMD 命令。如果指定多条，也只有最后一条被使用。

CMD [“echo”, “Hello Docker.”]

（6）、ENTRYPOINT

这条指令与 CMD 指令非常相似，但是有什么区别呢？CMD 指定的指令可以在容器运行的时候全部被覆盖，而 ENTRYPOINT 则不会被轻易覆盖（也可以使用 —entrypoint 来覆盖），docker run 命令行中指定的任何参数都会被当做参数再次传递给 ENTRYPOINT 。

如：使用 ENTRYPOINT 启动 nginx

ENTRYPOINT [ “/usr/sbin/nginx” ]

CMD [“-h”]

如果在启动容器时不指定任何参数，则容器默认会以 /usr/sbin/nginx -h 的方式启动。当我们运行 docker run -it NAME -g “daemon off;” 时，容器会以 /usr/sbin/nginx -g “daemon off;” 的方式启动。

（7）、WORKDIR

用来从镜像创建一个新容器时，在容器内部设置一个工作目录，CMD 或 ENTRYPOINT 指定的程序会在这个目录下执行。

可以使用 -w 参数在 docker run 的时候进行覆盖。

WORKDIR /opt/

（8）、ENV

用来在镜像构建过程中设置环境变量。可以在后续的任何 RUN 指令中使用。

如果需要，可以通过在环境变量前加上一个反斜杠来进行转义。

ENV JAVA_HOME /opt/java/

（9）、USER

用来指定该镜像会以什么样的用户去运行。

默认使用 root 用户运行。

USER nginx

（10）、VOLUMN

用来向基于镜像创建的容器添加卷，一个卷时可以存在于一个或多个容器内的特定的目录，这个目录可以绕过联合文件系统，并提供如下共享数据或者对数据进行持久化的功能。

- 卷可以在容器内共享。
- 一个容器可以不是必须和其它容器共享卷。
- 对卷的修改时实时生效的。
- 对卷的修改不会对更新镜像产生影响。
- 卷会一直存在知道没有任何容器使用它。

VOLUMN [“/data/log”]

这条指令会为基于此镜像的所有容器创建一个名为 /data/log 的挂载点。也可以指定多个挂载点 VOLUMN [“/data/log”, “/data/app”]

（11）、ADD

用于将构建环境下的文件和目录复制到镜像中。

只能对构建上下文或环境中文件名或者目录进行复制，不能对构建目录或者上下文之外的文件进行 ADD 操作。

对 tar 归档文件进行 ADD 时，Docker 会自动进行解压。

也可以 ADD 一个网络文件：ADD https://www.xxx.com/latest.zip /opt/latest.zip。对于网络文件，还不支持自动解压。

（12）、COPY

与 ADD 非常类型，唯一的区别就是不会对文件进行提取和解压的工作，仅仅是进行复制。

# 五、Dockerfile 自动构建

DockerHub 支持自动构件 GitHub 中的 Dockerfile。

大概流程就是：

1. GitHub 注册一个 push 钩子。
2. DockerHub 收到 push 通知，触发构件动作。

# Dockerfile构建镜像-Java

> ex: 构建一个java的jar运行

### Dockerfile

```shell
#基于java8
FROM java:8
 
#创建一个目录存放jar包(在容器里面创建目录)
RUN mkdir -p /huanglei/jar/demo/config /huanglei/jar/demo/logs
 
#复制jar包以及相关配置文件(复制jar包到容器的根目录)  xx.jar 和 Dockerfile 在同一级
COPY demo-0.0.1-SNAPSHOT.jar /demo-0.0.1-SNAPSHOT.jar
 
#添加进入docker容器后的目录
WORKDIR /huanglei/jar/demo
 
#配置项目端口
CMD ["--server.port=7001"]
 
#对外暴露的端口号
# [注：EXPOSE指令只是声明容器运行时提供的服务端口，给读者看有哪些端口，在运行时只会开启程序自身的端口！！]
EXPOSE 7001
 
#修改文件的创建修改时间
RUN bash -c 'touch /demo-0.0.1-SNAPSHOT.jar'
 
#运行脚本，启动springboot项目，这里我们指定加载配置文件的位置，并且通过数据卷挂载同步到容器中
ENTRYPOINT ["java","-jar","/demo-0.0.1-SNAPSHOT.jar","-Dspring.config.location=/huanglei/jar/demo/config/application.properties --logging.config=/huanglei/jar/demo/logs/logback.xml > /huanglei/jar/demo/logs/demo.log 2>&1 &"]
```

**关键字：**

| 关键字      | 作用                     | 备注                                                         |
| ----------- | ------------------------ | ------------------------------------------------------------ |
| FROM        | 指定父镜像               | 指定dockerfile基于哪个image构建                              |
| MAINTAINER  | 作者信息                 | 用来标明这个dockerfile谁写的                                 |
| LABEL       | 标签                     | 用来标明dockerfile的标签 可以使用Label代替Maintainer 最终都是在docker image基本信息中可以查看 |
| RUN         | 执行命令                 | 执行一段命令 默认是/bin/sh 格式: RUN command 或者 RUN ["command" , "param1","param2"] |
| CMD         | 容器启动命令             | 提供启动容器时候的默认命令 和ENTRYPOINT配合使用.格式 CMD command param1 param2 或者 CMD ["command" , "param1","param2"] |
| ENTRYPOINT  | 入口                     | 一般在制作一些执行就关闭的容器中会使用                       |
| COPY        | 复制文件                 | build的时候复制文件到image中                                 |
| ADD         | 添加文件                 | build的时候添加文件到image中 不仅仅局限于当前build上下文 可以来源于远程服务 |
| ENV         | 环境变量                 | 指定build时候的环境变量 可以在启动的容器的时候 通过-e覆盖 格式ENV name=value |
| ARG         | 构建参数                 | 构建参数 只在构建的时候使用的参数 如果有ENV 那么ENV的相同名字的值始终覆盖arg的参数 |
| VOLUME      | 定义外部可以挂载的数据卷 | 指定build的image那些目录可以启动的时候挂载到文件系统中 启动容器的时候使用 -v 绑定 格式 VOLUME ["目录"] |
| EXPOSE      | 暴露端口                 | 定义容器运行的时候监听的端口 启动容器的使用-p来绑定暴露端口 格式: EXPOSE 8080 或者 EXPOSE 8080/udp |
| WORKDIR     | 工作目录                 | 指定容器内部的工作目录 如果没有创建则自动创建 如果指定/ 使用的是绝对地址 如果不是/开头那么是在上一条workdir的路径的相对路径 |
| USER        | 指定执行用户             | 指定build或者启动的时候 用户 在RUN CMD ENTRYPONT执行的时候的用户 |
| HEALTHCHECK | 健康检查                 | 指定监测当前容器的健康监测的命令 基本上没用 因为很多时候 应用本身有健康监测机制 |
| ONBUILD     | 触发器                   | 当存在ONBUILD关键字的镜像作为基础镜像的时候 当执行FROM完成之后 会执行 ONBUILD的命令 但是不影响当前镜像 用处也不怎么大 |
| STOPSIGNAL  | 发送信号量到宿主机       | 该STOPSIGNAL指令设置将发送到容器的系统调用信号以退出。       |
| SHELL       | 指定执行脚本的shell      | 指定RUN CMD ENTRYPOINT 执行命令的时候 使用的shell            |

### 构建镜像

```shell
# 构建镜像
# -f：指定Dockerfile文件路径
# -t：镜像命名
# --no-cache：构建镜像时不使用缓存
# 最后有一个点 “.”：当构建的时候，由用户指定构建镜像的上下文环境路径，然后将此路径下的所有文件打包上传给Docker引擎，引擎内将这些内容展开后，就能获取到所有指定上下文中的文件了。
docker build -f /huanglei/jar/demoDockerfile -t demo-test:v1.0 --no-cache .
```

### 运行

在宿主机创建相关目录，并且上传配置文件

```shell
docker run -dit --name demo -p 7001:7001 \
-v /huanglei/jar/demo/config:/huanglei/jar/demo/config \
-v /huanglei/jar/demo/logs:/huanglei/jar/demo/logs \
--privileged=true \
demo:v1.0

```
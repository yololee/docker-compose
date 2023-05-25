# docker-compose

## 一、安装

```shell
$curl -L https://github.com/docker/compose/releases/download/1.17.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
$chmod +x /usr/local/bin/docker-compose
#查看版本
$docker-compose version

```

docker-compose将所管理的容器分为3层结构：project service container

docker-compose.yml组成一个project，project里包括多个service，每个service定义了容器运行的镜像>（或构建镜像），网络端口，文件挂载，参数，依赖等，每个service可包括同一个镜像的多个容器实例

即 project 包含 service ，service 包含 container

## 二、docker-compose配置常用字段

| 字段           | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| build          | 指定Dockerfile文件名（要指定的Dockerfile文件需要在build标签的子级标签中用dockerfile标签指定） |
| dockerfile     | 构建镜像上下文路径                                           |
| context        | 可以是dockerfile路径，或者是执行git 仓库的url地址            |
| image          | 指定镜像（已存在）                                           |
| command        | 执行命令，会覆盖容器启动后默认执行的命令（会覆盖Dockerfile的CMD指令） |
| container_name | 指定容器名称，由于容器名称是唯一的，如果指定自定义名称，则无法scale指定容器数量 |
| deploy         | 指定部署和运行服务相关配置，只能在Swarm模式使用              |
| environment    | 添加环境变量                                                 |
| networks       | 加入网络，引用顶级networks下条目                             |
| network-mode   | 设置容器的网络模式                                           |
| ports          | 暴露容器端口，与-p 相同，但是端口不能低于60                  |
| volumes        | 挂载一个宿主机目录或命令卷到容器，命名卷要在顶级volumes 定义卷名称 |
| volumes_from   | 从另一个服务或容器挂载卷，可选参数 :ro 和 :rw，（仅版本‘2’支持） |
| hostname       | 在容器内设置内核参数                                         |
| links          | 连接到另一个容器，- 服务名称[ : ]                            |
| privileged     | 用来给容器root权限，注意是不安全的，true                     |
| depends_on     | 此标签用于解决容器的依赖，启动先后问题。如启动应用容器，需要先启动数据库容器php:depends_on:- apache- mysql |
| restart        | 重启策略，定义是否重启容器；1.no，默认策略，在容器退出时不重启容器。2.on-failure，在容器非正常退出时（退出状态非0），才会重启容器。3.on-failure：3，在容器非正常退出时重启容器，最多重启3次。4.always，在容器退出时总是重启容器。5.unless-stopped，在容器退出时总是重启容器，但是不考虑在Docker守护进程启动时就已经停止了的容器 |

```yaml
version: '3'
services:
  back:
    image: backService:1.0
    container_name: back
    environment:
      - name=tom
      - DB_PATH=jdbc:sqlite:/data/ns.db
    restart: always
    privileged: true
    ports:
      - "9000:9000"
    networks:
      - "net"
    volumes:
      - "/root/k3s.kube.config:/k3s.kube.config"
      - "/root/data:/data"
      - "/etc/network/interfaces:/etc/network/interfaces"
  front:
    image: front:1.0
    container_name: front
    restart: always
    ports:
      - "10087:80"
    networks:
      - "net"
    volumes:
      - "/root/nginx.conf:/etc/nginx/nginx.conf"
networks:
  net:
    driver: bridge
```

**version：指定 docker-compose.yml 文件的写法格式**

**services：多个容器集合environment,环境变量配置，可以用数组或字典两种方式**

```yaml
environment:
    RACK_ENV: "development"
    SHOW: "ture"
-------------------------
environment:
    - RACK_ENV="development"
    - SHOW="ture"
```

**image：指定服务所使用的镜像**

```yaml
version: '2'
services:
  redis:
    image: redis:alpine
```

**expose：定义容器用到的端口（一般用来标识镜像使用的端口，方便用ports映射）**

```yaml
expose:
    - "3000"
    - "8000"
```

**ports：定义宿主机端口和容器端口的映射，可使用宿主机IP+宿主机端口进行访问 宿主机端口:容器端口**

```yaml
ports:   # 暴露端口信息  - "宿主机端口:容器暴露端口"
- "5000"
- "8081:8080"
```

**volumes：卷挂载路径，定义宿主机的目录/文件和容器的目录/文件的映射 **宿主机路径:容器路径****

```yaml
volumes:
  - "/var/lib/mysql"
  - "hostPath:containerPath"
  - "root/configs:/etc/configs"
```

**depend_on: 规定service加载顺序，例如数据库服务需要在后台服务前运行**

**extra_hosts：类似于docker里的--add-host参数 配置DNS域名解析(域名和IP的映射)**

```yaml
extra_hosts:
 - "googledns:8.8.8.8"
 - "dockerhub:52.1.157.61"
```

相当于在容器种的/etc/hosts文件里增加

```hosts
8.8.8.8 googledns
52.1.157.61 dockerhub
```

**restart: always ：配置重启，docker每次启动时会启动该服务**

```yaml
restart: always
```

**privileged: true ：开启特权模式**

```yaml
privileged: true
```

**networks：**

可参考：https://www.cnblogs.com/jsonhc/p/7823286.html

```yaml
version: '3'
services:
  front:
    image: front
    container_name: front
    depends_on:
      - php
    ports:
      - "80:80"
    networks:
      - "net1"
    volumes:
      - "/www:/usr/local/nginx/html"
  back:
    image: back
    container_name:back
    expose: 
      - "9000"
    networks:
      - "net1"
    volumes:
      - "/www:/usr/local/nginx/html"

networks:
  net1:
    driver: bridge
```

> front里可以直接用back替换IP访问back

**logging：日志服务**

driver：默认json-file，可选syslog

```yaml
logging:
  driver: syslog
  options:
    syslog-address: "tcp://192.168.0.42:123"
```

**network_mode：设置网络模式**

```yaml
network_mode: "bridge"
network_mode: "host"
network_mode: "none"
network_mode: "service:[service name]"
network_mode: "container:[container name/id]"
```

bridge：默认，需要单独配置ports映射主机port和服务的port，并且开启了容器间通信

host：和宿主机共享网络，比如service是8081端口，无需配置ports，直接可以用主机IP:8081访问

## 三、docker-compose命令

> 以下都需要在docker-compose.yml所在目录下执行，且名字就是默认的docker-compose.yml，否则需要加上 >-f yml地址
>
> Eg： docker-compose -f /usr/docker/docker-compose1.yml ps

| 字段                                              | 描述                                                         |
| ------------------------------------------------- | ------------------------------------------------------------ |
| **docker-compose pull**                           | 拉取服务里定义的镜像                                         |
| **docker-compose ps**                             | 列出project所有运行容器                                      |
| **docker-compose build**                          | 构建/重新构建所有镜像                                        |
| **docker-compose start [serviceName]**            | 启动已存在但停止的所有service，(可选）serviceName：表示启动某一个service |
| **docker-compose up -d（相当于 build + start ）** | 构建（容器）并启动（容器）整个project的所有service -d：后台进程 |
| **docker-compose up --scale**                     | 指定服务运行的容器个数（如果服务有对外的端口就不能指定多个容器，因为端口已经被占用）docker-compose up -d --scale web=1 --scale redis=2 |
| **docker-compose stop [serviceName]**             | 停止运行的service，（可选）serviceName：表示停止某一个service |
| **docker-compose rm -f [serviceName]**            | 删除已停止的所有service                                      |
| **docker-compose down -v（相当于 stop + rm ）**   | 停止并移除整个project的所有services，-v ：删除挂载卷和volunme的链接 |
| **docker-compose logs [serviceName]**             | 查看服务内所有容器日志输出，加上serviceName表示输出某一个service的日志，-f：实时输出日志 |
| **docker-compose run service command**            | 在某个服务上运行命令，Eg：docker-compose run web ping www.baidu.com |
| **docker-compose exec [serviceName] sh**          | 进入到某个容器                                               |
| **docker-compose restart [serviceName]**          | 重启服务                                                     |
| **docker-compose config**                         | 验证和查看compose文件                                        |
| **docker-compose images**                         | 列出所用的镜像                                               |
| **docker-cpmpose scale**                          | 设置服务个数 Eg：docker-compose scale web=2 worker=3         |
| **docker-compose pause [serviceName]**            | 暂停服务                                                     |
| **docker-compose unpause [serviceName]**          | 恢复服务                                                     |
# docker-compose添加外部创建好的网络

### 创建网络

在Swam主节点创建overlay网络段：为了保持后续创建的容器位于一个网络内，便于相互通信

–attachable ：表明这个网络是可以被container所加入，如果没有此参数，那么新增的这个自定义网络后只能被service使用，不能被普通容器使用。

```shell
docker network create --driver overlay --attachable mysql_network
```

### 执行命令查看网络是否创建成功

```shell
docker network ls
```

### 使用创建好的网络

```yml
version: '3'
services:
  mysql:
    image: mysql:5.7
    volumes:
      - "/root/mye/mysql/my.cnf:/etc/mysql/my.cnf"
      - "/root/mye/mysql/data:/var/lib/mysql"
      - "/root/mye/mysql/log/mysql/error.log:/var/log/mysql/error.log"
    environment:
      TZ: Asia/Shanghai
      LANG: en_US.UTF-8
      MYSQL_ROOT_PASSWORD: root
    ports:
      - "3306:3306"
    networks:
      - "mysql_network"
    deploy:
      replicas: 1 #可以指定该服务运行的容器数量
      resources:
        limits:
          cpus: "2"
          memory: 2048M
      restart_policy:
        condition: on-failure
      placement: #指定约束和偏好设置，这里指定该服务在manager节点启动
        constraints: [node.role == worker]
networks:
  mysql_network:
    external: true
```

### 主要核心配置是

```yml
networks:
  mysql_network:
    external: true
```

该部分表示使用外部网络demo，外部为true
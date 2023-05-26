# docker-stack

## 一、介绍

### 区别

Docker Compose，它是用来进行一个完整的应用程序相互依赖的多个容器的编排的，但是缺点是不能在分布式多机器上使用

Docker swarm，它构建了docker集群，并且可以通过docker service在不同集群节点上运行容器服务，但是缺点是不能同时编排多个服务

Docker Stack，它用于向swarm集群部署完整的应用程序堆栈，可以在分布式多机器上同时编排多个有依赖关系的服务

### stack介绍

Stack 能够在单个声明文件中定义复杂的多服务应用，还提供了简单的方式来部署应用并管理其完整的生命周期：初始化部署 -> 健康检查 -> 扩容 -> 更新 -> 回滚，以及其他功能！可以简单地理解为Stack是集群下的Compose

Docker Stack 编排依赖于声明文件，其实也就是Docker Compse文件，不过Stack对于Compose文件有一个要求，文件的规范版本（顶层关键字`version`）必须是`“3.0”或以上`，而且一些Docker Compse文件中的关键字不受支

- build
- cgroup_parent
- container_name
- devices
- tmpfs
- external_links
- links
- network_mode
- restart
- security_opt
- userns_mode

由于 build 关键字在 Stack 中不受支持，不能在编排的过程中构建镜像，docker stack部署用到的镜像必须是已经构建发布好并且发布到 docker 仓库的，在服务编排过程各个节点中直接从 docker 仓库拉取

## 二、案例

```yml
version: "3"
services:
  nginx:
    image: nginx
    ports:
      - 8888:80
    deploy:
      mode: replicated
      replicas: 3
  portainer:
    image: portainer/portainer
    ports:
      - "9000:9000"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:   # docker stack 部署docker-compose 必要参数
      mode: replicated
      # mode: global    # 每个节点都会创建一份
      replicas: 1   # 几个副本
      placement:
        constraints: [node.role == manager]
```

> 注意：挂载的时候在每个节点都需要有挂载的文件，不然会出现发布成功，实例没有运行起来

```yml
# nginx发布三个实例
deploy:
  mode: replicated
  replicas: 3
  
# nginx在每个节点上发布一个实例
deploy:   
  mode: global
  
# nginx指定在管理节点发布一个实例
deploy:   # docker stack 部署docker-compose 必要参数
  mode: replicated
  replicas: 1   # 几个副本
  placement:
    constraints: [node.role == manager]

# nginx指定在工作节点发布一个实例
deploy:   # docker stack 部署docker-compose 必要参数
  mode: replicated
  replicas: 1   # 几个副本
  placement:
    constraints: [node.role == worker]
```

## 三、命令

```shell
# 发布 myapp为容器前缀
docker stack deploy -c docker-compose.yml myapp

# 检查状态
docker stack ls

# 内部服务的详细信息
docker stack services myapp

# 每个服务容器的详细信息
docker stack ps myapp

# 删除
docker stack rm myapp
```


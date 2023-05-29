### docker-compose-portainer.yml

#### 单节点

> yml配置可参考 [docker-compose.yml](https://gitee.com/zhengqingya/docker-compose/blob/master/docker-compose.yml)

```shell
# 等同于：docker run -d -p 9000:9000 --restart=always --name portainer -v /var/run/docker.sock:/var/run/docker.sock registry.cn-hangzhou.aliyuncs.com/zhengqing/portainer:1.24.1
version: '3'
services:
  portainer:
    image: portainer/portainer:1.24.1  
    container_name: portainer                                            # 容器名为'portainer'
    restart: always                                                      # 指定容器退出后的重启策略为始终重启
    volumes:                                                             # 数据卷挂载路径设置,将本机目录映射到容器目录
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./portainer/data:/data"
    environment:                        # 设置环境变量,相当于docker run命令中的-e
      TZ: Asia/Shanghai
      LANG: en_US.UTF-8
    ports:                              # 映射端口
      - "9000:9000"
```

运行

```shell
# -p：项目名称
# -f：指定docker-compose.yml文件路径
# -d：后台启动
docker-compose -f docker-compose-portainer.yml -p portainer up -d
```

#### 集群

在Swam主节点创建overlay网络段：为了保持后续创建的全局服务portainer_agent与Portainer容器位于一个网络内，便于相互通信

–attachable ：表明这个网络是可以被container所加入，如果没有此参数，那么新增的这个自定义网络后只能被service使用，不能被普通容器使用。

```yml 
version: '3.3'

services:
  agent:
    image: portainer/agent
    labels:
      service: portainer_agent
    logging:
      options:
        labels: "service"
        max-size: "100m"
        max-file: "4"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /root/mye/portainer/data/volumes:/var/lib/docker/volumes
    environment:                               
      AGENT_CLUSTER_ADDR: tasks.agent
    networks:
      - portainer_net
    deploy:
      mode: global
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      placement:
        constraints: [node.platform.os == linux]

  portainer:
    image: portainer/portainer
    labels:
      service: portainer
    logging:
      options:
        labels: "service"
        max-size: "100m"
        max-file: "4"
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      - "19000:9000"
    volumes:
      - /root/mye/portainer/data/portainer_data:/data
    networks:
      - portainer_net
    deploy:
      mode: replicated
      replicas: 1
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      placement:
        constraints: [node.role == manager] 

networks:
  portainer_net:
    driver: overlay
    attachable: true
```

> 注意：首先每一个节点上都需要有挂载的目录,不然会挂载失败
>
> mkdir -p /root/mye/portainer/data/volumes
>
> mkdir -p /root/mye/portainer/data/portainer_data

发布，在当前docke-compose文件目录下执行

```shell
# myportainer这个是名字前缀
docker stack deploy -c docker-compose-portainer.yml myportainer

# 查看部署是否成功
docker stack ls

# 查看具体的服务是否启动成功
docker stack services myportainer

# 查看当个服务的运行情况
docker service ps myportainer_agent
```


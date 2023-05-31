# docker swarm 环境上搭建RabbitMQ

| ip             | 主机名    |
| -------------- | --------- |
| 116.211.105.10 | `worker1` |
| 116.211.105.11 | `worker2` |
| 116.211.105.12 | `worker3` |
| 116.211.105.13 | `manager` |

![](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230531162348400.png)

在管理节点的服务器上拉取文件，然后修改其中的docker-compose.yml文件

```shell
git clone https://github.com/liangxiaobo/RabbitMQ-Docker-cluster.git
```

执行`rabbitmq-cluster-network.sh`创建网络

```shell
docker network create --driver overlay --attachable rabbitmq-cluster
```

安装自己的需求修改`docker-compose-rabbitmq-cluster.yml`

使用了docker-compose编排3.3版本的`docker config`就不需要在每台mq的机器上放置两个配置文件

```yml
version: "3.3"
services:
  rabbit1:
    image: rabbitmq:3-management
    hostname: rabbit1
    environment:
      RABBITMQ_ERLANG_COOKIE: "secret string"
      RABBITMQ_NODENAME: rabbit1
    depends_on:
      - rabbit3
    ports:
      - "4369:4369"
      - "5671:5671"
      - "5672:5672"
      - "15671:15671"
      - "15672:15672"
      - "25672:25672"
    configs:
      - source: rabbitmq_config
        target: /etc/rabbitmq/rabbitmq.config
      - source: definitons_json
        target: /etc/rabbitmq/definitions.json
    networks:
      - rabbitmq-cluster
    deploy:
      replicas: 1
      placement:
        constraints: 
          - node.role == worker
          - node.hostname == worker1   
  rabbit2:
    image: rabbitmq:3-management
    hostname: rabbit2
    environment:
      RABBITMQ_ERLANG_COOKIE: "secret string"
      RABBITMQ_NODENAME: rabbit2
    depends_on:
      - rabbit3
    configs:
      - source: rabbitmq_config
        target: /etc/rabbitmq/rabbitmq.config
      - source: definitons_json
        target: /etc/rabbitmq/definitions.json
    networks:
      - rabbitmq-cluster
    deploy:
      replicas: 1
      placement:
        constraints: 
          - node.role == worker
          - node.hostname == worker2  
  rabbit3:
    image: rabbitmq:3-management
    hostname: rabbit3
    environment:
      RABBITMQ_ERLANG_COOKIE: "secret string"
      RABBITMQ_NODENAME: rabbit3
    configs:
      - source: rabbitmq_config
        target: /etc/rabbitmq/rabbitmq.config
      - source: definitons_json
        target: /etc/rabbitmq/definitions.json
    networks:
      - rabbitmq-cluster
    deploy:
      replicas: 1
      placement:
        constraints: 
          - node.role == worker 
          - node.hostname == worker3 

configs:
  rabbitmq_config:
    file: rabbitmq.config
  definitons_json:
    file: definitions.json

networks:
  rabbitmq-cluster:
    external: true
```

执行部署

```shell
docker stack deploy -c docker-compose-rabbitmq.yml rabbitmq
```

查看服务

![image-20230531163204115](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230531163204115.png)

成功后访问 http://集群的任一:15672 默认用户名和密码都是guest

![image-20230531163314645](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230531163314645.png)

> 参考文档
>
> [RabbitMQ-Docker-cluster](https://github.com/liangxiaobo/RabbitMQ-Docker-cluster)
>
> [RabbitMQ部署在DockerSwarm集群](https://www.jianshu.com/p/9243a9c020a3)


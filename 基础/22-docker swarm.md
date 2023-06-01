# Docker Swarm

- `Docker Compose`: 在单机上创建多个容器
- `Docker Swarm`: 在多服务器上创建容器集群服务 => 更有利于微服务的部署

##  环境准备

| 机器           | 说明               |
| -------------- | ------------------ |
| 192.168.10.26  | Node1（huanglei2） |
| 192.168.10.110 | Node2（huanglei）  |

### docker安装

[docker安装](https://gitee.com/huanglei1111/docker-compose/blob/master/%E5%9F%BA%E7%A1%80/01-Docker%20%E5%AE%89%E8%A3%85%E5%92%8C%E5%8D%B8%E8%BD%BD.md)

###  配置docker远程连接2375端口

> Swarm是通过监听2375端口进行通信的

[Docker配置远程连接2375端口](https://gitee.com/huanglei1111/docker-compose/blob/master/%E5%9F%BA%E7%A1%80/09-Docker%E9%85%8D%E7%BD%AE%E8%BF%9C%E7%A8%8B%E8%BF%9E%E6%8E%A52375%E7%AB%AF%E5%8F%A3.md)

##  Swarm集群部署

> 防火墙开启以下端口或者关闭防火墙：
>
> - TCP端口2377，用于集群管理通信；
> - TCP和UDP端口7946，用于节点之间通信；
> - UDP端口4789，用于覆盖网络。

###  初始化Swarm

在node1 和node2拉取swarm镜像

```shell
docker pull swarm
```

![image-20230525143336846](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230525143336846.png)

![image-20230525143417988](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230525143417988.png)

```shell
# `--advertise-addr`参数表示其它swarm中的worker节点使用此ip地址与manager联系。命令的输出包含了其它节点如何加入集群的命令。
docker swarm init --advertise-addr 192.168.10.26

# 查看管理节点的 token
docker swarm join-token manager
# 查看工作节点的 token
docker swarm join-token worker
```

在node1 初始化swarm ，在node2执行下面红色框框的内容

![image-20230525142903006](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230525142903006.png)

### 查看节点

```shell
# AVAILABILITY列说明：
#                     Active -> 调度程序可以将任务分配给节点。
#                     Pause -> 调度程序不会将新任务分配给节点，但现有任务仍在运行。
#                     Drain -> 调度程序不会向节点分配新任务。调度程序关闭所有现有任务并在可用节点上调度它们。
# MANAGER STATUS列说明：
#                     没有值 -> 表示不参与群管理的工作节点。
#                     Leader -> 该节点是使得群的所有群管理和编排决策的主要管理器节点。
#                     Reachable -> 节点是管理者节点正在参与Raft共识。如果领导节点不可用，则该节点有资格被选为新领导者。
#                     Unavailable -> 节点是不能与其他管理器通信的管理器。如果管理器节点不可用，您应该将新的管理器节点加入群集，或者将工作器节点升级为管理器。
docker node ls
# 查看节点详情
docker node inspect 节点名称|节点ID
```

![image-20230525142836673](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230525142836673.png)

###  添加集群节点

在节点二执行添加工作节点到swarm集群

![image-20230525143011679](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230525143011679.png)

然后在管理节点查看swarm集群

![image-20230525143120427](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230525143120427.png)

###  删除节点

#### 删除管理节点

```shell
# 删除节点之前需要先将该节点的 AVAILABILITY 改为 Drain。其目的是为了将该节点的服务迁移到其他可用节点上，确保服务正常。最好检查一下容器迁移情况，确保这一步已经处理完成再继续往下。
docker node update --availability drain 节点名称|节点ID

# 节点降级，由管理节点降级为工作节点
docker node demote 节点名称|节点ID

# 在已经降级为 Worker 的节点中运行以下命令，离开集群
docker swarm leave

# 删除节点（-f强制删除）
docker node rm 节点名称|节点ID
```

#### 删除工作节点

```shell
# 删除节点之前需要先将该节点的 AVAILABILITY 改为 Drain。其目的是为了将该节点的服务迁移到其他可用节点上，确保服务正常。最好检查一下容器迁移情况，确保这一步已经处理完成再继续往下。
docker node update --availability drain 节点名称|节点ID

# 在准备删除的 Worker 节点中运行以下命令，离开集群。
docker swarm leave

# 删除节点（-f强制删除）
docker node rm 节点名称|节点ID
```

##  Swarm实战

###  创建服务

```shell
# --replicas：指定一个服务有几个实例运行
# --name：服务名称
docker service create --replicas 2 --name mynginx -p 80:80 nginx

# 查看所有服务
docker service ls

# 查看服务详情
docker service inspect mynginx

# 查看服务运行在哪些节点上
docker service ps mynginx

# 测试删除其中1个nginx容器后，会自动创建1个新的nginx来保证一直运行上面指定的2个实例
docker rm -f 容器ID/容器名
```

![image-20230525144130311](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230525144130311.png)

访问 `集群任意IP:80`

![image-20230525144309926](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230525144309926.png)

###  弹性服务（动态扩缩容）

```shell
# 扩容到5个
docker service scale mynginx=5

# 查看服务运行在哪些节点上
docker service ps mynginx

# 缩容到3个
docker service update --replicas 3 mynginx

# 查看服务运行在哪些节点上
docker service ps mynginx
```

![image-20230525144540263](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230525144540263.png)

![image-20230525144633409](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230525144633409.png)

###  删除服务

```shell
docker service rm mynginx

# 查看所有服务
docker service ls
```

###  滚动更新&版本回滚

```shell
# 创建 5 个副本，每次更新 2 个，更新间隔 10s，20% 任务失败继续执行，超出 20% 执行回滚，每次回滚 2 个
#   --update-delay：定义滚动更新的时间间隔；
#   --update-parallelism：定义并行更新的副本数量，默认为 1；
#   --update-failure-action：定义容器启动失败之后所执行的动作；
#   --rollback-monitor：定义回滚的监控时间；
#   --rollback-parallelism：定义并行回滚的副本数量；
#   --rollback-max-failure-ratio：任务失败回滚比率，超过该比率执行回滚操作，0.2 表示 20%。
docker service create --replicas 5 --name redis \
    --update-delay 10s \
    --update-parallelism 2 \
    --update-failure-action continue \
    --rollback-monitor 20s \
    --rollback-parallelism 2 \
    --rollback-max-failure-ratio 0.2 \
    redis:5
    
# 查看服务运行情况
docker service ps redis
```

![image-20230525145020638](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230525145020638.png)

#### 滚动更新

```shell
docker service update --image redis:6 redis

# 查看服务运行情况
docker service ps redis
```

![image-20230525145213234](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230525145213234.png)

#### 版本回滚

回滚服务，只能回滚到上一次操作的状态，并不能连续回滚到指定操作

```shell
docker service update --rollback redis
```

![image-20230525145348808](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230525145348808.png)

##  Swarm常用命令	

### 管理配置文件

```shell
# 查看已创建配置文件
docker config ls

# 将已有配置文件添加到docker配置文件中
docker config create docker 配置文件名 本地配置文件

# 1、创建配置文件
docker config create nginx2_config nginx2.conf

# 2、删除旧配置文件
docker service update --config-rm ce_nginx_config 服务名

#3、添加新配置文件到服务
docker service update --config-add src=nginx2_config,target=/etc/nginx/nginx.conf ce_nginx

# 删除配置文件
docker service update --config-rm 配置文件名称 服务名
```

### 集群管理

```shell
# 初始化集群
docker swarm init 	
 
# 查看工作节点的 token
docker swarm join-token worker 	
 
# 查看管理节点的 token
docker swarm join-token manager
 
# 加入集群
docker swarm join 

# 离开swarm
docker swarm leave

# 对swarm集群更新配置
docker swarm update
```

### 管理swarm节点

```shell

# 查看集群中的节点
docker node ls

# 将manager角色降级为worker
docker node demote 节点名称|节点ID

# 将worker角色升级为manager
docker node promote 节点名称|节点ID

# 查看节点的详细信息，默认json格式
docker node inspect 节点名称|节点ID

# 查看节点信息平铺格式
docker node inspect --pretty 节点名称|节点ID

# 查看运行的一个或多个及节点任务数，默认当前节点
docker node ps

# 删除节点（-f强制删除）
docker node rm 节点名称|节点ID

# 更新节点
docker node update 节点名称|节点ID

# 对节点设置状态（"active"正常|"pause"暂停|"drain"排除自身work任务）
docker node update --availability
```

### 服务

```shell
# 创建服务
docker service create
 
# 查看所有服务
docker service ls 	
 
# 查看服务详情
docker service inspect 服务名称|服务ID
 	
# 查看服务日志
docker service logs 服务名称|服务ID
 	
# 删除服务（-f强制删除）
docker service rm 服务名称|服务ID
 	
# 设置服务数量
docker service scale 服务名称|服务ID=n 	
 
# 更新服务
docker service update 服务名称|服务ID 

# 容器加入指令
docker service update --args "指令" 服务名

# 更新服务容器版本
docker service update --image 更新版本 服务名

# 回滚服务容器版本
docker service update --rollback 回滚服务名

# 添加容器网络
docker service update --network-add 网络名 服务名

# 删除容器网络
docker service update --network-rm 网络名 服务名

# 服务添加暴露端口
docker service update --publish-add 暴露端口:容器端口 服务名

# 移除暴露端口
docker service update --publish-rm 暴露端口:容器端口 服务名

# 修改负载均衡模式为dnsrr
docker service update --endpoint-mode dnsrr 服务名

# 添加新的配置文件到容器内
docker service update --config-add 配置文件名称，target=/../容器内配置文件名 服务名
```

### 服务栈

```shell
# 通过.yml文件指令部署
docker stack deploy -c 文件名.yml 编排服务名

# 查看编排服务
docker stack ls

# 查看服务详情
docker stack services 编排服务名

# 删除服务
docker stack rm 编排服务名

# docker stack 不支持使用参数
build
cgroup_parent
container_name
devices
dns
dns_search
tmpfs
external_links
links
network_mode
security_opt
stop_signal
sysctls
userns_mode
```

### 添加网络

```shell
docker service create --network 网络名
```

### 挂载

```shell
# 创建volume类型数据卷
docker service create --mount type=volume,src=volume名称,dst=容器目录
# 创建bind读写目录挂载
docker service create --mount type=bind,src=宿主目录,dst=容器目录
# 创建bind只读目录挂载
docker service create --mount type=bind,src=宿主目录,dst=容器目录,readonly
# 创建dnsrr负载均衡模式
docker service create --endpoint-mode dnsrr 服务名
# 创建docker配置文件到容器本地目录
docker service create --config source=docker配置文件,target=配置文件路径
```

### 暴露端口

```shell
# 创建添加端口
docker service create --publish 暴露端口:容器端口 服务名
```

> [docker swarm 命令](https://blog.51cto.com/yangxingzhen/5980323)


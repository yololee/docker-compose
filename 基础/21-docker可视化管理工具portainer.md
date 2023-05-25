### docker-compose-portainer.yml

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


### 一、前言

Harbor是一个用于存储和分发Docker镜像的企业级Registry服务器。

本文将基于docker和docker-compose环境简单部署Harbor，并通过docker推送/拉取镜像操作

1. Docker version 20.10.22, build 3a2c30b
2. Docker Compose version v2.15.1

### 二、部署Harbor

```shell
# 进入自己的安装目录
cd /opt/soft-dev/docker/harbor

# 下载： https://github.com/goharbor/harbor/releases/
wget https://github.com/goharbor/harbor/releases/download/v2.3.2/harbor-offline-installer-v2.3.2.tgz

# 解压
tar xvf harbor-offline-installer-v2.3.2.tgz

# 进入harbor
cd harbor

# 拷贝配置文件`harbor.yml`
cp harbor.yml.tmpl harbor.yml
# 修改配置(可参考后面给出的demo)
vim harbor.yml

# 安装
./install.sh
```

修改harbor.yml

```yaml
# 填写并且ip或者自己的主机名
hostname: harbor.yolo.com

# 我们这里不采用https，全部注释掉
#https:
  # https port for harbor, default is 443
#  port: 443
  # The path of cert and key files for nginx
#  certificate: /your/certificate/path
#  private_key: /your/private/key/path

#然后就是修改数据和日志存放位置

log:
  local:
    location: /opt/soft-dev/docker/harbor/logs
    
data_volume: /opt/soft-dev/docker/harbor/data
```

安装成功如下：

![image-20230522155623976](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230522155623976.png)

访问 [http://127.0.0.1:11000](http://127.0.0.1:11000)，登录`admin/Harbor12345`
这里新建项目`test`，后面docker测试推送镜像需要，否则后面会报错: `unauthorized: project test not found: project test not found`


其它：

```shell
# 温馨小提示：在harbor安装目录下执行如下命令

# 运行harbor
docker-compose start

# 停止harbor
docker-compose stop
# 删除harbor
docker-compose rm
```


### 三、docker推送/拉取镜像

#### 1、docker登录harbor

```shell
docker login -u admin -p Harbor12345 127.0.0.1:11000
# 通过ip认证需要配置`/etc/docker/daemon.json`
# docker login -u admin -p Harbor12345 172.100.40.215:11000
```

![image-20230522155830525](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230522155830525.png)
解决

> 可参考： [https://goharbor.io/docs/2.3.0/install-config/run-installer-script/#connect-http](https://goharbor.io/docs/2.3.0/install-config/run-installer-script/#connect-http)

###### 法一：{ "insecure-registries":["harbor的ip:port"] }

```shell
sudo vim /etc/docker/daemon.json
# 新增配置 { "insecure-registries":["harbor的ip:port"] }
{
   "insecure-registries": [ "172.100.40.215:11000" ] 
}

# 加载配置文件
systemctl daemon-reload
# 重启docker
systemctl restart docker
```

![image-20230522155850097](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230522155850097.png)

![image-20230522155902081](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230522155902081.png)

#### 2、docker推送镜像

```shell
//登录 Harbor
docker login -u admin -p Harbor12345 127.0.0.1:11000
 
//下载镜像进行测试
docker pull nginx
 
//将镜像打标签
格式：docker tag 镜像:标签  仓库IP/项目名称/镜像名:标签
docker tag nginx:latest 127.0.0.1:11000/test/nginx:v1
 
//上传镜像到 Harbor
docker push 127.0.0.1:11000/test/nginx:v1
```

推送成功如下，harbor中查看

![image-20230330111008194](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230330111008194.png)

![image-20230330111030454](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230330111030454.png)


#### 3、docker拉取镜像

```shell
docker pull 127.0.0.1:11000/test/nginx:v1
```


---


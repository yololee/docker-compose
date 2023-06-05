# docker开启TLS和CA认证

由于开启2375远程访问不安全，所以需要配置docker的ca证书

## 一、生成证书

```shell 
mkdir -p /root/docker/ca
cd /root/docker/ca
touch auto-generate-docker-tls-ca.sh
```

```shell
#!/bin/sh

ip=116.211.105.117
password=eY9bU0cgsgfss8oX3r
dir=/root/docker/cert # 证书生成位置
validity_period=10    # 证书有效期10年

# 将此shell脚本在安装docker的机器上执行，作用是生成docker远程连接加密证书

if [ ! -d "$dir" ]; then
  echo ""
  echo "$dir , not dir , will create"
  echo ""
  mkdir -p $dir
else
  echo ""
  echo "$dir , dir exist , will delete and create"
  echo ""
  rm -rf $dir
  mkdir -p $dir
fi

cd $dir || exit
# 创建根证书RSA私钥
openssl genrsa -aes256 -passout pass:"$password" -out ca-key.pem 4096
# 创建CA证书
openssl req -new -x509 -days $validity_period -key ca-key.pem -passin pass:"$password" -sha256 -out ca.pem -subj "/C=NL/ST=./L=./O=./CN=$ip"
# 创建服务端私钥
openssl genrsa -out server-key.pem 4096
# 创建服务端签名请求证书文件
openssl req -subj "/CN=$ip" -sha256 -new -key server-key.pem -out server.csr

echo subjectAltName = IP:$ip,IP:0.0.0.0 >>extfile.cnf

echo extendedKeyUsage = serverAuth >>extfile.cnf
# 创建签名生效的服务端证书文件
openssl x509 -req -days $validity_period -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -passin "pass:$password" -CAcreateserial -out server-cert.pem -extfile extfile.cnf
# 创建客户端私钥
openssl genrsa -out key.pem 4096
# 创建客户端签名请求证书文件
openssl req -subj '/CN=client' -new -key key.pem -out client.csr

echo extendedKeyUsage = clientAuth >>extfile.cnf

echo extendedKeyUsage = clientAuth >extfile-client.cnf
# 创建签名生效的客户端证书文件
openssl x509 -req -days $validity_period -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -passin "pass:$password" -CAcreateserial -out cert.pem -extfile extfile-client.cnf
# 删除多余文件
rm -f -v client.csr server.csr extfile.cnf extfile-client.cnf

chmod -v 0400 ca-key.pem key.pem server-key.pem

chmod -v 0444 ca.pem server-cert.pem cert.pem
```

生成证书

```shell
sh auto-generate-docker-tls-ca.sh
```

##  二、开启远程访问与配置认证

```shell
vim /lib/systemd/system/docker.service

ExecStart=/usr/bin/dockerd --tlsverify --tlscacert=/root/docker/cert/ca.pem --tlscert=/root/docker/cert/server-cert.pem --tlskey=/root/docker/cert/server-key.pem  -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock

# 如果防火墙开启的话，需要开放2375端口

# 重载服务并重启docker
systemctl daemon-reload && systemctl restart docker
```

![image-20230601233806954](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230601233806954.png)

在服务器测试

```shell
docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem -H tcp://116.211.105.117:2375 version
```

注：请切换至证书文件存放目录，执行上述命令，tcp://docke服务器宿主机地址:端口 进行链接测试，如过输出正确的docker版本信息，则表明配置成功。

![image-20230601214445114](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230601214445114.png)

### 三、本地测试

需要把证书放到本地的一个位置，进入到证书目录执行

```shell
curl https://116.211.105.117:2375/info --cert cert.pem --key key.pem --cacert ca.pem
```

![image-20230601214423993](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230601214423993.png)

### 四、Idea配置

![image-20230601214257275](https://gitee.com/huanglei1111/phone-md/raw/master/images/image-20230601214257275.png)


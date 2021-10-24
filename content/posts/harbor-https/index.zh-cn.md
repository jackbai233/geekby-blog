---
weight: 7
title: "Harbor使用自签名证书配置https认证"
date: 2020-03-27T15:45:54+08:00
lastmod: 2020-03-27T15:45:54+08:00
draft: false
author: "JackBai"
authorLink: "https://www.geekby.cn"
description: "本篇文章介绍了怎样使用自签名的证书配置Harbor的https."
featuredImage: "https://cdn.jsdelivr.net/gh/jackbai233/image-hosting@master/20211024/harbor-https.5rj63h0kqg40.png"

tags: ["Harbor", "https"]
categories: ["云原生"]

lightgallery: true
---

我们知道Harbor是用来存储docker镜像的仓库系统。目前docker从镜像仓库pull或push镜像都是采用https形式的（例如官方的Docker hub），故有必要将Harbor配置成https访问，并使其他Docker机器能成功推送、拉取镜像。

<!--more-->

## 步骤
关于Harbor配置https的文档[官方](https://goharbor.io/docs/1.10/install-config/configure-https/)有非常详尽的说明。这里主要对一些我遇到的问题做一个补充。
因我这里配置Harbor的机器没有申请到域名，故只能采用Harbor主机IP代替，假设IP为：10.34.56.78。

### 1. 生成证书
在10.34.56.78的机器目录下新建文件夹例如叫ca_files(绝对路径假设为/home/usr/ca_files),然后在该目录下依照官方文档步骤生成证书（下面每个命令以###隔开）：
```shell
mkdir ca_files
###
cd ca_files
###
openssl genrsa -out ca.key 4096
###
openssl req -x509 -new -nodes -sha512 -days 3650 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=10.34.56.78" -key ca.key -out ca.crt

```
上面的命令运行完后，可以看到在ca_files下生成了ca.key、ca.crt两个文件，继续运行如下命令来生成私钥：
```shell
openssl genrsa -out 10.34.56.78.key 4096
###
openssl req -sha512 -new -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=10.34.56.78" -key 10.34.56.78.key -out 10.34.56.78.csr
```
上面命令运行完后可以看到在ca_files下继续生成了10.34.56.78.key、10.34.56.78.csr两个文件。
接着生成一个v3.ext文件并使用这个文件来获取认证，命令如下：
```shell
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = IP:10.34.56.78
EOF
### 
openssl x509 -req -sha512 -days 3650 -extfile v3.ext -CA ca.crt -CAkey ca.key -CAcreateserial -in 10.34.56.78.csr -out 10.34.56.78.crt
```
不出意外又会在ca_files文件夹下生成一些文件.这里要注意，上面第一个命令中的：**subjectAltName = IP:10.34.56.78** 这一行与官方文档中有些不同，因为我们这里没有申请域名，所以直接以IP指定。
### 2. 配置Harbor
假设我这里在10.34.56.78机器上安装Harbor的绝对路径为/home/usr/harbor，则在harbor目录下找到 **harbor.yml** 文件（我这里Harbor安装的版本为v1.10.1）进行编辑：
```yaml
# https related config
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  # 注意这里配置证书路径
  certificate: /home/usr/ca_files/10.34.56.78.crt
  private_key: /home/usr/ca_files/10.34.56.78.key
```
配置完成后，若Harbor还没有安装，则直接在harbor目录下运行：
```shell
sudo ./install.sh
```
若Harbor已安装过，则在harbor目录下运行如下命令：
```shell
sudo ./prepare
###
docker-compose down -v
###
docker-compose up -d
```
此时，若没有报错，则Harbor已经变成https访问了。
### 3. 在其他机器上docker login Harbor
假设我有另一台机器（IP为：12.34.56.78），这台机器已配置完docker环境，并且我已经做好了一个docker的image,假设image名为：my_nginx:v1.0。
这时，为了使该机器能“docker login 10.34.56.78”成功，我们还需要做一些配置。首先在12.34.56.78机器上创建一个文件夹（例如名为：my_ca,绝对路径是：/home/me/my_ca），然后将10.34.56.78机器（即Harbor部署的主机）的ca_files文件夹下的：**10.34.56.78.crt**、**10.34.56.78.key**、**ca.crt**这三个文件拷贝到my_ca目录下。
拷贝完成后，进入到my_ca目录下，运行如下命令：

```shell
openssl x509 -inform PEM -in 10.34.56.78.crt -out 10.34.56.78.cert
```
可以看到my_ca目录下新生成了10.34.56.78.cert这个文件。
下面我们需要在12.34.56.78这台机器的/etc/docker/目录下创建一对父子文件夹（certs.d/10.34.56.78）,命令如下：

```shell
mkdir –p /etc/docker/certs.d/10.34.56.78
```
创建完后成，我们将my_ca目录下的一些文件拷贝到certs.d/10.34.56.78目录下，即：
```shell
cd /home/me/my_ca/
###
cp 10.34.56.78.cert /etc/docker/certs.d/10.34.56.78/
###
cp 10.34.56.78.key /etc/docker/certs.d/10.34.56.78/
###
cp ca.crt /etc/docker/certs.d/10.34.56.78/
```
接着重启docker服务：
```shell
systemctl restart docker
```
上面的操作进行完后，我们就能在12.34.56.78这台机器上 **docker login 10.34.56.78** 并推送“my_nginx:v1.0”这个image了。
例如在Harbor系统网站内建立一个名为test的项目，则推送image到该项目下的命令为：

```shell
docker tag my_nginx:v1.0 10.34.56.78/test/my_nginx:v1.0
docker push 10.34.56.78/test/my_nginx:v1.0
```
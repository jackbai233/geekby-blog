# Docker下搭建mongodb副本集


有需求需要对mongodb做一个容灾备份。根据官网，发现mongodb最新版本(4.0)已经抛弃了**主从模式**而采用**副本集**进行容灾。副本集的优势在于：”有自动故障转移和恢复特性，其任意节点都可以是主节点，并能实现读写分离，提供高负载“。官方建议副本集最低配置三个节点。关于副本集的原理更多请参考[这位小姐姐的博客](https://www.cnblogs.com/Joans/p/7680846.html#4330762)

<!--more-->

## 搭建步骤
#### 制作mongodb镜像
首先需要做一个mongodb的docker镜像，这里我采用dockerfile进行制作，dockerfile内容如下：
```dockerfile
# 指定镜像源
FROM  ubuntu:16.04

RUN apt-get update
RUN apt-get install gcc -y
RUN apt-get install g++ -y
RUN apt-get install make -y
RUN apt-get install wget -y

# Install MongoDB.
RUN \
  apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10 && \
  echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' > /etc/apt/sources.list.d/mongodb.list && \
  apt-get update && \
  apt-get install -y mongodb-org && \
  rm -rf /var/lib/apt/lists/*

# Define mountable directories.
VOLUME ["/data/db"]

# Define working directory.
WORKDIR /data
RUN mkdir bin && mkdir log
COPY mongodb.conf ./bin/

# Expose ports.
#   - 27017: process
#   - 28017: http
EXPOSE 27017
EXPOSE 28017

# Define default command.
# CMD ["mongod", "-f", "./bin/mongodb.conf"]
```
这里指定了以ubuntu为源镜像制作mongodb镜像。并将mongodb配置文件mongodb.conf拷贝到镜像中，最后运行mongodb的启动服务。(关于mongodb的dockerfile请参考：[这里](http://dockerfile.github.io/#/mongodb))
mongodb.conf的配置如下：

```log
# 指定数据库保存位置
dbpath = /data/db/
# 指定mongodb 运行中log的保存位置
logpath = /data/log/logs.log
# mongodb的暴露端口
port = 27017
# 是否在后台运行
# fork = true
# 是否进行用户认证
# auth = true
```
可以看到配置文件中只指定了前三项。其中dockerfile文件与mongodb.conf必须在同一目录下。然后在该目录下使用docker命令制作mongodb镜像：
```
docker build -t mongodb:v1.0 .
```
上面命令的最后一个点不要忘记了。
没有问题的话，mongodb的基本镜像算是制作成功了。下面就是启动一个该镜像的容器来进行验证：
```
docker run -it --name mongodb_test -p 27017:27017 -p 27018:27018 -v /home/root/user/mongodb_dir/data/db:/data/db -v /home/root/user/mongodb_dir/data/log:/data/log mongodb:v1.0
```
运行命令中将27017与27018两个端口进行了映射。并在宿主机中建立mongodb_dir/data文件夹,该文件夹下有db与log子文件夹，这两个子文件夹与mongodb_test的容器中文件夹进行了映射。进入到容器中后运行”**mongod -f ./bin/mongodb.conf**“即可打开mongodb服务。
#### 制作mongodb副本集
上一步仅仅只是完成了mongodb服务，下面便是制作mongodb副本集了。因为只有一台机器，故最低需要三个节点制作副本集，这个要求我们可以用三个mongodb容器来代替。
首先在宿主机上建立一个文件夹db_test（名字随意取），然后在db_test文件夹下建立data1、data2、data3三个文件夹下，这三个文件夹下再分别建立db、log文件夹。其目录大致如下：
```
--db_test
 --data1
   --db
   --log
 --data2
   --db
   --log
 --data3
   --db
   --log
```
这三个data文件夹分别用来存放三个节点(即三个mongodb容器)的数据与log。为了满足副本集的需要，我们需要重新修改mongodb.conf配置文件
mongodb.conf文件如下：
```
dbpath = /data/db/
logpath = /data/log/logs.log
port = 27017
# 副本集的名称
replSet = testrepl
fork = true
```
因mongodb.conf配置文件变化了。所以mongodb的镜像我们需要重新编一次，为了好区分这里将新的镜像版本变为2.0，运行命令：
```Docker
docker build -t mongodb:v2.0 .
```
然后运行以下命令，创建三个mongodb容器：
```dockerfile
docker run -it --name mongodb_test1 -p 27017:27017 -p 27018:27018 -v /home/root/user/db_test/data1/db/:/data/db -v /home/root/user/db_test/data1/log/:/data/log mongodb_vs:v2.0
docker run -it --name mongodb_test2 -p 27019:27017 -p 27020:27018 -v /home/root/user/db_test/data2/db/:/data/db -v /home/root/user/db_test/data2/log/:/data/log mongodb_vs:v2.0
docker run -it --name mongodb_test3 -p 27021:27017 -p 27022:27018 -v /home/root/user/db_test/data3/db/:/data/db -v /home/root/user/db_test/data3/log/:/data/log mongodb_vs:v2.0
```
如此，我们便打开了三个mongodb容器：mongodb_test1、mongodb_test2、mongodb_test3。然后在三个容器下运行”**mongod -f ./bin/mongodb.conf**“命令，打开三个mongodb服务。此时相当于有三个mongodb节点，但这三个节点还没有在一个副本集中。我们在最先打开的那个mongodb服务节点中(例如最先打开了mongodb_test1容器的服务），运行命令：**mongo**
然后输入如下命令，将三个节点加入到一个副本集中：

```
config={  
     _id:"testrepl",
    members:[
        {_id:0,host:"10.22.33.44:27017"},        
        {_id:1,host:"10.22.33.44:27019"},
        {_id:2,host:"10.22.33.44:27021"}]
    } 
```
上面的ip是宿主机的ip地址，三个端口分别对应于三个mongodb节点， _id对应的是副本集的名称
上述命令运行完成后
再继续输入命令：
```
rs.initiate(config)
```
若初始化成功，则返回以下log：
```
{
    "ok" : 1
}
```
我们也可以输入**rs.status()** 命令查看副本集的状态及各个节点的主从状态。
以上步骤若是没有报错的话，则一个mongodb副本集便搭建成功了。我们可以在主节点上插入数据，然后看其他从节点是否同步了该数据。并可以尝试把主节点断开，然后输入 **rs.status()** 命令查看主从关系。
## 总结
以上，我们就搭建了一个mongodb的副本集。若是副本集要进行用户认证管理，可以参考[这篇文章](https://www.jianshu.com/p/f021f1f3c60b)。
## 参考文章
[https://www.cnblogs.com/Joans/p/7680846.html#4330762](https://www.cnblogs.com/Joans/p/7680846.html#4330762)

[https://www.cnblogs.com/Joans/p/7723554.html#4330857](https://www.cnblogs.com/Joans/p/7723554.html#4330857)

[https://www.jianshu.com/p/f021f1f3c60b](https://www.jianshu.com/p/f021f1f3c60b)

[https://blog.csdn.net/duzm200542901104/article/details/81781291](https://blog.csdn.net/duzm200542901104/article/details/81781291)

[https://www.jianshu.com/p/ba63f6c5ad04](https://www.jianshu.com/p/ba63f6c5ad04)


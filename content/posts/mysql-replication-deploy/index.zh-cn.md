---
weight: 1
title: "MySQL 搭建主从复制环境"
date: 2022-05-08T09:59:05+08:00
lastmod: 2022-05-08T09:59:05+08:00
draft: false
author: "geekby"
authorLink: "https://www.geekby.cn"
description: "这篇文章介绍了在docker环境下搭建MySQL主从复制环境的过程."
featuredImage: "https://cdn.staticaly.com/gh/jackbai233/image-hosting@master/20211024/mysql-replication-deploy.5y7m9pwx1iw0.webp"

tags: ["MySQL"]
categories: ["后端"]

lightgallery: true
---

本文主要介绍了怎样在 docker 环境下搭建 MySQL 主从复制环境.

<!--more-->

为了数据库数据能够容灾备份，提高读写效率，其一般都会搭建备份环境来作为解决方案，而主从复制环境就是一种主流的解决方案。本文将介绍在docker环境下搭建MySQL主从复制环境的具体流程。
## 前期准备
1. 准备三台 linux 机器(假设我这里为ubuntu。一般主从复制环境所需机器至少为三台)，其中一台为主节点，剩下两台为从节点。如下：
```shell
    # 主节点
    10.23.45.67

    # 从节点
    10.23.45.89
    10.23.45.99
```
2. 为准备的三台机器安装docker环境(docker版本取最新即可)。因本文主要涉及MySQL相关内容，关于docker环境的安装，可参考[官方安装指南](https://docs.docker.com/engine/install/ubuntu/)。
3. 三台机器可互相通信。即三台机器可以互相ping通，且能互相访问其`3306`端口。

## 环境搭建
首先需要三台机器有 MySQL运行环境，故在三台机器上分别拉取 MySQL 指定版本的镜像。我这里取MySQL 5.7.32 版本：
```shell
    # 主节点 10.23.45.67
    docker pull mysql:5.7.32
    
    # 从节点 10.23.45.89
    docker pull mysql:5.7.32
    
    # 从节点 10.23.45.99
    docker pull mysql:5.7.32
```
镜像拉取完之后，就是创建并运行 MySQL 容器了。为了使 MySQL数据不随容器死亡而消失，需要在宿主机下创建相关目录与容器内部作映射，假如三台机器的当前用户目录均为 `/home/user/`，操作如下：
```shell
    # 先创建 顶层目录如 mysql, 用于放置 mysql 的数据或配置文件等
    sudo mkdir mysql && sudo chmod 777 mysql
    # 进入 mysql 目录下
    cd mysql
    # 创建数据目录 data、配置目录 conf 和运行后产生的 log 目录
    sudo mkdir data conf log
    # 先赋予最大权限
    sudo chmod 777 data conf log
```
{{< admonition type=note title="注意" open=true >}}
上述创建目录操作在三台机器上都需要执行
{{< /admonition >}}

存放数据的目录创建完后，就是运行 MySQL 容器了。因搭建主从复制环境需要对 MySQL 配置做一些更改，故这里分别对主从节点进行配置，并在容器启动时加载对应配置文件。
### 1. 主节点操作
首先在主节点(10.23.45.67)下，创建配置文件：
```shell
    # 进入conf目录下：
    cd /home/user/mysql/conf/
    # 创建 配置文件 my.cnf
    sudo touch my.cnf
    sudo chmod 777 my.cnf
```
然后使用编辑器打开 `my.cnf` 文件，放入以下内容进行保存：
```shell
    [mysqld]
    ## 设置server_id , 同一局域网中需要唯一, 该id值自己可以随意取，但需要与从节点id值区分
    server_id=100
    ## 指定不需要同步的数据库名称
    binlog-ignore-db=mysql
    ## 开启二进制日志功能，这一步是必须配置的
    log-bin=mall-mysql-bin
    ## 设置二进制日志使用内存大小
    binlog_cache_size=1M
    ## 设置使用的二进制日志格式（mixed,statement,row）
    binlog_format=mixed
    ## 二进制日志过期清理时间。 默认值为0，表示不自动清理。
    expire_logs_days=7
    ## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
    ## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
    slave_skip_errors=1062
```
配置完成之后，就是运行容器了。这里需要说明下， MySQL 启动时会寻找系统下不同位置的配置文件，并对配置文件中的配置项进行读取，具体寻找顺序如下表格：

| 路径名 | 备注 |
|---|---|
| /etc/my.cnf | |
| /etc/mysql/my.cnf | |
| SYSCONFDIR/my.cnf | |
| $MYSQL_HOME/my.cnf | |
| ~/.my.cnf | 特定于用户的配置文件 |

也就是说，如果有多个配置文件，且配置文件中设置了相同的配置项，则以最后一个配置文件中的为准。例如`/etc/mysql/my.cnf` 与 `~/.my.cnf` 中有相同的配置项，则以 `~/.my.cnf` 的配置为准。
不过，在 默认启动容器(mysql:5.7.32)时，其只会有一个已存在的配置文件 `/etc/mysql/my.cnf`。因此，我们只需要将自己创建的配置文件映射掉容器中对应的配置文件即可。

{{< admonition type=note title="注意" open=true >}}
在启动容器前，还需要将my.cnf 文件的权限进行更改，不然容器启动后会出现配置项不生效问题：
```shell
    # 更改 /home/user/mysql/conf/my.cnf 的权限为644
    sudo chmod 644 /home/user/mysql/conf/my.cnf
```
{{< /admonition >}}

接下来运行 MySQL 容器：
```shell
    docker run --restart=always -p 3306:3306 --name mysql-master \
    -v /home/user/mysql/log:/var/log/mysql \
    -v /home/user/mysql/data:/var/lib/mysql \
    -v /home/user/mysql/conf/my.cnf:/etc/mysql/my.cnf \
    -e MYSQL_ROOT_PASSWORD=mima \
    -d mysql:5.7.32
```
容器启动后，使用`docker ps` 命令查看  mysql-master 的容器的 STATUS 状态 是否为 Up，若为 Up 则容器运行成功。
接下来需要连接启动后的主节点 MySQL 服务，对配置项生效与否进行检查，并进行一些数据库更改操作。
首先以客户端方式连接 MySQL 服务：
```shell
    # mysql -uroot -pmima 为以用户：root(mysql默认用户) 密码：mima(启动容器时MYSQL_ROOT_PASSWORD变量的值)
    # 来连接 MySQL服务
    docker exec -it  mysql-master mysql -uroot -pmima
```
若正常，则终端下会出现 mysql 服务交互的命令提示符 `mysql>`。首先，检查 my.cnf 配置文件的内容是否生效(这里可检查 server_id是否为100)：
```shell
    # 注意 mysql> 为提示符，并不是输入的命令
    mysql> SHOW GLOBAL VARIABLES like 'server\_id';
```
若无报错，则输出如下：
```shell
    mysql> SHOW GLOBAL VARIABLES like 'server\_id';
    +---------------+-------+
    | Variable_name | Value |
    +---------------+-------+
    | server_id     |100    |
    +---------------+-------+
    1 row in set (0.00 sec)
```
可见配置文件生效了。接下来需创建数据同步的用户：
```shell
    # 创建用户：slave 及对应密码。账户密码可根据自己喜好创建
    mysql> CREATE USER 'slave'@'%' IDENTIFIED BY '123456';
```
授予权限：
```shell
    mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';
```
刷新权限：
```shell
    mysql> FLUSH PRIVILEGES;
```
最后，需要查看 主节点 的状态，并将其记录值下来，以供后续从节点使用：
```shell
    mysql> SHOW master STAUTS;
```
输出内容如下：
```shell
    +-------------------+----------+--------------+-------------------------------+ 
    | File              | Position | Binlog_Do_DB | Binlog_Ignore_DB              | 
    +-------------------+----------+--------------+-------------------------------+ 
    | mysqld-bin.000001 | 3267     |              | mysql,test,information_schema | 
    +-------------------+----------+--------------+-------------------------------+ 
```
{{< admonition type=note title="注意" open=true >}}
File参数的值 `mysqld-bin.000001`和 Position 参数的值 `3260`需要记录保存。
{{< /admonition >}}

断开 MySQL 服务连接：
```shell
    mysql> exit;
```
至此，主节点操作完成。
### 从节点操作
从节点(10.23.45.89、10.23.45.99)的操作步骤相同。首先，也需要创建名为 `my.cnf` 的配置文件来启动从节点容器：
```shell
    # 创建配置文件
    sudo touch /home/user/mysql/conf/my.cnf
    sudo chmod 777 /home/user/mysql/conf/my.cnf
```
使用编辑器打开 my.cnf 文件，放入以下内容保存：
```shell
    [mysqld]
    ## 设置从节点server_id , 需跟主节点及任意从节点id值不同
    server_id=101
    ## 指定不需要同步的数据库名称
    binlog-ignore-db=mysql
    ## 开启二进制日志功能
    log-bin=mall-mysql-slave1-bin
    ## 设置二进制日志使用内存大小
    binlog_cache_size=1M
    ## 设置使用的二进制日志格式（mixed,statement,row）
    binlog_format=mixed
    ## 二进制日志过期清理时间。 默认值为0，表示不自动清理。
    expire_logs_days=7
    ## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
    ## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
    slave_skip_errors=1062
    ## relay_log配置中继日志
    relay_log=mall-mysql-relay-bin
    ## log_slave_updates表示slave将复制事件写进自己的二进制日志
    log_slave_updates=1
    ## slave设置为只读（具有super权限的用户除外）
    read_only=1
```
不要忘记将 my.cnf 的权限改回为644：
```shell
    sudo chmod 644 /home/user/mysql/conf/my.cnf
```
运行 MySQL 容器：
```shell
    docker run --restart=always -p 3306:3306 --name mysql-slave1 \
    -v /home/user/mysql/log:/var/log/mysql \
    -v /home/user/mysql/data:/var/lib/mysql \
    -v /home/user/mysql/conf/my.cnf:/etc/mysql/my.cnf \
    -e MYSQL_ROOT_PASSWORD=mima \
    -d mysql:5.7.32
```
检查容器运行状态成功后，连接从节点 MySQL 服务：
```shell
    docker exec -it mysql-slave1 mysql -uroot -pmima
```
输入以下命令，告诉从节点其需要同步的主节点ip、数据库同步账户等内容：
```shell
    mysql> CHANGE master TO master_host='10.23.45.67',
        master_user='slave',
        master_password='123456',
        master_port=3306,
        master_log_file='mysqld-bin.000001',
        master_log_pos=3267,
        master_connect_retry=30;
```
- master_host: 主数据库的IP地址；
- master_port: 主数据库的运行端口；
- master_user： 在主数据库创建的用于同步数据的用户账号；
- master_password: 在主数据创建的用于同步数据的用户密码；
- master_log_file: 指定从数据库要复制数据的日志文件，通过查看主节点的状态，获取File参数；
- master_log_file: 指定从数据库从哪个位置开始复制数据，通过查看主节点的状态，获取Poition 参数；
- master_connect_retry: 连接失败重试的时间间隔，单位为秒。

接下来，开启主从同步服务：
```shell
    mysql> start slave;
```
最后，查看从节点slave的状态：
```shell
    mysql> show slave status \G;
```
其输出日志中`Slave_IO_Running`、`Slave_SQL_Running` 值为 YES 说明主从复制配置成功。
断开从节点 MySQL 服务连接：
```shell
    mysql> exit;
```
至此，MySQL 主从复制环境搭建完成。

## 附录
1. MySQL主从复制搭建：[https://www.jianshu.com/p/214893f2bdda](https://www.jianshu.com/p/214893f2bdda)
2. MySQL主从复制搭建：[https://www.1024sou.com/article/645416.html](https://www.1024sou.com/article/645416.html)
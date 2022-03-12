---
weight: 1
title: "MySQL 高可用方案研究"
date: 2022-03-12T17:11:21+08:00
lastmod: 2022-03-12T17:11:21+08:00
draft: false
author: "JackBai"
authorLink: "https://www.geekby.cn"
description: "这篇文章讲述了 MySQL 高可用的几种方案"
featuredImage: "https://cdn.jsdelivr.net/gh/jackbai233/image-hosting@master/20211024/Image-(6).2g16fjf5jls0.webp"

tags: ["MySQL"]
categories: ["后端"]

lightgallery: true
---

本文主要介绍了什么是高可用、为什么要进行高可用以及 MySQL 中关于高可用方案的集中实现.

<!--more-->
注：文中图片部分来自于[https://severalnines.com/database-blog/overview-mysql-database-high-availability](https://severalnines.com/database-blog/overview-mysql-database-high-availability)
{{< admonition type=note title="声明" open=true >}}
本文假设你使用过数据库，并熟悉 MySQL 数据库软件的基本使用。
{{< /admonition >}}

## 高可用是什么
可用的意思不言自明，如果数据库中的数据可以被你的应用查到，那么该数据库就是可用的。而高可用则是另外一层意思，对于一些企业来说，它可能指一年内最多有几分钟的停机时间，对另一些企业可能意味着每月几个小时的停机时间，换句话说，它取决于具体的业务需求。
因此，必须明白你所在组织的具体需求，即能忍受多长的停机时间。这也将会影响你的高可用(HA)部署方案。不过高可用性方案一般建立在处理主机故障并在必要时恢复的能力上，以使数据库服务可提供 99.999% 的正常运行时间。
  
## 为什么要高可用？
数据作为当前web端、移动端、社交、商业以及云服务必须物，对于任何组织，确保数据及服务一直可用是最高优先级，而数分钟内的停机就可能造成它们名誉与收入的巨大损失 。
数据库一直是存储数据的结构化容器，而高可用环境为数据库能尽可能长久地运行提供了实际好处，一种高可用的数据库环境可以跨多台机器提供数据库功能，换句话说，这种环境的数据库不会有“单点故障”。

## 高可用方案实现
{{< admonition type=tip title="注意" open=true >}}
在阅读下文之前，假设你已经熟悉 MySQL 中 `存储引擎`、`binlog`、`事务`的概念，如果没有了解过，下方给出了简单的概念：
- **binlog**：binlog 是一组日志文件，其中包含有关对 MySQL 服务器实例进行的数据修改的信息。通过使用 `--log-bin` 选项启动服务器可以启用 binlog。更多关于 binlog 的描述可见[官网](https://dev.mysql.com/doc/internals/en/binary-log-overview.html)。

- **存储引擎**：存储引擎是数据库管理系统用来在数据库中进行创建、读取、更新和删除（CRUD）数据的底层软件组件。大多数数据库管理系统都提供应用程序接口 (API)，允许程序员通过 API 与其底层的存储引擎进行交互 。许多数据库管理系统支持多个存储引擎，例如 MySQL 支持 InnoDB 以及 MyISAM等其他存储引擎 。MySQL的默认存储引擎为 InnoDB。关于存储引擎的更多介绍见[官网](https://dev.mysql.com/doc/refman/5.6/en/storage-engines.html)。

- **事务**： 事务允许您执行一组 MySQL 操作语句，以确保数据库不包含部分操作的结果。在一组操作中，如果其中一个操作失败，则会执行回滚将数据库还原到其原始状态。事务的使用需要存储引擎支持。
{{< /admonition >}}

下面主要介绍了一些实施高可用主流方案。
#### 缓存层--数据库读写
在谈及高可用方案时，有时候我们会考虑对于用户侧的影响，有时候用户并不需要频繁地修改数据，而只是查看数据，这时我们可以在应用与数据库之间添加一层缓存：
{{< figure src="https://cdn.jsdelivr.net/gh/jackbai233/image-hosting@master/20211024/dha_cache.24f9vfyggjx.webp" title="dha_cache" >}}

对于读的缓存，我们有许多方案： memcached、Redis、couchbase等。缓存的刷新可以在需要时由后台线程从数据库中读取数据并写入到缓存里进行刷新。当然当数据库服务掉线或后台线程不能刷新时，缓存的数据会过期。不过当数据库掉线后，应用后台可以从缓存里获取数据提供给用户，短期来看，对用户来说并不会有糟糕的体验。

#### 块级别复制(DRBD)
DRBD(Distributed Replicated Block Device, 分布式复制块设备)是一个软件实现的、无共享的、服务器之间镜像块设备内容的存储复制解决方案。简单说，DRBD是实现活动节点存储数据变动后自动复制到备用节点相应存储位置的软件。
{{< figure src="https://cdn.jsdelivr.net/gh/jackbai233/image-hosting@master/20211024/image.6ghy5rmz9p00.webp" title="drbd" >}}

上图是DRBD的工作栈模型，可以看到DRBD需要运行在各个节点上，且运行在节点主机的内核中(Linx2.6.33版本起DRBD已整合进内核模块)。

上图中假设左节点为活动节点，右节点为备用节点。左节点接收到数据发往内核的数据通路，DRBD 在数据通路中注册钩子检查数据（类似 ipvs），当发现接收到的数据是发往到自己管理的存储位置，一份存储到本机的 DRBD 存储设备，然后复制另一份发给 TCP/IP 协议栈，通过网卡网络传输到另一节点主机的网上 TCP/IP 协议栈；而另一节点运行的 DRBD 模块同样在数据通路上检查数据，当发现传输过来的数据时，就存储到 DRBD 存储设备对应的位置。

如果左节点宕机，右节点可以在高可用集群中成为活动节点，当接收到数据后先存储到本地，在左节点恢复上线时，再把宕机后右节点变动的数据镜像到左节点。

DRBD有几个缺点，首先，该方案中只能使用一个节点(即活动节点)，其他被动节点无法提供临时的查询等服务，也就无法实现读写分离了。另外，如果系统崩溃，主库进程中断，故障转移后必须得在挂掉的数据库上做数据库崩溃恢复，系统需要的容灾恢复时间较长。

#### MySQL主从复制(MySQL Replication)
      
MySQL主从复制是最经典和也许最流行实现MySQL高可用的方案之一，它的架构一般为[master-slave模式](https://zh.wikipedia.org/wiki/%E4%B8%BB%E4%BB%8E%E6%A8%A1%E5%BC%8F)。如下图：
{{< figure src="https://cdn.jsdelivr.net/gh/jackbai233/image-hosting@master/20211024/image.1q0hnsqgvjq8.webp" title="replication" >}}

它将来自一台 MySQL 数据库服务器（源,即master）的数据复制到一台或多台 MySQL 数据库服务器（副本）。复制默认是异步的，因此副本不需要永久连接来接收来自源的更新。这意味着更新可以通过长距离连接进行，甚至可以通过临时或间歇性连接（如拨号服务）进行。根据配置，您可以复制数据库中的所有数据库、选定的数据库甚至选定的表。

MySQL 主从复制的优点包括：
- 横向扩展解决方案——在多个副本之间分散负载以提高性能。在此环境中，所有写入和更新都必须在源服务器上进行。然而，读取可能发生在一个或多个副本上。该模型可以提高写入性能（因为源专用于更新），同时显著提高副本数量的读取速度，即实现读写分离。
- 数据安全——因为数据被复制到副本，并且副本可以暂停复制过程，所以可以在副本上运行备份服务而不会损坏源上的相应数据。
- 分析——可以在源上创建实时数据，而对信息的分析可以在副本上进行，而不会影响源的性能。
- 远程数据分发——如果分支机构想要使用您的主要数据的副本，您可以使用复制来创建数据的本地副本以供其使用，而无需永久访问源。
- Replication 模式中，在 master 上发生的数据变更都将被立即写入 binlog(通常数据库的更新、删除等操作会被记录进binlog)，此后 slaves 将获取master上binlog的一个副本，并在该副本节点上执行binlog中的变更操作，从而实现 “replication”，因此 Replication 并不是将master的数据传输给slave的(扩展：mongodb 的 replication 也是类似原理)。

replication 支持两种模式：asynchronous（异步）、semi-synchronous（半同步）。     
##### 1. 异步模式：
异步模式是默认模式。master 继续处理其他的 write 请求，而不会等待 slaves 对此 update 信息的复制或者应用；此后的任何时候，slaves 均可以与 master 建立链接并复制那些尚未获取的变更日志，然后在本地应用（apply）。

异步模式，无法保证当 master 失效后所有的 updates 已经复制到了 slaves 上，只有重启 master 才能继续恢复这些数据，如果 master 因为宿主机器物理损坏而无法修复，那些尚未复制到 slaves 上的 updates 将永久性丢失；因此异步方式存在一定的数据丢失的风险，但它的优点就是 master 支持的 write 并发能力较强，因为 master 上的 writes 操作与 slaves 的复制是互为独立的。

不过这种模式，slaves 总有一定的延后，这种延后在事务操作密集的应用中更加明显，不过通常这种延后时间都极其短暂的。从另一个方面来说，异步方式不要求 slaves 必须时刻与 master 建立链接，可能 slaves 离线、中断了 replication 进程或者链接的传输延迟很高，这都不会影响 master 对 writes 请求的处理效率。比如对于 “远距分布” 的 slaves，异步复制是比较好的选择。

此模式下，如果 master 失效，我们通常的做法是重启 master，而不是 failover 到其他的 slave，除非 master 无法恢复；因为 master 上会有些 updates 尚未复制给 slaves，如果此时 failover 则意味着那些 updates 将丢失。
##### 2. 半同步模式：
“半同步”并不是 MySQL 内置的 replication 模式，而且由插件实现，即在使用此特性之前，需要在 master 和 slaves 上安装插件，且通过配置文件开启 “半同步”。当 slave 与 master 建立连接时会表明其是否开启了“半同步” 特性；此模式正常运作，需要 master 和至少一个 slaves 同时开启，否则仍将采用 “异步” 复制。

在 master 上执行事务提交的线程，在事务提交后将会阻塞，直到至少一个 “半同步” 的 slave 返回确认消息（ACK）或者所有的半同步 slave 都等待超时；slave 将接收到事务的信息写入到本地的 relay log 文件且 flush 到磁盘后，才会向 master 返回确认消息，需要注意 slave 并不需要此时就执行事务提交，此过程可以稍后进行。当所有的半同步 slaves 均在指定的时间内没有返回确认消息，即 timeout，那么此后 master 将转换成异步复制模式，直到至少一个半同步 slave 完全跟进才会转换为半同步模式。在 master 阻塞结束后才会返回给客户端执行的状态，此期间不会处理其他的事务提交，当 write 请求返回时即表明此操作在 master 上提交成功，且在至少一个半同步 slaves 也复制成功或者超时，阻塞超时并不会导致事务的 rollback。=（对于事务性的表，比如 innodb，默认是事务自动提交，当然可以关闭 “autocommit” 而手动提交事务，它们在 replication 复制机制中并没有区别）。

半同步模式需要在 master 和 slaves 上同时开启，如果仅在 master 上开启，或者 master 开启而 slaves 关闭，最终仍然不能使用半同步复制，而是采用异步复制。

与异步复制相比，半同步提高了数据一致性，降低了数据丢失的风险。但是它也引入了一个问题，就是 master 阻塞等待 slaves 的确认信息，在一定程度上降低了 master 的 writes 并发能力，特别是当 slaves 与 master 之间网络延迟较大时；因此我们断定，半同步 slaves 应该部署在与 master 临近的网络中，为了提高数据一致性，我们有必要将半同步作为 replication 的首选模式。

在实际的部署环境中，并不要求所有的 slaves 都开启半同步，我们可以将与 master 临近的 slaves 开启半同步，将那些 “远距分布” 的 slaves 使用异步。

复制格式有两种核心类型。基于语句的复制 (SBR)，它复制整个 SQL 语句，以及基于行的复制 (RBR)，它只复制更改的行。您还可以使用第三种类型，基于混合的复制 (MBR)。在 MySQL 5.6 中，基于语句的格式是默认的。有关复制格式的更多信息，可参考[官方文档](https://dev.mysql.com/doc/refman/5.6/en/replication-formats.html)。  

MySQL 5.6.5 及更高版本支持基于 全局事务标识符 (GTID) 的事务复制。使用这种类型的复制时，无需直接处理日志文件或这些文件中的位置，这大大简化了许多常见的复制任务。因为使用 GTID 的复制完全是事务性的，所以只要在源上提交的所有事务也已应用于副本上，就可以保证源和副本之间的一致性。有关 GTID 和基于 GTID 的复制的更多信息，可参考[官方文档](https://dev.mysql.com/doc/refman/5.6/en/replication-gtids.html)。
      
在MySQL Replication方案中，master故障转移包括以下几个阶段：    
- 1. 你需要找到性能最优的slave。
- 2. 如果有很多可选的slave，选择一个作为master，将其余的治理任务交给新的master。
- 3. 如果只有一个“性能最好”的slave，您应该尝试确认丢失的事务并在其余从站上重放它们以使它们同步。
- 4. 如果上一步不可行，您将只能重头创建slave，并从master同步数据。
- 5. 执行切换（更改代理配置、移动虚拟 IP、将流量移动到新主服务器等）。

上述几个阶段是一个繁琐的过程，并且在手动执行过程中可能会出错。业界有比较好的自动化方案，其中一个是 MHA(Master High Availability)，它在MySQL故障切换过程中，MHA能做到在0~30秒之内自动完成数据库的故障切换操作，并且在进行故障切换的过程中，MHA能在最大程度上保证数据的一致性，以达到真正意义上的高可用(更多关于MHA的介绍见[这里](https://blog.51cto.com/u_14154700/2472806))。

总得来说，MySQL Replication 默认在异步模式下，可能由于master数据库无法访问，导致slave无法及时获取到对应的binlog或事务而丢失部分数据，因此在使用中可以将一个slave设置为半同步模式，但这并非没有影响，事务的提交可能会变慢，因为master需要等待半同步的slave来记录事务，具体的影响深度取决于你的工作负荷。
      
#### MySQL 集群(Cluster)  
最后的(也是最终的)一个方案是使用一种同步模式的集群，它就是 MySQL Cluster(也称为MySQL NDB Cluster)。MySQL Cluster 与我们平时使用的MySQL存储引擎 InnoDB 不同，它基于 NDB 存储引擎，该存储引擎专门为 Cluster 而设计，它主要将数据存储在内存中，并且独立于MySQL server 实例。NDB 代表“网络数据库”。如下为官方的 Cluster 组件示例图：
{{< figure src="https://cdn.jsdelivr.net/gh/jackbai233/image-hosting@master/20211024/image.43azpr1mw360.webp" title="cluster" >}}

MySQL NDB Cluster 使用非共享架构，通过多台服务器构建成集群，实现多点读写的关系型数据库。它具有在线维护功能，并且排除单一故障，具有非常高的可用性。此外，它的主要数据保存在内存中，可以高速处理大量的事务，是面向实时性应用程序的一款数据库产品。

MySQL NDB Cluster 不是常规的 MySQL/InnoDB，行为也不同。它存储数据的方式(跨多个数据节点分区)使得一些查询更加昂贵，因为获取数据和准备结果需要相当多的网络活动。
相比较于 MySQL Replication。MySQL NDB Cluster 的架构模式和原理更加复杂，Cluster 是一个全分布式架构，是面向大规模数据存储的解决方案。

#### 高可用架构实施工具 Galera     
[Galera Cluster](http://galeracluster.com) 是 Codership 公司开发的一套免费开源的高可用方案。它是基于InnoDB存储引擎的同步模式的多主复制工具，它与常规的 MySQL Replication 不同，它解决了许多问题，包括在多master节点写入时的写冲突、复制延迟以及slave与master不同步等。用户不必知道他们从哪一个master节点上写数据以及哪一台slave上读数据。

应用可以在 Galera Cluster 的任何一个节点上写入数据。其基于证书认证的 Replication 会把提交的事务应用到所有节点上。基于证书认证的 Replication 是使用组通信和事务排序技术来进行同步数据库 Replication 的一种方法。

最小的 Galera 集群由3个节点组成，建议使用奇数个节点运行。原因是，如果在一个节点上应用事务出现问题(例如，网络问题或计算机变得没有响应) ，另外两个节点将有仲裁(即多数) ，并能够继续进行事务提交。

下图展示了 MySQL Replication 和 Galera Cluster 之间的一些差异：
{{< figure src="https://cdn.jsdelivr.net/gh/jackbai233/image-hosting@master/20211024/dha_galera.5842p8tgjuo0.webp" title="dha_galera" >}}
      
Galera Cluster 相较于传统的 MySQL Replication 有以下好处：
- 具有同步复制、故障转移和重新同步的高可用性解决方案。
- 所有服务器都有最新数据（无从属滞后）。
- “相当不错”的写入可扩展性和读取可扩展性。
- 跨数据中心的高可用性。

当然它也有一些限制：
- 它仅支持 InnoDB 或 XtraDB 存储引擎。
- 随着可写 master 数量的增加，事务回滚率可能会增加，尤其是在同一数据集（又名热点）上存在写争用的情况下。这增加了事务延迟。
- 慢速/过载的主节点可能会影响 Galera Cluster 的性能，因此建议在整个集群中使用统一的服务器。

#### 代理层
通过这样或那样的方式设置 MySQL 是不足以实现高可用性的。下一步将是解决另一个问题：“我应该如何连接到数据库层，以便我总是连接到主机是一直可用的？”，其中一个答案就是使用代理层。
##### HAProxy
HAProxy 大概是 MySQL 世界里最流行的一个TCP/HTTP 负载平衡器。它非常轻快、易配置。它是免费、快速并且可靠的一种解决方案。对于 MySQL Replication 或者 Galera Cluster，我们可以利用 haproxy 在 tcp 层对数据库的读请求进行代理，从而实现多个库的负载均衡。其工作的示意图如下：
{{< figure src="https://cdn.jsdelivr.net/gh/jackbai233/image-hosting@master/20211024/dha_haproxy.4q0xkm7qjxg0.webp" title="haproxy" >}}
    
##### MaxScale
另一个最新的代理方案时 MaxScale。它是一款由 mariadb 公司出品的中间件 Maxscale，该中间件能实现读写分离和读负载均衡，安装和配置都十分简单。与 HAProxy 的主要区别在于 MaxScale 是数据库感知的。它旨在与 MySQL 一起使用，并为 DBA 提供了更大的灵活性。除了作为代理之外，它还具有大量功能。例如，如果您需要一个 binlog 服务器，MaxScale 可以在这里为您提供帮助。但从 HA 的角度来看，最重要的特性是它能够理解 MySQL 状态。如果您使用传统的 Mysql Replication，MaxScale 将能够确定哪个节点是主节点，哪个节点是从节点。在故障转移的情况下，这可以减少需要记住的配置更改。
{{< figure src="https://cdn.jsdelivr.net/gh/jackbai233/image-hosting@master/20211024/dha_maxscale.2kz798r90ma0.webp" title="maxscale" >}}
    
## 总结
MySQL 中的高可用性是一个复杂的主题，需要大量的研究，并且很大程度上取决于您使用的环境。比起一篇文章，可能更适合以一本书来讨论，上述只是列出了一些主要的方案，具体的部署细节并未给出，更多详情可以自行查询搜索。
  
## 附录
1. MySQL 高可用方案介绍：[https://severalnines.com/database-blog/overview-mysql-database-high-availability](https://severalnines.com/database-blog/overview-mysql-database-high-availability)
2. MySQL 高可用方案选择：[https://www.percona.com/blog/2016/06/07/choosing-mysql-high-availability-solutions/](https://www.percona.com/blog/2016/06/07/choosing-mysql-high-availability-solutions/)
3. MySQL 高可用性可扩展性官方介绍：[https://www.percona.com/blog/2016/06/07/choosing-mysql-high-availability-solutions/](https://www.percona.com/blog/2016/06/07/choosing-mysql-high-availability-solutions/)
4. DRBD 工作原理：[https://www.cnblogs.com/chenghuan/articles/7531984.html](https://www.cnblogs.com/chenghuan/articles/7531984.html)
5. 高可用架构之MHA：[https://www.cnblogs.com/gomysql/p/3675429.html](https://www.cnblogs.com/gomysql/p/3675429.html)
6. 官方MySQL Replication介绍：[https://dev.mysql.com/doc/refman/5.6/en/replication.html](https://dev.mysql.com/doc/refman/5.6/en/replication.html)
7. MySQL Replication 原理：[https://www.cnblogs.com/nulige/p/9491850.html](https://www.cnblogs.com/nulige/p/9491850.html)
8. Galera Cluster 教程：[https://severalnines.com/resources/database-management-tutorials/galera-cluster-mysql-tutorial](https://severalnines.com/resources/database-management-tutorials/galera-cluster-mysql-tutorial)
9. Galera Cluster 原理：[https://segmentfault.com/a/1190000013652043](https://segmentfault.com/a/1190000013652043)
10. 官方 MySQL NDB Cluster 介绍：[https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster.html](https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster.html)
# k8s hosPort网络模式不生效


本文主要介绍了在k8s中，采用flannel部署网络后，当pod中容器的网络模式为`hostPort`时，其结果并未生效的问题，并就该问题进行了分析解决。

<!--more-->

## 介绍

首先，**hostPort** 是直接将容器的端口与所调度的k8s工作节点的端口路由，这样用户就可以通过对应工作节点的ip加端口来访问应用了。如：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: whoami
spec:
  containers:
    - name: whami
      image: whoami
      ports:
        - containerPort: 8080
          hostPort: 8080
```
## 问题定位
当我们将`hostPort`网络模式的应用通过 `kubectl`命令放入到k8s中后，若在工作节点上采用对应端口( 如8080 )访问时发现应用服务未跑起来，这时我们可以先看一下自己的CNI网络插件portmap( 见[官网 ](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/))是否正常，通过 `journalctl -fu kubelet`命令返回的日志可以进行查看，如果CNI网络插件未安装，我们看下自己当时的的flannel.yaml文件中是否设置了 **portMappings capability** 属性的值为true，如下：
```yaml
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.0",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "nodename": "127.0.0.1",
      "ipam": {
        "type": "host-local",
        "subnet": "usePodCidr"
      },
      "policy": {
        "type": "k8s"
      },
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true}
    }
  ]
}
```
如果本来是true，在apply flannel.yaml文件后还是没有CNI插件，则可以根据上述日志中的路径从其他k8s节点上复制一份。
当网络插件正常后，若还是无法访问应用，则可以在应用被调度的工作节点上执行如下命令，查看iptables中的nat表中生成的CNI-HOSTPORT-DNAT：

```shell
# iptables -nvL CNI-HOSTPORT-DNAT -t nat
Chain CNI-HOSTPORT-DNAT (2 references)
 pkts bytes target     prot opt in     out     source               destination
    4   240 CNI-DN-550b4bf3691bef6919331  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* dnat name: "podman" id: "aea317babc7f3ec7607b242227b39c90a43b50a7dd0f0b8fbccc82620370831d" */ multiport dports 8080
```

假如当前k8s中只部署了一个hostPort模式的应用，则通过以上命令只会看到一个类似 *CNI-DN-550b4bf3691bef6919331* 的数据，若是有多个，则就说明有问题，应用未能正确访问的原因就是生成的CNI网络被占用了（笔者这里就是因为出现了两个，所以导致自己的应用无法正常访问）。这时就需要清理iptables，采用如下命令对iptables进行清理：

```shell
root@user:/# iptables -t nat -F 
root@user:/# iptables -t nat -X
root@user:/# iptables -t nat -nvL
```

现在再重新部署应用并在对应的工作节点上查看iptables的nat表是否正常，若只有一条，再尝试访问应用。若还是不能正常访问，则查看iptables对应表中的 *CNI-DN-550b4bf3691bef6919331* 数据的规则是否完整，可以执行以下命令：

```shell
[root@user ~]# iptables -n -t nat -L CNI-DN-550b4bf3691bef6919331
Chain CNI-DN-66a2679082f9abc22ecf1 (1 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8086 to:10.244.3.254:8080
```

此时可以再尝试如下命令，将对应的工作节点（如ip为10.23.34.56）再绑定一遍：

```shell
[root@user ~]# iptables -t nat -R  CNI-DN-550b4bf3691bef6919331 1 -p tcp -d 10.23.34.56 --dport 8080 -j DNAT --to-destination 10.244.3.254:8086
```

注意上面的工作节点ip及8080端口替换为自己实际的ip和端口。

## 参考

1. [官方nodePort介绍](https://kubernetes.io/zh/docs/concepts/services-networking/service/#nodeport)
2. [hostPort异常排查](http://liupeng0518.github.io/2018/12/29/k8s/Network/%E5%BC%82%E5%B8%B8%E6%8E%92%E9%94%99/)
3. [查看iptables表](https://arusso.io/Podman_CNI_Networking_on_CentOS_7/)
4. [iptables清理](https://www.jianshu.com/p/c8f6fb0314cb)

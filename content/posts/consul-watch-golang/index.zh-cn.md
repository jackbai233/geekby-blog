---
weight: 4
title: "使用Consul的watch机制监控服务的变化"
date: 2021-07-10T11:09:53+08:00
lastmod: 2021-07-10T11:09:53+08:00
draft: false
author: "JackBai"
authorLink: "https://www.geekby.cn"
description: "这篇文章讲述了Consul的watch用法，以及使用Golang来监控Consul中service的变化."
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["Consul", "Golang"]
categories: ["分布式"]

lightgallery: true
---

这篇文章讲述了Consul的watch用法，以及使用Golang来监控Consul中service的变化.

<!--more-->

## 前言
consul常常被用来作服务注册与服务发现，而它的watch机制则可被用来监控一些数据的更新，包括：nodes, KV pairs, health checks等。另外，在监控到数据变化后，还可以调用外部处理程序，此处理程序可以是任何可执行文件或HTTP调用，具体说明可见[官网](https://www.consul.io/docs/dynamic-app-config/watches)。
## watch 探究
根据官方描述，watch的实现依赖于consul的 [HTTP API](https://www.consul.io/api) 的 **[blocking queries](https://www.consul.io/api-docs/features/blocking)** 。引用官方的文档：
> Many endpoints in Consul support a feature known as "blocking queries". A blocking query is used to wait for a potential change using long polling. Not all endpoints support blocking, but each endpoint uniquely documents its support for blocking queries in the documentation.

> Endpoints that support blocking queries return an HTTP header named `X-Consul-Index`. This is a unique identifier representing the current state of the requested resource.

> On subsequent requests for this resource, the client can set the `index` query string parameter to the value of `X-Consul-Index`, indicating that the client wishes to wait for any changes subsequent to that index.

也就是说，blocking queries 属于长轮询的一种，如果 consul 的 http api (官方称作endpoint)支持 blocking queries，那么之后对该api 进行请求的时候添加`index`参数，那么该请求将会被阻塞，知道其请求的数据有变化时才会响应结果，换句话说，就是对该 endpoint 启动了长轮询。
长轮训减少了频繁轮训的所造成的不必要的带宽和服务器资源开销，用在服务发现上，即时性也能有所保证。
说了这么多，让我们来尝试一下：

1. 以生产环境启动consul的一个agent：
```shell
$ ./consul agent -dev
```

2. 在consul中添加一个键值对：
```shell
./consul kv put loglevel ERROR
```

3. 启动一个访问consul kv数据中心的 http api：
```shell
curl -v http://localhost:8500/v1/kv/loglevel
```
在我这里，该api请求返回的响应头中的 `X-Consul-Index`的值为9981

4. 对上一步的api添加index请求参数继续请求：
```shell
curl -v http://localhost:8500/v1/kv/loglevel?index=9981
```
  此时该请求将会被阻塞。

5. 在另一个terminal命令行中，更新第二步添加的key的值：
```shell
./consul kv put loglevel INFO
```
  此时，第四步被阻塞的请求不再阻塞，并立马返回响应。

上述步骤演示了watch利用 blocking queries 对KV进行监控，而正如文章开始说的，watch还支持对其他数据类型进行监控，其监控的数据类型有以下这些：
- `key` - Watch a specific KV pair
- `keyprefix` - Watch a prefix in the KV store
- `services` - Watch the list of available services
- `nodes` - Watch the list of nodes
- `service`- Watch the instances of a service
- `checks` - Watch the value of health checks
- `event` - Watch for custom user events
其中对各个监控类型的使用方法，见[官方文档](https://www.consul.io/docs/dynamic-app-config/watches#watch-types)
## Golang 实现watch 对服务变化的监控
consul官方提供了[Golang版的watch包](https://pkg.go.dev/github.com/hashicorp/consul/api/watch)。其实际上也是对watch机制进行了一层封装，最终代码实现的还是对consul HTTP API 的 `endpoints`的使用。
文章开始说过，“在监控到数据变化后，还可以调用外部处理程序”。是了，数据变化后调用外部处理程序才是有意义的，Golang的watch包中对应的外部处理程序是一个函数handler。因为业务的关系，这里只实现了watch对service的变化的监控，话不多说，上代码：
```golang
package main

import (
	"fmt"
	consulapi "github.com/hashicorp/consul/api"
	"github.com/hashicorp/consul/api/watch"
	"sync"
)

// watch包的使用方法为使用watch.Parse(查询参数)生成Plan，绑定Plan的handler，运行Plan

// 定义Consul的结构体
type ConsulRegistry struct {
	Address  string                             // consul agent 的地址："127.0.0.1:8500"
	Services map[string]*consulapi.ServiceEntry // 存储consul启动后其所有注册的服务
	RWMutex  *sync.RWMutex                      // 因存在同时对services进行读写，故对Services需要加锁
}

// 定义watcher
type Watcher struct {
	Wp       *watch.Plan            // 总的Services变化对应的Plan
	watchers map[string]*watch.Plan // 存储每个service变化对应的Plan
}

// 创建ConsulRegistry
func newConsulRegistry(consulAddr string) *ConsulRegistry {
	return &ConsulRegistry{
		Address:  consulAddr,
		Services: make(map[string]*consulapi.ServiceEntry),
		RWMutex:  new(sync.RWMutex),
	}
}

// 将consul新增的service加入，并监控
func (w *Watcher) registerServiceWatcher(serviceName string, registry *ConsulRegistry) error {
	// watch endpoint 的请求参数，具体见官方文档：https://www.consul.io/docs/dynamic-app-config/watches#service
	wp, err := watch.Parse(map[string]interface{}{
		"type":    "service",
		"service": serviceName,
	})
	if err != nil {
		return err
	}

	// 定义service变化后所执行的程序(函数)handler
	wp.Handler = func(idx uint64, data interface{}) {
		switch d := data.(type) {
		case []*consulapi.ServiceEntry:
			for _, i := range d {
				// 这里只是简单打印了一下变化的service名字
				fmt.Printf("service %s 已变化", i.Service.Service)
				// 添加新增的service
				registry.RWMutex.Lock()
				registry.Services[i.Service.Service] = i
				registry.RWMutex.Unlock()
			}
		}
	}
	// 启动监控
	go wp.Run(registry.Address)
	// 将已启动监控的service加入
	w.watchers[serviceName] = wp

	return nil
}

func (cr *ConsulRegistry) NewWatcher(watchType string, opts map[string]string) (*Watcher, error) {
	var options = map[string]interface{}{
		"type": watchType,
	}
	// 组装请求参数。(监控类型不同，其请求参数不同)
	for k, v := range opts {
		options[k] = v
	}

	wp, err := watch.Parse(options)
	if err != nil {
		return nil, err
	}

	w := &Watcher{
		Wp:       wp,
		watchers: make(map[string]*watch.Plan),
	}

	wp.Handler = func(idx uint64, data interface{}) {
		switch d := data.(type) {
		// 这里只实现了对services的监控，其他监控的data类型判断参考：https://github.com/dmcsorley/avast/blob/master/consul.go
		// services
		case map[string][]string:
			for i := range d {
				// 为什么会多一个consul，参考官方文档services监听的返回值：https://www.consul.io/docs/dynamic-app-config/watches#services
				if i == "consul" {
					continue
				}
				// 如果该service已经加入到ConsulRegistry的services里监控了，就不再加入
				if _, ok := w.watchers[i]; ok {
					continue
				}

				w.registerServiceWatcher(i, cr)
			}

			// 读取ConsulRegistry中的services，移除已下线的服务
			cr.RWMutex.RLock()
			rs := cr.Services
			cr.RWMutex.RUnlock()

			// remove unknown services from registry
			for s := range rs {
				if _, ok := d[s]; !ok {
					cr.RWMutex.Lock()
					delete(cr.Services, s)
					cr.RWMutex.Unlock()
				}
			}

			// remove unknown services from watchers
			for i, svc := range w.watchers {
				if _, ok := d[i]; !ok {
					svc.Stop()
					delete(w.watchers, i)
				}
			}
		default:
			fmt.Printf("不能判断监控的数据类型: %v", &d)
		}
	}

	return w, nil
}

func (cr *ConsulRegistry) RegisterWatcher(watchType string, opts map[string]string) error {
	w, err := cr.NewWatcher(watchType, opts)
	if err != nil {
		fmt.Println(err)
		return err
	}
	go w.Wp.Run(cr.Address)

	return nil
}

func main() {
	address := "127.0.0.1:8500"
	cr := newConsulRegistry(address)
	cr.RegisterWatcher("services", nil)
}

```
当启动consul Agent后，运行上述代码，然后注册服务(服务注册的代码可利用搜索引擎)，观察代码运行的结果。
## 参考资料
1. [Consul-Watches](https://www.consul.io/docs/dynamic-app-config/watches#watches)
2. [Immediate response on changes in Consul](https://medium.com/@DevRaccoon/immediate-response-on-changes-in-consul-234e63689308)
3. [玩转CONSUL(1)–WATCH机制探究](http://vearne.cc/archives/13983)
4. [avast/consul.go](https://github.com/dmcsorley/avast/blob/master/consul.go)


---
weight: 2
title: "[转]Go语言实现SSH远程终端及WebSocket"
date: 2021-10-24T17:02:53+08:00
lastmod: 2021-10-24T17:02:53+08:00
draft: false
author: "JackBai"
authorLink: "https://www.geekby.cn"
description: "这篇文章介绍了使用golang实现ssh远程终端."
featuredImage: "https://cdn.jsdelivr.net/gh/jackbai233/image-hosting@master/20211024/golang-ssh.2jel1ppf5va0.png"

tags: ["Golang", "WebSocket"]
categories: ["后端"]

lightgallery: true
---

本文主要介绍了使用golang结合webSocket通信原理来实现远程ssh终端的方法.

<!--more-->

{{< admonition type=tip title="声明" open=true >}}
本文转载至[https://www.cnblogs.com/you-men/p/13934845.html](https://www.cnblogs.com/you-men/p/13934845.html)
{{< /admonition >}}

## 使用
### 下载
```shell
go get "github.com/mitchellh/go-homedir"
 go get "golang.org/x/crypto/ssh"
```
### 使用密码认证连接
连接包含了认证,可以使用password或者sshkey 两种方式认证,下面采用密码认证方式完成连接
```go
package main

import (
	"fmt"
	"golang.org/x/crypto/ssh"
	"log"
	"time"
)

func main()  {
	sshHost := "39.108.140.0"
	sshUser := "root"
	sshPasswrod := "youmen"
	sshType := "password"  // password或者key
	//sshKeyPath := "" // ssh id_rsa.id路径
	sshPort := 22

	// 创建ssh登录配置
	config := &ssh.ClientConfig{
		Timeout: time.Second, // ssh连接time out时间一秒钟,如果ssh验证错误会在一秒钟返回
		User: sshUser,
		HostKeyCallback: ssh.InsecureIgnoreHostKey(),  // 这个可以,但是不够安全
		//HostKeyCallback: hostKeyCallBackFunc(h.Host),
	}
	if sshType == "password" {
		config.Auth = []ssh.AuthMethod{ssh.Password(sshPasswrod)}
	} else {
		//config.Auth = []ssh.AuthMethod(publicKeyAuthFunc(sshKeyPath))
		return
	}

	// dial 获取ssh client
	addr := fmt.Sprintf("%s:%d",sshHost,sshPort)
	sshClient,err := ssh.Dial("tcp",addr,config)
	if err != nil {
		log.Fatal("创建ssh client 失败",err)
	}
	defer sshClient.Close()

	// 创建ssh-session
	session,err := sshClient.NewSession()
	if err != nil {
		log.Fatal("创建ssh session失败",err)
	}

	defer session.Close()

	// 执行远程命令
	combo,err := session.CombinedOutput("whoami; cd /; ls -al;")
	if err != nil {
		log.Fatal("远程执行cmd失败",err)
	}
	log.Println("命令输出:",string(combo))
}

//func publicKeyAuthFunc(kPath string) ssh.AuthMethod  {
//	keyPath ,err := homedir.Expand(kPath)
//	if err != nil {
//		log.Fatal("find key's home dir failed",err)
//	}
//
//	key,err := ioutil.ReadFile(keyPath)
//	if err != nil {
//		log.Fatal("ssh key file read failed",err)
//	}
//
//	signer,err := ssh.ParsePrivateKey(key)
//	if err != nil {
//		log.Fatal("ssh key signer failed",err)
//	}
//	return ssh.PublicKeys(signer)
//}
```
代码解读：
```go
// 配置ssh.ClientConfig
/*
		建议TimeOut自定义一个比较端的时间
		自定义HostKeyCallback如果像简便就使用ssh.InsecureIgnoreHostKey会带哦,这种方式不是很安全
		publicKeyAuthFunc 如果使用key登录就需要用哪个这个函数量读取id_rsa私钥, 当然也可以自定义这个访问让他支持字符串.
*/

// ssh.Dial创建ssh客户端
/*
		拼接字符串得到ssh链接地址,同时不要忘记defer client.Close()
*/

// sshClient.NewSession创建会话
/*
		可以自定义stdin,stdout
		可以创建pty
		可以SetEnv
*/

// 执行命令CombinnedOutput run...
go run main.go
2020/11/06 00:07:31 命令输出: root
total 84
dr-xr-xr-x. 20 root  root   4096 Sep 28 09:38 .
dr-xr-xr-x. 20 root  root   4096 Sep 28 09:38 ..
-rw-r--r--   1 root  root      0 Aug 18  2017 .autorelabel
lrwxrwxrwx.  1 root  root      7 Aug 18  2017 bin -> usr/bin
dr-xr-xr-x.  4 root  root   4096 Sep 12  2017 boot
drwxrwxr-x   2 rsync rsync  4096 Jul 29 23:37 data
drwxr-xr-x  19 root  root   2980 Jul 28 13:29 dev
drwxr-xr-x. 95 root  root  12288 Nov  5 23:46 etc
drwxr-xr-x.  5 root  root   4096 Nov  3 16:11 home
lrwxrwxrwx.  1 root  root      7 Aug 18  2017 lib -> usr/lib
lrwxrwxrwx.  1 root  root      9 Aug 18  2017 lib64 -> usr/lib64
drwx------.  2 root  root  16384 Aug 18  2017 lost+found
drwxr-xr-x.  2 root  root   4096 Nov  5  2016 media
drwxr-xr-x.  3 root  root   4096 Jul 28 21:01 mnt
drwxr-xr-x   4 root  root   4096 Sep 28 09:38 nginx_test
drwxr-xr-x.  8 root  root   4096 Nov  3 16:10 opt
dr-xr-xr-x  87 root  root      0 Jul 28 13:26 proc
dr-xr-x---. 18 root  root   4096 Nov  4 00:38 root
drwxr-xr-x  27 root  root    860 Nov  4 21:57 run
lrwxrwxrwx.  1 root  root      8 Aug 18  2017 sbin -> usr/sbin
drwxr-xr-x.  2 root  root   4096 Nov  5  2016 srv
dr-xr-xr-x  13 root  root      0 Jul 28 21:26 sys
drwxrwxrwt.  8 root  root   4096 Nov  5 03:09 tmp
drwxr-xr-x. 13 root  root   4096 Aug 18  2017 usr
drwxr-xr-x. 21 root  root   4096 Nov  3 16:10 var

```
以上内容摘自：[https://mojotv.cn/2019/05/22/golang-ssh-session](https://mojotv.cn/2019/05/22/golang-ssh-session)

## WebSocket简介
HTML5开始提供的一种浏览器与服务器进行双工通讯的网络技术,属于应用层协议,它基于TCP传输协议,并复用HTTP的握手通道:

对大部分web开发者来说,上面描述有点枯燥,只需要几下以下三点
```
/*
		1. WebSocket可以在浏览器里使用
		2. 支持双向通信
		3. 使用很简单
*/
```
### 优点
对比HTTP协议的话,概括的说就是: 支持双向通信,更灵活,更高效,可扩展性更好
```
/*
		1. 支持双向通信,实时性更强
		2. 更好的二进制支持
		3. 较少的控制开销,连接创建后,客户端和服务端进行数据交换时,协议控制的数据包头部较小,在不包含头部的情况下,
				服务端到客户端的包头只有2-10字节(取决于数据包长度), 客户端到服务端的话,需要加上额外4字节的掩码,
				而HTTP每次同年高新都需要携带完整的头部
		4. 支持扩展,ws协议定义了扩展, 用户可以扩展协议, 或者实现自定义的子协议
*/

```

## 基于web的Terminal终端控制台
完成这样一个Web Terminal的目的主要是解决几个问题:
```
/*
		1. 一定程度上取代xshell，secureRT，putty等ssh终端
		2. 可以方便身份认证, 访问控制
		3. 方便使用, 不受电脑环境的影响
*/
```
要实现远程登录的功能,其数据流向大概为
```
/*
		浏览器 <-->  WebSocket  <---> SSH <---> Linux OS
*/
```
### 实现流程
- 1. 浏览器将主机的信息(ip, 用户名, 密码, 请求的终端大小等)进行加密, 传给后台, 并通过HTTP请求与后台协商升级协议. 协议升级完成后, 后续的数据交换则遵照web Socket的协议.
- 2. 后台将HTTP请求升级为web Socket协议, 得到一个和浏览器数据交换的连接通道
- 3. 后台将数据进行解密拿到主机信息, 创建一个SSH 客户端, 与远程主机的SSH 服务端协商加密, 互相认证, 然后建立一个SSH Channel
- 4. 后台和远程主机有了通讯的信道, 然后后台将终端的大小等信息通过SSH Channel请求远程主机创建一个 pty(伪终端), 并请求启动当前用户的默认 shell
- 5. 后台通过 Socket连接通道拿到用户输入, 再通过SSH Channel将输入传给pty, pty将这些数据交给远程主机处理后按照前面指定的终端标准输出到SSH Channel中, 同时键盘输入也会发送给SSH Channel
- 6. 后台从SSH Channel中拿到按照终端大小的标准输出后又通过Socket连接将输出返回给浏览器, 由此变实现了Web Terminal

![avatar](https://cdn.jsdelivr.net/gh/jackbai233/image-hosting@master/20211024/image.54fxpcesfi00.png)


![avatar2](https://cdn.jsdelivr.net/gh/jackbai233/image-hosting@master/20211024/image.2qsqdsa5h4o0.png)

按照上面的使用流程基于代码解释如何实现
### 升级HTTP协议为WebSocket
```go
var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin: func(r *http.Request) bool {
		return true
	},
}
```
### 升级协议并获得socket连接
```go
conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
if err != nil {
    c.Error(err)
    return
}
```
conn就是socket连接通道, 接下来后台和浏览器之间的通讯都将基于这个通道
### 后台拿到主机信息，建立ssh客户端
ssh客户端结构体
```go
type SSHClient struct {
	Username  string `json:"username"`
	Password  string `json:"password"`
	IpAddress string `json:"ipaddress"`
	Port      int    `json:"port"`
	Session   *ssh.Session
	Client    *ssh.Client
	channel   ssh.Channel
}

//创建新的ssh客户端时, 默认用户名为root, 端口为22
func NewSSHClient() SSHClient {
	client := SSHClient{}
	client.Username = "root"
	client.Port = 22
	return client
}
```
初始化的时候我们只有主机的信息, 而Session, client, channel都是空的, 现在先生成真正的client:
```go
func (this *SSHClient) GenerateClient() error {
	var (
		auth         []ssh.AuthMethod
		addr         string
		clientConfig *ssh.ClientConfig
		client       *ssh.Client
		config       ssh.Config
		err          error
	)
	auth = make([]ssh.AuthMethod, 0)
	auth = append(auth, ssh.Password(this.Password))
	config = ssh.Config{
		Ciphers: []string{"aes128-ctr", "aes192-ctr", "aes256-ctr", "aes128-gcm@openssh.com", "arcfour256", "arcfour128", "aes128-cbc", "3des-cbc", "aes192-cbc", "aes256-cbc"},
	}
	clientConfig = &ssh.ClientConfig{
		User:    this.Username,
		Auth:    auth,
		Timeout: 5 * time.Second,
		Config:  config,
		HostKeyCallback: func(hostname string, remote net.Addr, key ssh.PublicKey) error {
			return nil
		},
	}
	addr = fmt.Sprintf("%s:%d", this.IpAddress, this.Port)
	if client, err = ssh.Dial("tcp", addr, clientConfig); err != nil {
		return err
	}
	this.Client = client
	return nil
}

```
ssh.Dial(“tcp”, addr, clientConfig)创建连接并返回客户端, 如果主机信息不对或其它问题这里将直接失败
### 通过ssh客户端创建ssh channel，并请求pty终端，请求用户默认会话
如果主机信息验证通过, 可以通过ssh client创建一个通道:
```go
channel, inRequests, err := this.Client.OpenChannel("session", nil)
if err != nil {
    log.Println(err)
    return nil
}
this.channel = channel
```
ssh通道创建完成后, 请求一个标准输出的终端, 并开启用户的默认shell:
```go
ok, err := channel.SendRequest("pty-req", true, ssh.Marshal(&req))
if !ok || err != nil {
    log.Println(err)
    return nil
}
ok, err = channel.SendRequest("shell", true, nil)
if !ok || err != nil {
    log.Println(err)
    return nil
}
```
### 远程主机与浏览器实时数据交换
现在为止建立了两个通道, 一个是websocket, 一个是ssh channel, 后台将起两个主要的协程, 一个不停的从websocket通道里读取用户的输入, 并通过ssh channel传给远程主机:
```go
//这里第一个协程获取用户的输入
go func() {
    for {
        // p为用户输入
        _, p, err := ws.ReadMessage()
        if err != nil {
            return
        }
        _, err = this.channel.Write(p)
        if err != nil {
            return
        }
    }
}()
```
第二个主协程将远程主机的数据传递给浏览器, 在这个协程里还将起一个协程, 不断获取ssh channel里的数据并传给后台内部创建的一个通道, 主协程则有一个死循环, 每隔一段时间从内部通道里读取数据, 并将其通过websocket传给浏览器, 所以数据传输并不是真正实时的,而是有一个间隔在, 我写的默认为100微秒, 这样基本感受不到延迟, 而且减少了消耗, 有时浏览器输入一个命令获取大量数据时, 会感觉数据出现会一顿一顿的便是因为设置了一个间隔:
```go
//第二个协程将远程主机的返回结果返回给用户
go func() {
    br := bufio.NewReader(this.channel)
    buf := []byte{}
    t := time.NewTimer(time.Microsecond * 100)
    defer t.Stop()
    // 构建一个信道, 一端将数据远程主机的数据写入, 一段读取数据写入ws
    r := make(chan rune)

    // 另起一个协程, 一个死循环不断的读取ssh channel的数据, 并传给r信道直到连接断开
    go func() {
        defer this.Client.Close()
        defer this.Session.Close()

        for {
            x, size, err := br.ReadRune()
            if err != nil {
                log.Println(err)
                ws.WriteMessage(1, []byte("\033[31m已经关闭连接!\033[0m"))
                ws.Close()
                return
            }
            if size > 0 {
                r <- x
            }
        }
    }()

    // 主循环
    for {
        select {
        // 每隔100微秒, 只要buf的长度不为0就将数据写入ws, 并重置时间和buf
        case <-t.C:
            if len(buf) != 0 {
                err := ws.WriteMessage(websocket.TextMessage, buf)
                buf = []byte{}
                if err != nil {
                    log.Println(err)
                    return
                }
            }
            t.Reset(time.Microsecond * 100)
        // 前面已经将ssh channel里读取的数据写入创建的通道r, 这里读取数据, 不断增加buf的长度, 在设定的 100 microsecond后由上面判定长度是否返送数据
        case d := <-r:
            if d != utf8.RuneError {
                p := make([]byte, utf8.RuneLen(d))
                utf8.EncodeRune(p, d)
                buf = append(buf, p...)
            } else {
                buf = append(buf, []byte("@")...)
            }
        }
    }
}()

```
web terminal的后台建好了
### 前端
前端我选择用了vue框架(其实这么小的项目完全不用vue), 终端工具用的是xterm, vscode内置的终端也是采用的xterm.这里贴一段关键代码, [前端项目地址](https://github.com/chengjoey/web-terminal-client)
```javascript
mounted () {
    var containerWidth = window.screen.height;
    var containerHeight = window.screen.width;
    var cols = Math.floor((containerWidth - 30) / 9);
    var rows = Math.floor(window.innerHeight/17) - 2;
    if (this.username === undefined){
        var url = (location.protocol === "http:" ? "ws" : "wss") + "://" + location.hostname + ":5001" + "/ws"+ "?" + "msg=" + this.msg + "&rows=" + rows + "&cols=" + cols;
    }else{
        var url = (location.protocol === "http:" ? "ws" : "wss") + "://" + location.hostname + ":5001" + "/ws"+ "?" + "msg=" + this.msg + "&rows=" + rows + "&cols=" + cols + "&username=" + this.username + "&password=" + this.password;
    }
    let terminalContainer = document.getElementById('terminal')
    this.term = new Terminal()
    this.term.open(terminalContainer)
    // open websocket
    this.terminalSocket = new WebSocket(url)
    this.terminalSocket.onopen = this.runRealTerminal
    this.terminalSocket.onclose = this.closeRealTerminal
    this.terminalSocket.onerror = this.errorRealTerminal
    this.term.attach(this.terminalSocket)
    this.term._initialized = true
    console.log('mounted is going on')
}

```
{{< admonition type=note title="注意" open=true >}}
在使用xterm时需要将其css文件也导入，不然会有显示问题
{{< /admonition >}}

## 参考
1. [gin-web-machine-ws](https://github.com/piupuer/gin-web/blob/dev/api/v1/sys_machine_ws.go)
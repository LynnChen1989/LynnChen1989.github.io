---
layout: post
title: TCP队列溢出-全连接
subtitle: 
categories: 网络原理
tags: [network]
---


## 全连接队列

## 0.半连接全连接队列示意

<img src="/assets/images/tcp/tcp-comm.png" width="100%" height="100%" alt="" />


## 1.对于 LISTEN 状态的 socket `(ss -lnt)`

    Recv-Q：当前全连接队列的占用大小，即已完成三次握手等待应用程序 accept() 的 TCP 链接
    Send-Q：全连接队列的最大长度，即全连接队列的大小

## 2.对于非 LISTEN 状态的 socket `(ss -nt)`

    Recv-Q：已收到但未被应用程序读取的字节数
    Send-Q：已发送但未收到确认的字节数

## 3.说明

+ （1）当Recv-Q ≤ Send-Q 正常工作，当Recv-Q > Send-Q 时数据包被DROP或者回复RST。
+ （2）可以通过设置/proc/sys/net/ipv4/tcp_abort_on_overflow使 Server 端在全连接队列满时，向 Client 端发送 RST 报文。 tcp_abort_on_overflow 有两种可选值：

        0：如果全连接队列满了，Server 端 DROP Client 端回复的 ACK
        1：如果全连接队列满了，Server 端向 Client 端发送 RST 报文，终止 TCP socket 链接

+ (3) netstat 命令通过netstat -s命令可以查看 TCP 半连接队列、全连接队列的溢出情况

        $ netstat -s | grep -i "listen"
            189088 times the listen queue of a socket overflowed
            30140232 SYNs to LISTEN sockets dropped

+ (4) 全连接队列最大长度控制TCP 全连接队列的最大长度由min(somaxconn, backlog)控制，其中： somaxconn 是 Linux 内核参数，由 /proc/sys/net/core/somaxconn
  指定backlog 是 TCP 协议中 listen 函数的参数之一，即 int listen(int sockfd, int backlog) 函数中的 backlog 大小。在 Golang 中，listen 的 backlog
  参数使用的是/proc/sys/net/core/somaxconn文件中的值。

## 4.关键参数

| 参数                                               | 作用                                        |
|--------------------------------------------------|-------------------------------------------|
| /proc/sys/net/core/somaxconn                     | TCP全连接队列的最大长度Send-Q的设定                    |
| /proc/sys/net/ipv4/tcp_abort_on_overflow         | 当TCP队列满时的处理方式设定（RST/DROP）                 |


## 5.模拟

### 5.1 server代码

```
import (
	"log"
	"net"
	"time"
)

func main() {
	l, err := net.Listen("tcp4", ":8888")
	if err != nil {
		log.Printf("failed to listen due to %v", err)
	}
	defer l.Close()
	log.Println("listen :8888 success")

	for {
		time.Sleep(time.Second * 100)
	}
}

```
### 5.2 client代码

```
package net

import (
	"context"
	"log"
	"net"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

var wg sync.WaitGroup

func establishConn(ctx context.Context, i int) {
	defer wg.Done()
	conn, err := net.DialTimeout("tcp", "10.168.0.30:8888", time.Second*5)
	if err != nil {
		log.Printf("%d, dial error: %v", i, err)
		return
	}
	log.Printf("%d, dial success", i)
	_, err = conn.Write([]byte("hello world"))
	if err != nil {
		log.Printf("%d, send error: %v", i, err)
		return
	}
	select {
	case <-ctx.Done():
		log.Printf("%d, dail close", i)
	}
}

func StartClient() {
	ctx, cancel := context.WithCancel(context.Background())
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go establishConn(ctx, i)
	}

	go func() {
		sc := make(chan os.Signal, 1)
		signal.Notify(sc, syscall.SIGINT)
		select {
		case <-sc:
			cancel()
		}
	}()

	wg.Wait()
	log.Printf("client exit")
}

```
### 5.3 开始模拟

#### 5.3.1 运行服务
```
1、查看系统默认值
# cat /proc/sys/net/core/somaxconn 
4096

2、在默认值下运行server后查看队列情况,
# ss -ant | grep 8888
State      Recv-Q Send-Q    Local Address:Port         Peer Address:Port
LISTEN     0      4096            *:8888                     *:*         

3、临时修改队列大小为5
# echo 5 > /proc/sys/net/core/somaxconn

# cat /proc/sys/net/core/somaxconn       
5

5、修改后运行server
# ss -ant | grep 8888
State      Recv-Q Send-Q    Local Address:Port         Peer Address:Port
LISTEN     0      5            *:8888                     *:* 

6、运行客户端发送10个请求，修改代码中i < 10控制
# go run client.go

```

#### 5.3.2 查看TCP队列情况


##### 5.3.2.1 查看客户端情况

发现客户端有一些成功,一些请求超时，我们得到的结果是有**8个成功2个超时(记住这个结果)**，每次执行的结果可能不一样，但是超时的个数不会超过4个，因为我们的队列能够容忍的上限是6个；

```
=== RUN   TestStartClient
2022/10/08 17:55:05 5, dial success
2022/10/08 17:55:05 7, dial success
2022/10/08 17:55:05 1, dial success
2022/10/08 17:55:05 8, dial success
2022/10/08 17:55:05 0, dial success
2022/10/08 17:55:05 9, dial success
2022/10/08 17:55:05 2, dial success
2022/10/08 17:55:05 6, dial success
2022/10/08 17:55:10 3, dial error: dial tcp 10.168.0.30:8888: i/o timeout
2022/10/08 17:55:10 4, dial error: dial tcp 10.168.0.30:8888: i/o timeout

```



##### 5.3.2.2 查看服务端情况


```
# 在server端执行，马上执行ss -ant | grep 8888，输出如下：
State      Recv-Q Send-Q    Local Address:Port         Peer Address:Port
LISTEN     6      5            *:8888                     *:*                  
SYN-RECV   0      0      10.168.0.30:8888               192.168.10.42:62516              
SYN-RECV   0      0      10.168.0.30:8888               192.168.10.42:62525              
ESTAB      11     0      10.168.0.30:8888               192.168.10.42:62514              
ESTAB      11     0      10.168.0.30:8888               192.168.10.42:62515              
SYN-RECV   0      0      10.168.0.30:8888               192.168.10.42:62523              
SYN-RECV   0      0      10.168.0.30:8888               192.168.10.42:62527              
ESTAB      11     0      10.168.0.30:8888               192.168.10.42:62513              
ESTAB      11     0      10.168.0.30:8888               192.168.10.42:62512              
ESTAB      11     0      10.168.0.30:8888               192.168.10.42:62519              
ESTAB      11     0      10.168.0.30:8888               192.168.10.42:62520
```


发现处于LISTEN状态的，Recv-Q 当前全连接队列的大小是 6 ，大于了Send-Q 全连接队列最大长度是 5，并且出现了大量的半连接状态。


```
# 等待一段时间持续执行ss -ant | grep 8888，输出如下：

LISTEN     6      5            *:8888                     *:*                  
ESTAB      11     0      10.168.0.30:8888               192.168.10.42:62805              
ESTAB      11     0      10.168.0.30:8888               192.168.10.42:62806              
ESTAB      11     0      10.168.0.30:8888               192.168.10.42:62801              
ESTAB      11     0      10.168.0.30:8888               192.168.10.42:62799              
ESTAB      11     0      10.168.0.30:8888               192.168.10.42:62802              
ESTAB      11     0      10.168.0.30:8888               192.168.10.42:62800 
```

我们可以看到处于ESTAB连接状态的只会有6个，达到全连接队列的上限。

##### 5.3.2.3 抓包分析

(1) 在我们客户端发起访问请求的时候，我们通过tcpdump在服务端进行抓包分析，`tcpdump -i ens192 tcp port 8888  -w server.pcap `
将抓取的server.pcap包导入到wireshark进行分析。

(2) 我的客户端是windows，直接在windows中运行wireshark进行抓包。


我们尝试建立10个连接，所以tcp的数据流也只会有10个，一个数据流包含n个数据包，可以通过 `tcp.stream eq 0  ~ tcp.stream eq 9`在wireshark中进行过滤。

记住刚刚从日志打印看到success 8 个 fail 2个，

**但是:**

  （1）实际数据流0-5共6个才是真正成功的，6个就是TCP队列能够容忍的上限；

  （2）6-7的2个是客户端发送了数据但是没有收到server端的确认，客户端以为成功；

  （3）8-9的2个是完全失败的；



**情况一：Client 成功与 Server 端建立 tcp socket 链接，发送数据成功tcp.stream eq 0 ~ tcp.stream eq 5**

`tcp.stream eq 0 ~ tcp.stream eq 5` 实际数据流0-5共6个才是真正成功的，6个就是TCP队列能够容忍的上限；

<img src="/assets/images/tcp/client情况1.png" width="100%" height="100%" alt="" />





**情况二：Client 认为成功与 Server 端建立 tcp socket 连接，后续发送数据失败，持续 RETRY；Server 端认为 TCP 连接未建立，一直在发送SYN+ACK**

`tcp.stream eq 6 ~ tcp.stream eq 7` 的2个是客户端发送了数据但是没有收到server端的确认，客户端以为成功;

<img src="/assets/images/tcp/client情况2.png" width="100%" height="100%" alt="" />

**TCP状态解释** 

 + retransmission 超时重传；
 + spurious retransmission 意味着发送端认为发送的包已经丢失了，然后就重传了，尽管此时接收端已经发送了对这些包的确认（确认还没收到或者已经丢失了）。这是需要在发送端抓包，分析原因。如果接收端以前收到过相同的包，接收端就会报spurious retransmission，但是发送端是不会报spurious retransmission的；
 + dup ack XXX#X 就是重复应答#前的表示报文到哪个序号丢失，#后面的是表示第几次丢失；
 + 

**情况三：Client 向 Server 发送 SYN 未得到响应，一直在 RETRY**

`tcp.stream eq 8 ~ tcp.stream eq 9` 8-9的2个是tcp建立连接完全失败的；

<img src="/assets/images/tcp/client情况3.jpg" width="100%" height="100%" alt="" />


## 一点实用的东西

（1）如果要想知道客户端连接不上服务端，是不是服务端 TCP 全连接队列满的原因，那么可以把 tcp_abort_on_overflow 设置为 1，这时如果在客户端异常中可以看到很多 connection reset by peer 的错误，那么就可以证明是由于服务端 TCP 全连接队列溢出的问题


## 参考

  https://www.cnblogs.com/xiaolincoding/p/12995358.html



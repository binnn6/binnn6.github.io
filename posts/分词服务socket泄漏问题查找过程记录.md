---
title: 线上响应超时问题分析
date: 2019-03-19 01:32:29
tags: 
- 网络编程
- golang
---



# 线上响应超时问题分析

## 现象

调用分词服务的服务发现超时并告警，查看分词服务被调耗时发现一切正常；本机手动请求发现确实存在响应慢的问题。

重启后发现响应正常，重启线上服务，确保线上服务正常；并保留一台进行观察。

## 日志

日志中只一些业务错误的记录，未发现明显导致问题的错误；

## 网络问题

由于分词服务被调耗时正常，根据经验首先是怀疑网络问题：

因为分词服务响应包较小，所以被调的时间是接收到请求的时间到把响应写到tcp发送缓冲区的时间,不包括网络时间。(查看Linux默认tcp写缓冲区为16K，如果响应大于16K，那么被调会包含部分网络时间，但不是全部)

```
#min default max, SO_SNDBUF and SO_RCVBUF 设置的最大值由net.core.wmem_max定义
#使用SO_SNDBUF和SO_RCVBUF设置后，实际申请时会翻倍
#为最大值net.core.wmem_max时也会翻倍
net.ipv4.tcp_mem = 377637	503519	755274
net.ipv4.tcp_rmem = 4096	87380	6291456
net.ipv4.tcp_wmem = 4096	16384	4194304

net.core.rmem_default = 262144  
net.core.wmem_default = 262144 

# SO_SNDBUF and SO_RCVBUF 
net.core.rmem_max = 16777216  
net.core.wmem_max = 16777216  
```

查看主备调监控服务，发现网络流量没有较大的波动；**暂时排除网络问题**。

## 服务运行情况

top查看服务负载正常。没有头绪，但是运气很好。

```
lsof -p `pidof httpseg`	
```

打开文件数确实挺多，很多如下条目：

`httpseg 17125 webdev  149u     sock        0,6      0t0 2289696420 protocol: TCP`

此处开始怀疑是不是打开的文件数超过了服务的限制。

```
ll /proc/`pidof httpseg`/fdinfo|wc -l
1024
cat /proc/`pidof httpseg`/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        2048                 unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             63034                63034                processes
Max open files            1024                 1000000              files
Max locked memory         65536                65536                bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       63034                63034                signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
```

问题：too many open files 

此处有个严重的失误：

**按理说一开始就能发现问题，但是查看日志时只看了业务日志，没有看stdout,stderr的日志，而且直到重启完所有有问题的机器，也没有看这些日志；更悲剧的是这些日志使用的是覆盖的方式，而非追加的方式，导致重启后日志全没了。**

也是在于对golang http的服务相关代码不够了解导致。

## too many open files的影响

代码细节

```
	for {
		rw, e := l.Accept()
		if e != nil {
			select {
			case <-srv.getDoneChan():
				return ErrServerClosed
			default:
			}
			if ne, ok := e.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return e
		}
		tempDelay = 0
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew) // before Serve can return
		go c.serve(ctx)
	}
```


![日志](https://oops-oom.github.io/img/accept_error.png)

**打开文件数超过程序限制会导致accept失败，accept失败后会循环重试，这里的log默认是输出的stdout的。**

### 一些延伸

accept失败会导致accept队列中的连接不能被及时取出，accept队列满了怎么办？

**三次握手产生的：sync队列和accept队列**

**accept是取得accept队列中的Establish状态的连接**

**accept-queue满了怎么办**

accept队列长度： `min(/proc/sys/net/core/somaxconn, backlog)`

> The  backlog argument defines the maximum length to which the queue of pending connections for sockfd may grow.  If a connection request arrives when the queue is full, the client may receive an error with an indication of ECONNREFUSED or, if the underlying protocol supports retransmission, the request may be ignored so that a later reattempt at connection succeeds.

默认情况下，即`/proc/sys/net/ipv4/tcp_abort_on_overflow`为0时，服务端会忽略客户端响应的ack(连接会停留在syn队列)，等待超时，服务端重新发送sync+ack给客户端(`net.ipv4.tcp_synack_retries`)；
`/proc/sys/net/ipv4/tcp_abort_on_overflow`为1时，accept-queue满，服务端会响应rst。

**syn队列满了怎么办**
`/proc/sys/net/ipv4/tcp_max_syn_backlog`

若SYN队列满，则会直接丢弃请求，即新的SYN网络分组会被丢弃；客户端则会超时重传syn.

> 比如syn floods 攻击就是针对半连接队列的，攻击方不停地建连接，但是建连接的时候只做第一步，第二步中攻击方收到server的syn+ack后故意扔掉什么也不做，导致server上这个队列满其它正常请求无法进来

此处没有考虑tcp\_syncookies的影响

**命令查看队列满了的丢包现象**

```bash
netstat -s | egrep "listen|LISTEN" 
19219 times the listen queue of a socket overflowed
19234 SYNs to LISTEN sockets dropped
```

![[图片来自莿鸟栖草堂](https://www.cnxct.com/something-about-phpfpm-s-backlog/)](https://oops-oom.github.io/img/tcp.jpg)

too  many open files导致accept失败会重试导致响应耗时增加，同时accept失败会导致accept队列中的连接不能被及时取出，accept队列会慢；

accept队列满了，server端会丢弃ack，超时后server重发syn + ack，导致耗时增加；

syn队列慢了，server端会丢弃syn，超时后clienth会重发syn，导致耗时增加。

以上原因导致请求分词服务响应会慢，但是由于被调时间是从连接完成开始计算的，所以从被调上是看不出问题的。

## socket泄漏

由于服务是简单的golang提供http服务，调用分词库，所以第一时间就怀疑是分词库的问题。

> 业务背景：分词库会请求鉴权服务进行鉴权，鉴权失败的话，分词库是不能正常使用的。

但是问题是，没有分词服务的源代码。如何定位？

最小化服务，排除干扰，只调用分词库进行分析。

```
strace ./demo
```

![strace输出](https://oops-oom.github.io/img/strace01.png)

分析发现，确实创建了两个socket，但是只close了一个。

这里是采用非阻塞的方式发起连接，此处connect时，返回EINPROGRESS；

直接看man 2 connect:

> EINPROGRESS
> The socket is non-blocking and the connection cannot be completed immediately. It is possible to select(2) or poll(2) for completion by selecting the socket for writing.After select(2) indicates writability, use getsockopt(2) to read the SO\_ERROR option at level SOL\_SOCKET to determine whether connect() completed successfully (SO\_ERROR is zero) or unsuccessfully (SO\_ERROR is one of the usual error codes listed here, explaining the reason for the failure).

服务将对应的fd放入poll侦听可写状态，侦听到可写后调用getsockopt判断是连接成功还是失败。

第一个连接获得的SO\_ERROR为110，查看errno 110 为Connection timed out;

问题就在这个地方，连接失败了，但是没有关闭socket！

第二个连接获得的SO\_ERROR为0，说明连接正常，后续也正常关闭了连接。(问题已反馈给词库开发同学)

## 总结

1. 线上服务qps 100+的服务使用了默认的limit，不应该；
2. 日志不应该重启就丢了，太low；
3. 定位问题的手段：

   日志:  没有查看标准错误日志,里面有明确的错误原因；

   网络状况: 可以看到因为accept队列满而丢包的情况；

   明明可以靠日志或或者现象找到问题，偏偏靠运气才找到。

## 参考

[关于TCP 半连接队列和全连接队列](http://jm.taobao.org/2017/05/25/525-1/)

[高性能网络编程](https://www.taohui.pub/)
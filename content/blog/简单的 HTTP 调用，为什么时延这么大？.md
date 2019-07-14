+++
author = "涤生"
categories = ["Java","HTTP"]
tags = ["HTTP", "时延", "HTTPServer","Nagle","Delayed ACK"]
date = "2019-07-14"
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "简单的 HTTP 调用，为什么时延这么大？"
type = "post"

+++

## 1. 背景

最近项目测试遇到个奇怪的现象，在测试环境通过 Apache HttpClient 调用后端的 HTTP 服务，平均耗时居然接近 **39.2ms**。可能你乍一看觉得这不是很正常吗，有什么好奇怪的？其实不然，我再来说下一些基本信息，该后端的 HTTP 服务并没有什么业务逻辑，只是将一段字符串转成大写然后返回，字符串长度也仅只有 **100** 字符，另外网络 ping 延时只有 **1.9ms** 左右。因此，理论上该调用耗时应该在 **2-3ms** 左右，但为什么平均耗时 **39.2ms** 呢？

![调用时延](/img/2019/07/httpoptimized/invoke-delay.png)
![ping 时延](/img/2019/07/httpoptimized/ping-delay.png)

由于工作原因，调用耗时的问题，对我来说，已经见怪不怪了，经常会帮业务解决内部 RPC 框架调用超时的相关问题，但是 HTTP 调用耗时第一次遇到。不过，排查问题的套路是一样的。主要方法论无外乎由外而内、至上而下等排查方法。我们先来看看外围的一些指标，看能否发现蛛丝马迹。

## 2. 外围指标

### 2.1 系统指标

主要看外围的一些系统指标（注意：**调用与被调用的机器**都要看）。例如负载、CPU。只需一个 top 命令就能一览无余。

因此，确认了下 CPU 和负载都很空闲。由于当时没有截图，这里就不放图了。

### 2.2 进程指标

Java 程序进程指标主要看 GC、线程堆栈情况（注意：**调用与被调用的机器**都要看）。

Young GC 都非常少，而且耗时也在 **10ms** 以内，因此没有长时间的 STW。

因为平均调用时间 39.2ms，比较大，如果耗时是代码导致，线程堆栈应该能发现点啥。看了之后一无所获，服务的相关线程堆栈主要表现是线程池的线程在等任务，这就意味着线程并不忙。

是不是感觉黔驴技穷了，接下来该怎么办呢？

## 3. 本地复现

如果本地（本地是 MAC 系统）能复现，对排查问题也是极好的。

因此在本地使用 Apache HttpClient 写了个简单 Test 程序，直接调用后端的 HTTP 服务，发现平均耗时在 **55ms** 左右。咦，怎么跟测试环境 **39.2ms** 的结果有点区别。主要是本地与测试环境的后端的 HTTP 服务机器跨地区了，ping 时延在 **26ms** 左右，所以延时增大了。不过本地确实也是存在问题的，因为ping 时延是 **26ms**，后端 HTTP 服务逻辑简单，几乎不耗时，因此本地调用平均耗时应该在 **26ms** 左右，为什么是 **55ms**？

是不是越来越迷惑，一头雾水，不知如何下手？

期间怀疑过 Apache HttpClient 是不是有什么地方使用的不对，因此使用 JDK 自带的 HttpURLConnection 写了简单的程序，做了测试，结果一样。

## 4. 诊断

### 4.1 定位

其实从外围的系统指标、进程指标，以及本地复现来看，大致能够断定不是程序上的原因。那 TCP 协议层面呢？

有网络编程经验的同学一定知道 TCP 什么参数会引起这个现象。对，你猜的没错，就是 TCP_NODELAY。

那调用方和被调用方哪边的程序没有设置呢？

调用方使用的是 Apache HttpClient ，tcpNoDelay 默认设置的就是 true。我们再来看看被调用方，也就是我们的后端 HTTP 服务，这个 HTTP 服务用的是 JDK自带的 HttpServer
``` java
HttpServer server = HttpServer.create(new InetSocketAddress(config.getPort()), BACKLOGS);
```
居然没看到直接设置 tcpNoDelay 接口，翻了下源码。哦，原来在这里，在 ServerConfig 的类中有这段静态块，用来获取启动参数，默认 ServerConfig.noDelay 为 false。
```java
static {
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {
            ServerConfig.idleInterval = Long.getLong("sun.net.httpserver.idleInterval", 30L) * 1000L;
            ServerConfig.clockTick = Integer.getInteger("sun.net.httpserver.clockTick", 10000);
            ServerConfig.maxIdleConnections = Integer.getInteger("sun.net.httpserver.maxIdleConnections", 200);
            ServerConfig.drainAmount = Long.getLong("sun.net.httpserver.drainAmount", 65536L);
            ServerConfig.maxReqHeaders = Integer.getInteger("sun.net.httpserver.maxReqHeaders", 200);
            ServerConfig.maxReqTime = Long.getLong("sun.net.httpserver.maxReqTime", -1L);
            ServerConfig.maxRspTime = Long.getLong("sun.net.httpserver.maxRspTime", -1L);
            ServerConfig.timerMillis = Long.getLong("sun.net.httpserver.timerMillis", 1000L);
            ServerConfig.debug = Boolean.getBoolean("sun.net.httpserver.debug");
   	        ServerConfig.noDelay = Boolean.getBoolean("sun.net.httpserver.nodelay");
            return null;
        }
    });
}
```

### 4.2 验证

在后端 HTTP 服务，加上启动"-Dsun.net.httpserver.nodelay=true"参数，再试试。效果很明显，平均耗时从**39.2ms** 降到 **2.8ms**。
![优化后调用时延](/img/2019/07/httpoptimized/invoke-delay-optimized.png)


问题是解决了，但是到这里如果你就此止步，那就太便宜了这个案例了，简直暴殄天物。因为还有一堆疑惑等着你呢？

*   为什么加了 TCP_NODELAY ，时延就从 **39.2ms** 降低到 **2.8ms**？
*   为什么本地测试的平均时延是 **55ms**，而不是 ping 的时延 **26ms**？
*   TCP 协议究竟是怎么发送数据包的？

来，我们接着乘热打铁。

## 5. 解惑

### 5.1 TCP_NODELAY 何许人也？

在 Socket 编程中，TCP_NODELAY 选项是用来控制是否开启 Nagle 算法。在 Java 中，为 ture 表示关闭 Nagle 算法，为 false 表示打开 Nagle 算法。你一定会问 Nagle 算法是什么？

### 5.2 Nagle 算法是什么鬼？

Nagle 算法是一种通过减少通过网络发送的数据包数量来提高 TCP/IP 网络效率的方法。它使用发明人 John Nagle 的名字来命名的，John Nagle 在 1984 年首次用这个算法来尝试解决福特汽车公司的网络拥塞问题。

试想如果应用程序每次产生 1 个字节的数据，然后这 1 个字节数据又以网络数据包的形式发送到远端服务器，那么就很容易导致网络由于太多的数据包而过载。在这种典型情况下，传送一个只拥有1个字节有效数据的数据包，却要花费 40 个字节长包头（即 IP 头部 20 字节 + TCP 头部 20 字节）的额外开销，这种有效载荷（payload）的利用率是极其低下。

Nagle 算法的内容比较简单，以下是伪代码：
```
if there is new data to send
  if the window size >= MSS and available data is >= MSS
    send complete MSS segment now
  else
    if there is unconfirmed data still in the pipe
      enqueue data in the buffer until an acknowledge is received
    else
      send data immediately
    end if
  end if
end if
```
具体的做法就是：

*   如果发送内容大于等于 1 个 MSS， 立即发送；
*   如果之前没有包未被 ACK， 立即发送；
*   如果之前有包未被 ACK， 缓存发送内容；
*   如果收到 ACK， 立即发送缓存的内容。
（MSS 为 TCP 数据包每次能够传输的最大数据分段）

### 5.3 Delayed ACK 又是什么玩意？

大家都知道 TCP 协议为了保证传输的可靠性，规定在接受到数据包时需要向对方发送一个确认。只是单纯的发送一个确认，代价会比较高（IP 头部 20 字节 + TCP 头部 20 字节）。**TCP Delayed ACK（延迟确认）就是为了努力改善网络性能，来解决这个问题的，它将几个 ACK 响应组合合在一起成为单个响应，或者将 ACK 响应与响应数据一起发送给对方，从而减少协议开销**。

具体的做法是：

*   当有响应数据要发送时，ACK 会随响应数据立即发送给对方；
*   如果没有响应数据，ACK 将会延迟发送，以等待看是否有响应数据可以一起发送。在 Linux 系统中，默认这个延迟时间是 40ms；
*   如果在等待发送 ACK 期间，对方的第二个数据包又到达了，这时要立即发送 ACK。但是如果对方的三个数据包相继到达，第三个数据段到达时是否立即发送 ACK，则取决于以上两条。

### 5.4 Nagle 与 Delayed ACK 一起会发生什么化学反应？

Nagle 与 Delayed ACK 都能提高网络传输的效率，但在一起会好心办坏事。例如，以下这个场景：

A 和 B 进行数据传输 : A 运行 Nagle 算法，B 运行 Delayed ACK 算法。

如果 A 向 B 发一个数据包，B 由于 Delayed ACK 不会立即响应。而 A 使用 Nagle 算法，A 就会一直等 B 的 ACK，ACK 不来一直不发送第二个数据包，如果这两个数据包是应对同一个请求，那这个请求就会被耽误了 **40ms**。

### 5.5 抓个包玩玩吧

我们来抓个包验证下吧，在后端HTTP服务上执行以下脚本，就可以轻松完成抓包过程。
```
sudo tcpdump -i eth0 tcp and host 10.48.159.165 -s 0 -w traffic.pcap
```
如下图，这是使用 Wireshark 分析包内容的展示，红框内是一个完整的 POST 请求处理过程，看 130 序号和 149 序号之间相差 40ms（0.1859 - 0.1448 = 0.0411s = 41ms），这个就是 Nagle 与 Delayed ACK 一起发送的化学反应，其中 10.48.159.165 运行的是 Delayed ACK，10.22.29.180 运行的是 Nagle 算法。10.22.29.180 在等 ACK，而 10.48.159.165 触发了 Delayed ACK，这样傻傻等了 40ms。

![测试环境数据包分析](/img/2019/07/httpoptimized/test-traffic.png)

**这也就解释了为什么测试环境耗时是 39.2ms，因为大部分都被 Delayed ACK 的 40ms 给耽误了。**

但是本地复现时，为什么本地测试的平均时延是 **55ms**，而不是 ping 的时延 **26ms**？我们也来抓个包吧。

如下图，红框内是一个完整的 POST 请求处理过程，看 8 序号和 9 序号之间相差 25ms 左右，再减去网络延时约是ping延时的一半 13ms，因此 Delayed Ack 约 12ms 左右（由于本地是 MAC 系统与 Linux 有些差异）。

![本地环境数据包分析](/img/2019/07/httpoptimized/local-traffic.png)


```
1. Linux 使用的是 /proc/sys/net/ipv4/tcp_delack_min 这个系统配置来控制 Delayed ACK 的时间，Linux 默认是 40ms；
2. MAC 是通过 net.inet.tcp.delayed_ack 系统配置来控制 Delayed ACK 的。
  delayed_ack=0 responds after every packet (OFF)
  delayed_ack=1 always employs delayed ack, 6 packets can get 1 ack 
  delayed_ack=2 immediate ack after 2nd packet, 2 packets per ack  (Compatibility Mode)
  delayed_ack=3 should auto detect when to employ delayed ack, 4packets per ack.  (DEFAULT)
设置为 0 表示禁止延迟 ACK，设置为 1 表示总是延迟 ACK，设置为 2 表示每两个数据包回复一个 ACK，设置为 3 表示系统自动探测回复 ACK 的时机。
```

### 5.6 为什么 TCP_NODELAY 能够解决问题？

TCP_NODELAY 关闭了 Nagle 算法，即使上个数据包的 ACK 没有到达，也会发送下个数据包，进而打破 Delayed ACK 造成的影响。一般在网络编程中，非常建议开启 TCP_NODELAY，来提升响应速度。

当然也可以通过 Delayed ACK 相关系统的配置来解决问题，但由于需要修改机器配置，很不方便，因此，这种方式不太推荐。

## 6. 总结

本文是从一个简单的 HTTP 调用，时延比较大而引发的一次问题排查过程。过程中，首先由外而内的分析了相关问题，然后定位问题并验证解决方案。最后刨根问底对 TCP 传输的中的 Nagle 与 Delayed ACK 做了全面的讲解，更加透测的剖析了该问题案例。





******
> 涤生的博客。

> 转载请注明原创出处，谢谢！

> 欢迎关注我的微信公众号：「涤生的博客」，获取更多技术分享。

![涤生-微信公共号](/img/main/officialAccount.jpg)

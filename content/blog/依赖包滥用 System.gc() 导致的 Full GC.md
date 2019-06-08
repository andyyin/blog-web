+++
author = "涤生"
categories = ["Java","GC","JVM"]
tags = ["Java", "JVM", "GC", "System.gc()","Full GC"]
date = "2017-10-11"
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "依赖包滥用 System.gc() 导致的 Full GC"
type = "post"
+++

# 介绍
业务部门的一个同事遇到个奇怪的 Full GC 问题，有个服务迁移到新的应用后，一直频繁 Full GC。新应用机器的配置是 4c 8g，老应用是 4c 4g，老应用 GC 都很正常，并且代码没有变更，所以比较奇怪。

# 现象
问题的现象是，从监控图上看一直有大量的 Full GC
![GC 监控图](/img/2017/10/cmsgc/gcinfo.png)

# 排查
遇到这个问题，一般都是先看看各个区的内存占用情况：
![Old/Young Gen 监控图](/img/2017/10/cmsgc/oldgenpercent.png)
![Perm Gen 监控图](/img/2017/10/cmsgc/permgenpercent.png)

从监控图上看 Old Gen、Young Gen、Perm Gen，没什么问题，不会触发 Full GC，当然这里看各个 Gen 是否会触发 Full GC 需要结合 JVM 参数配置来看。

顺便也看了下 GC 日志，一直狂暴 CMS GC 日志，而且可以看到老年代使用空间也不大，细心可以发现，大量的 CMS GC 中夹杂着 Young、Perm Gen 的回收，所以其实是 Full GC。GC 日志如下：
![GC 日志](/img/2017/10/cmsgc/gclog.png)

老应用的 JVM 参数配置
![老应用的 JVM 参数配置](/img/2017/10/cmsgc/oldapp-jvmparam.png)

新应用的 JVM 参数配置
![新应用的 JVM 参数配置](/img/2017/10/cmsgc/newapp-jvmparam.png)

通过上面的观察，再根据一般触发 CMS GC 几个可能性：

* Old Gen 使用达到一定的比率，默认为 92%，这里看 CMSInitiatingOccupancyFraction=80%，而实际才使用 2%（看监控图表）不到，所以排除这种情况。
* 配置了 CMSClassUnloadingEnabled，且 Perm Gen 的使用达到一定的比率默认为 92%，这里看 CMSInitiatingPermOccupancyFraction=80%，而实际才使用 30％（看监控图表）不到，所以排除这种情况。
* 配置了 ExplictGCInvokesConcurrent 且未配置 DisableExplicitGC 的情况下显示调用了 System.gc()。
* Hotspot 自己根据估计决定是否要触法，如 CMS 悲观策略，这类可以通过 GC 日志分析。

大致判断很可能是 System.gc() 导致的问题，但是怎么定位调用 System.gc() 的代码呢？
当时就想如果是 System.gc() 引起的频繁 Full GC，jstack 线程堆栈应该能看到一些信息，果不其然，确实通过线程堆栈找到了。
![jstack 线程堆栈](/img/2017/10/cmsgc/jstackinfo.png)

jstack 作用非常大，很多问题都能从这里发现，而且比较轻量，对应用基本无影响。某次的 jstack 信息只代表那个时刻的线程堆栈，有时只看一个 jstack 信息可能看不出什么问题，一般可以多 jstack 几次，然后对比去看，基本就能发现一些问题。
（当然该问题，也可能不是频繁的 Full GC，可能通过 jstack 定位不到问题，可以 jstat -gccause pid 1000，来查看 GC 原因。）

很明显，是由于 jxl 这个包中的 close 方法显示调用了 System.gc() 导致的问题。

跟了下代码，自然确实存在这段代码，不过有个设置开关，可以 disable 这个功能，所以在使用的时候可以设置 setGCDisabled(true)，关闭触发 System.gc()。
![触发 System.gc() 代码](/img/2017/10/cmsgc/jxl-systemgc.png)

但是为什么老应用没有问题呢，主要是因为它 -XX:+DisableExplicitGC，屏蔽了 System.gc() 动作，新应用的JVM没有这个配置。

可能大家还有个疑问，都知道 System.gc() 会触发 Full GC，那为什么一直进行 CMS GC（通过 GC 日志）呢？
主要是因为这个参数 -XX:+ExplicitGCInvokesConcurrent，打开此参数后，会做并行 Full GC，只有配置 -XX:+UseConcMarkSweepGC 这个参数，该参数才会生效。因此，System.gc() 时 Old Gen 会进行 CMS GC，可提高 Full GC 效率。

# 总结
尽量减少显示使用 System.gc() 来触发 Full GC，这会导致频繁 Full GC，非常影响应用性能。


******
> 涤生的博客。

> 转载请注明原创出处，谢谢！

> 欢迎关注我的微信公众号：「涤生的博客」，获取更多技术分享。

![涤生-微信公共号](/img/main/officialAccount.jpg)
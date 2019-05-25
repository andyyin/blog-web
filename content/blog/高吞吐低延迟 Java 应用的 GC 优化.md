
+++
author = "涤生"
categories = ["Java", "GC"]
tags = ["高可用", "低延时", "Java", "GC", "CMS"]
date = "2019-04-21"
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "[译]高吞吐低延迟 Java 应用的 GC 优化"
type = "post"

+++

## 说明
本篇原文作者是 LinkedIn 的 Swapnil Ghike，这篇文章讲述了 LinkedIn 的 Feed 产品的 GC 优化过程，虽然文章写作于 April 8, 2014，但其中的很多内容和知识点非常有参考意义。因此，翻译后献给各位同学。
原文链接：[Garbage Collection Optimization for High-Throughput and Low-Latency Java Applications](https://engineering.linkedin.com/garbage-collection/garbage-collection-optimization-high-throughput-and-low-latency-java-applications)。

## 背景
高性能应用构成了现代网络的支柱。LinkedIn 内部有许多高吞吐量服务来满足每秒成千上万的用户请求。为了获得最佳的用户体验，以低延迟响应这些请求是非常重要的。

例如，我们的用户经常使用的产品是 Feed —— 它是一个不断更新的专业活动和内容的列表。Feed 在 LinkedIn 的系统中随处可见，包括公司页面、学校页面以及最重要的主页资讯信息。基础 Feed 数据平台为我们的经济图谱（会员、公司、群组等）中各种实体的更新建立索引，它必须高吞吐低延迟地实现相关的更新。
![LinkedIn Feeds](https://upload-images.jianshu.io/upload_images/3017145-932aa0be5ce180d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
为了将这些高吞吐量、低延迟类型的 Java 应用程序用于生产，开发人员必须确保在应用程序开发周期的每个阶段都保持一致的性能。确定最佳垃圾收集（Garbage Collection, GC）配置对于实现这些指标至关重要。

这篇博文将通过一系列步骤来明确需求并优化 GC，它的目标读者是对使用系统方法进行 GC 优化来实现应用的高吞吐低延迟目标感兴趣的开发人员。在 LinkedIn 构建下一代 Feed 数据平台的过程中，我们总结了该方法。这些方法包括但不限于以下几点：[并发标记清除（Concurrent Mark Sweep，CMS）](http://www.oracle.com/technetwork/java/javase/memorymanagement-whitepaper-150215.pdf) 和 [G1](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html) 垃圾回收器的 CPU 和内存开销、避免长期存活对象导致的持续 GC、优化 GC 线程任务分配提升性能，以及可预测 GC 停顿时间所需的 OS 配置。

## 优化 GC 的正确时机？
GC 的行为可能会因代码优化以及工作负载的变化而变化。因此，在一个已实施性能优化的接近完成的代码库上进行 GC 优化非常重要。而且在端到端的基本原型上进行初步分析也很有必要，该原型系统使用存根代码并模拟了可代表生产环境的工作负载。这样可以获取该架构延迟和吞吐量的真实边界，进而决定是否进行纵向或横向扩展。

在下一代 Feed 数据平台的原型开发阶段，我们几乎实现了所有端到端的功能，并且模拟了当前生产基础设施提供的查询工作负载。这使我们在工作负载特性上有足够的多样性，可以在足够长的时间内测量应用程序性能和 GC 特征。

## 优化 GC 的步骤
下面是一些针对高吞吐量、低延迟需求优化 GC 的总体步骤。此外，还包括在 Feed 数据平台原型实施的具体细节。尽管我们还对 G1 垃圾收集器进行了试验，但我们发现 ParNew/CMS 具有最佳的 GC 性能。

### 1. 理解 GC 基础知识
由于 GC 优化需要调整大量的参数，因此理解 GC 工作机制非常重要。Oracle 的 Hotspot JVM 内存管理[白皮书](https://www.oracle.com/technetwork/java/javase/memorymanagement-whitepaper-150215.pdf)是开始学习 Hotspot JVM GC 算法非常好的资料。而了解 G1 垃圾回收器的理论知识，可以参阅该[文章](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html)。

### 2. 仔细考量 GC 需求
为了降低对应用程序性能的开销，可以优化 GC 的一些特征。像吞吐量和延迟一样，这些 GC 特征应该在长时间运行的测试中观察到，以确保应用程序能够在经历多个 GC 周期中处理流量的变化。
* Stop-the-world 回收器回收垃圾时会暂停应用线程。停顿的时长和频率不应该对应用遵守 SLA 产生不利的影响。
* 并发 GC 算法与应用线程竞争 CPU 周期。这个开销不应该影响应用吞吐量。
* 非压缩 GC 算法会引起堆碎片化，进而导致的 Full GC 长时间 Stop-the-world，因此，堆碎片应保持在最小值。
* 垃圾回收工作需要占用内存。某些 GC 算法具有比其他算法更高的内存占用。如果应用程序需要较大的堆空间，要确保 GC 的内存开销不能太大。
* 要清楚地了解 GC 日志和常用的 JVM 参数，以便轻松地调整 GC 行为。因为 GC 运行随着代码复杂性增加或工作负载特性的改变而发生变化

我们使用 Linux 操作系统、Hotspot [Java7u51](https://www.oracle.com/technetwork/java/javase/7u51-relnotes-2085002.html)、32GB 堆内存、6GB 新生代（Young Gen）和 -XX:CMSInitiatingOccupancyFraction 值为 70（Old GC 触发时其空间占用率）开始实验。设置较大的堆内存是用来维持长期存活对象的对象缓存。一旦这个缓存生效，晋升到 Old Gen 的对象速度会显著下降。

使用最初的 JVM 配置，每 3 秒发生一次 80ms 的 Young GC 停顿，超过 99.9% 的应用请求延迟 100ms（999线）。这样的 GC 效果可能适合于 SLA 对延迟要求不太严格应用。然而，我们的目标是尽可能减少应用请求的 999 线。GC 优化对于实现这一目标至关重要。

### 3. 理解 GC 指标

衡量应用当前情况始终是优化的先决条件。了解 [GC 日志的详细细节](https://engineering.linkedin.com/26/tuning-java-garbage-collection-web-services)（使用以下选项）：
```
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps 
-XX:+PrintTenuringDistribution -XX:+PrintGCApplicationStoppedTime
```
可以对该应用的 GC 特征有总体的把握。

在 LinkedIn 的内部监控 [inGraphs](https://engineering.linkedin.com/32/eric-intern-origin-ingraphs) 和报表系统 [Naarad](https://github.com/linkedin/naarad)，生成了各种有用的指标可视化图形，比如 GC 停顿时间百分比、一次停顿最大持续时间以及长时间内 GC 频率。除了 Naarad，有很多开源工具比如 [gclogviewer](https://code.google.com/p/gclogviewer/) 可以从 GC 日志创建可视化图形。
在此阶段，可以确定 GC 频率和暂停持续时间是否满足应用程序满足延迟的要求。

### 4. 降低 GC 频率
在分代 GC 算法中，降低 GC 频率可以通过：(1)降低对象分配/晋升率；(2)增加各代空间的大小。

在 Hotspot JVM 中，Young GC 停顿时间取决于一次垃圾回收后存活下来的对象的数量，而不是 Young Gen 自身的大小。增加 Young Gen 大小对于应用性能的影响需要仔细评估：
* 如果更多的数据存活而且被复制到 Survivor 区域，或者每次 GC 更多的数据晋升到 Old Gen，增加 Young Gen 大小可能导致更长的 Young GC 停顿。较长的 GC 停顿可能会导致应用程序延迟增加和(或)吞吐量降低。
* 另一方面，如果每次垃圾回收后存活对象数量不会大幅增加，停顿时间可能不会延长。在这种情况下，降低 GC 频率可能会使整个应用总体延迟降低和(或)吞吐量增加。

对于大部分为短期存活对象的应用，仅仅需要控制上述的参数；对于长期存活对象的应用，就需要注意，被晋升的对象可能很长时间都不能被 Old GC 周期回收。如果 Old GC 触发阈值（Old Gen 占用率百分比）比较低，应用将陷入持续的 GC 循环中。可以通过设置高的 GC 触发阈值可避免这一问题。

由于我们的应用在堆中维持了长期存活对象的较大缓存，将 Old GC 触发阈值设置为 
```
-XX:CMSInitiatingOccupancyFraction=92 -XX:+UseCMSInitiatingOccupancyOnly
```
来增加触发 Old GC 的阈值。我们也试图增加 Young Gen 大小来减少 Young GC 频率，但是并没有采用，因为这增加了应用的 999 线。

### 5. 缩短 GC 停顿时间
减少 Young Gen 大小可以缩短 Young GC 停顿时间，因为这可能导致被复制到 Survivor 区域或者被晋升的数据更少。但是，正如前面提到的，我们要观察减少 Young Gen 大小和由此导致的 GC 频率增加对于整体应用吞吐量和延迟的影响。Young GC 停顿时间也依赖于 tenuring threshold （晋升阈值）和 Old Gen 大小（如步骤 6 所示）。

在使用 CMS GC 时，应将因堆碎片或者由堆碎片导致的 Full GC 的停顿时间降低到最小。通过控制对象晋升比例和减小 -XX:CMSInitiatingOccupancyFraction 的值使 Old GC 在低阈值时触发。所有选项的细节调整和他们相关的权衡，请参考 [Web Services 的 Java 垃圾回收](https://engineering.linkedin.com/26/tuning-java-garbage-collection-web-services) 和 [Java 垃圾回收精粹](http://mechanical-sympathy.blogspot.com/2013/07/java-garbage-collection-distilled.html)。

我们观察到 Eden 区域的大部分 Young Gen 被回收，几乎没有 3-8 年龄对象在 Survivor 空间中死亡，所以我们将 tenuring threshold 从 8 降低到 2 (使用选项：-XX:MaxTenuringThreshold=2 ),以降低 Young GC 消耗在数据复制上的时间。

我们还注意到 Young GC 暂停时间随着 Old Gen 占用率上升而延长。这意味着来自 Old Gen 的压力使得对象晋升花费更多的时间。为解决这个问题，将总的堆内存大小增加到 40GB，减小 -XX:CMSInitiatingOccupancyFraction 的值到 80，更快地开始 Old GC。尽管 -XX:CMSInitiatingOccupancyFraction 的值减小了，增大堆内存可以避免频繁的 Old GC。在此阶段，我们的结果是  Young GC 暂停 70ms，应用的 999 线在 80ms。

### 6. 优化 GC 工作线程的任务分配
为了进一步降低 Young GC 停顿时间，我们决定研究 GC 线程绑定任务的参数来进行优化。

-XX:ParGCCardsPerStrideChunk 参数控制 GC 工作线程的任务粒度，可以帮助不使用[补丁](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=7068625)而获得最佳性能，这个补丁用来优化 Young GC 中的 [Card table（卡表扫描时间）](http://blog.ragozin.info/2012/03/secret-hotspot-option-improving-gc.html)。有趣的是，Young GC 时间随着 Old Gen 的增加而延长。将这个选项值设为 32678，Young GC 停顿时间降低到平均 50ms。此时应用的 999 线在 60ms。

还有一些的参数可以将任务映射到 GC 线程，如果操作系统允许的话，-XX:+BindGCTaskThreadsToCPUs 参数可以绑定 GC 线程到个别的 CPU 核。使用亲缘性 -XX:+UseGCTaskAffinity 参数可以将任务分配给 GC 工作线程。然而，我们的应用并没有从这些选项带来任何好处。实际上，一些调查显示这些选项在 Linux 系统不起作用[1,2]。

### 7. 了解 GC 的 CPU 和内存开销
并发 GC 通常会增加 CPU 使用率。虽然我们观察到 CMS 的默认设置运行良好，但是 G1 收集器的并发 GC 工作会导致 CPU 使用率的增加，显著降低了应用程序的吞吐量和延迟。与 CMS 相比，G1 还增加了内存开销。对于不受 CPU 限制的低吞吐量应用程序，GC 导致的高 CPU 使用率可能不是一个紧迫的问题。

![ParNew/CMS 和 G1 的 CPU 使用百分比：相对来说 CPU 使用率变化明显的节点使用 G1
选项 -XX:G1RSetUpdatingPauseTimePercent=20](https://upload-images.jianshu.io/upload_images/3017145-a96b1149a52ea550.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![ParNew/CMS 和 G1 每秒服务的请求数：吞吐量较低的节点使用 G1
选项 -XX:G1RSetUpdatingPauseTimePercent=20](https://upload-images.jianshu.io/upload_images/3017145-2098d0a539ce9d0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 8. 为 GC 优化系统内存和 I/O 管理
通常来说，GC 停顿有两种特殊情况：
(1)低 user time，高 sys time 和高 real time
(2)低 user time，低 sys time 和高 real time。
这意味着基础的进程/OS设置存在问题。
情况 (1) 可能意味着 JVM 页面被 Linux 窃取；
情况 (2) 可能意味着 GC 线程被 Linux 用于磁盘刷新，并卡在内核中等待 I/O。
在这些情况下，如何设置参数可以参考该[PPT](http://www.slideshare.net/cuonghuutran/gc-andpagescanattacksbylinux)。

另外，为了避免在运行时造成性能损失，我们可以使用 JVM 选项 -XX:+AlwaysPreTouch 在应用程序启动时先访问所有分配给它的内存，让操作系统把内存真正的分配给 JVM。我们还可以将 vm.swappability 设置为0，这样操作系统就不会交换页面到 swap（除非绝对必要）。

可能你会使用 mlock 将 JVM 页固定到内存中，这样操作系统就不会将它们交换出去。但是，如果系统用尽了所有的内存和交换空间，操作系统将终止一个进程来回收内存。通常情况下，Linux 内核会选择具有高驻留内存占用但运行时间不长的进程 ([OOM 情况下杀死进程的工作流](https://www.kernel.org/doc/gorman/html/understand/understand016.html))。
在我们的例子中，这个进程很有可能就是我们的应用程序。优雅的降级是服务优秀的属性之一，不过服务突然终止的可能性对于可操作性来说并不好 —— 因此，我们不使用 mlock，只是通过 vm.swapability 来尽可能避免交换内存页到 swap 的惩罚。

## LinkedIn Feed 数据平台的 GC 优化
对于该 Feed 平台原型系统，我们使用 Hotspot JVM 的两个 GC 算法优化垃圾回收：

* Young GC 使用 ParNew，Old GC 使用 CMS。
* Young Gen 和 Old Gen 使用 G1。G1 试图解决堆大小为 6GB 或更大时，暂停时间稳定且可预测在 0.5 秒以下的问题。在我们用 G1 实验过程中，尽管调整了各种参数，但没有得到像 ParNew/CMS 一样的 GC 性能或停顿时间的可预测值。我们查询了使用 G1 发生内存泄漏相关的一个 bug[3]，但还不能确定根本原因。

使用 ParNew/CMS，应用每三秒进行一次 40-60ms 的 Young GC 和每小时一个 CMS GC。JVM 参数如下：
```
// JVM sizing options
-server -Xms40g -Xmx40g -XX:MaxDirectMemorySize=4096m -XX:PermSize=256m -XX:MaxPermSize=256m   
// Young generation options
-XX:NewSize=6g -XX:MaxNewSize=6g -XX:+UseParNewGC -XX:MaxTenuringThreshold=2 -XX:SurvivorRatio=8 -XX:+UnlockDiagnosticVMOptions -XX:ParGCCardsPerStrideChunk=32768
// Old generation  options
-XX:+UseConcMarkSweepGC -XX:CMSParallelRemarkEnabled -XX:+ParallelRefProcEnabled -XX:+CMSClassUnloadingEnabled  -XX:CMSInitiatingOccupancyFraction=80 -XX:+UseCMSInitiatingOccupancyOnly   
// Other options
-XX:+AlwaysPreTouch -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -XX:+PrintGCApplicationStoppedTime -XX:-OmitStackTraceInFastThrow
```
使用这些参数，对于成千上万读请求的吞吐量，我们应用程序的 999 线降低到 60ms。

## 感谢
参与了原型应用程序开发的同学有：Ankit Gupta、Elizabeth Bennett、Raghu Hiremagalur、Roshan Sumbaly、Swapnil Ghike、Tom Chiang 和 Vivek Nelamangala。
另外，感谢 Cuong Tran、David Hoa 和 Steven Ihde 在系统优化方面的帮助。

## 参考
[1] -XX:+BindGCTaskThreadsToCPUs 参数似乎在Linux 系统上不起作用，因为 hotspot/src/os/linux/vm/os_linux.cpp 的 distribute_processes 方法在 JDK7 或 JDK8 中没有实现。
[2] -XX:+UseGCTaskAffinity 参数在 JDK7 和 JDK8 的所有平台似乎都不起作用，因为任务的亲缘性属性永远被设置为 sentinel_worker = (uint) -1。源码见 hotspot/src/share/vm/gc_implementation/parallelScavenge/{gcTaskManager.cpp，gcTaskThread.cpp, gcTaskManager.cpp}。
[3] G1 存在一些内存泄露的 bug，可能 Java7u51 没有修改。这个 [bug](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=2212435) 仅在 Java 8 修正了。

******
欢迎关注我的微信公众号：「涤生的博客」，获取更多技术分享。

![涤生-微信公共号](https://upload-images.jianshu.io/upload_images/3017145-277e5f278eb59205.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

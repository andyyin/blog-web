+++
author = "涤生"
categories = ["Java","GC","JVM"]
tags = ["Java", "JVM", "GC", "CMS GC", "backgroud collector","foreground collector"]
date = "2019-06-07"
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "JVM 源码解读之 CMS GC 触发条件"
type = "post"
+++

# 前言
经常有同学会问，为啥我的应用 Old Gen 的使用占比没达到 CMSInitiatingOccupancyFraction 参数配置的阈值，就触发了 CMS GC，表示很莫名奇妙，不知道问题出在哪？

其实 CMS GC 的触发条件非常多，不只是 CMSInitiatingOccupancyFraction 阈值触发这么简单。本文通过源码全面梳理了触发 CMS GC 的条件，尽可能的帮你了解平时遇到的奇奇怪怪的 CMS GC 问题。

先抛出一些问题，来吸引你的注意力。

> 为什么 Old Gen 使用占比仅 50% 就进行了一次 CMS GC？

> Metaspace 的使用也会触发 CMS GC 吗？

> 为什么 Old Gen 使用占比非常小就进行了一次 CMS GC？


# 触发条件
CMS GC 在实现上分成 foreground collector 和 background collector。foreground collector 相对比较简单，background collector 比较复杂，情况比较多。

下面我们从 foreground collector 和 background collector 分别来说明他们的触发条件：

 > 说明：本文内容是基于 JDK 8

 > 说明：本文仅涉及 CMS GC 的触发条件，至于算法的具体过程，以及什么时候进行 MSC（mark sweep compact）不在本文范围
 
## foreground collector
foreground collector 触发条件比较简单，一般是遇到对象分配但空间不够，就会直接触发 GC，来立即进行空间回收。采用的算法是 mark sweep，不压缩。


## background collector
说明 background collector 的触发条件之前，先来说下 background collector 的流程，它是通过 CMS 后台线程不断的去扫描，过程中主要是判断是否符合 background collector 的触发条件，一旦有符合的情况，就会进行一次 background 的 collect。

```
void ConcurrentMarkSweepThread::run() {
  ...//省略
  while (!_should_terminate) {
    sleepBeforeNextCycle();
    if (_should_terminate) break;
    GCCause::Cause cause = _collector->_full_gc_requested ?
      _collector->_full_gc_cause : GCCause::_cms_concurrent_mark;
    _collector->collect_in_background(false, cause);
  }
  ...//省略
}
```
每次扫描过程中，先等 CMSWaitDuration 时间，然后再去进行一次 shouldConcurrentCollect 判断，看是否满足 CMS background collector 的触发条件。CMSWaitDuration 默认时间是 2s（经常会有业务遇到频繁的 CMS GC，注意看每次 CMS GC 之间的时间间隔，如果是 2s，那基本就可以断定是 CMS 的 background collector）。
```
void ConcurrentMarkSweepThread::sleepBeforeNextCycle() {
  while (!_should_terminate) {
    if (CMSIncrementalMode) {
      icms_wait();
      if(CMSWaitDuration >= 0) {
        // Wait until the next synchronous GC, a concurrent full gc
        // request or a timeout, whichever is earlier.
        wait_on_cms_lock_for_scavenge(CMSWaitDuration);
      }
      return;
    } else {
      if(CMSWaitDuration >= 0) {
        // Wait until the next synchronous GC, a concurrent full gc
        // request or a timeout, whichever is earlier.
        wait_on_cms_lock_for_scavenge(CMSWaitDuration);
      } else {
        // Wait until any cms_lock event or check interval not to call shouldConcurrentCollect permanently
        wait_on_cms_lock(CMSCheckInterval);
      }
    }
    // Check if we should start a CMS collection cycle
    if (_collector->shouldConcurrentCollect()) {
      return;
    }
    // .. collection criterion not yet met, let's go back
    // and wait some more
  }
}
```
那 shouldConcurrentCollect() 方法中都有哪些条件呢？
```
bool CMSCollector::shouldConcurrentCollect() {

  // 第一种触发情况
  if (_full_gc_requested) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print_cr("CMSCollector: collect because of explicit "
                             " gc request (or gc_locker)");
    }
    return true;
  }
  
  // For debugging purposes, change the type of collection.
  // If the rotation is not on the concurrent collection
  // type, don't start a concurrent collection.
  NOT_PRODUCT(
    if (RotateCMSCollectionTypes &&
        (_cmsGen->debug_collection_type() !=
          ConcurrentMarkSweepGeneration::Concurrent_collection_type)) {
      assert(_cmsGen->debug_collection_type() !=
        ConcurrentMarkSweepGeneration::Unknown_collection_type,
        "Bad cms collection type");
      return false;
    }
  )
  FreelistLocker x(this);
  // ------------------------------------------------------------------
  // Print out lots of information which affects the initiation of
  // a collection.
  if (PrintCMSInitiationStatistics && stats().valid()) {
    gclog_or_tty->print("CMSCollector shouldConcurrentCollect: ");
    gclog_or_tty->stamp();
    gclog_or_tty->print_cr("");
    stats().print_on(gclog_or_tty);
    gclog_or_tty->print_cr("time_until_cms_gen_full %3.7f",
      stats().time_until_cms_gen_full());
    gclog_or_tty->print_cr("free="SIZE_FORMAT, _cmsGen->free());
    gclog_or_tty->print_cr("contiguous_available="SIZE_FORMAT,
                           _cmsGen->contiguous_available());
    gclog_or_tty->print_cr("promotion_rate=%g", stats().promotion_rate());
    gclog_or_tty->print_cr("cms_allocation_rate=%g", stats().cms_allocation_rate());
    gclog_or_tty->print_cr("occupancy=%3.7f", _cmsGen->occupancy());
    gclog_or_tty->print_cr("initiatingOccupancy=%3.7f", _cmsGen->initiating_occupancy());
    gclog_or_tty->print_cr("metadata initialized %d",
      MetaspaceGC::should_concurrent_collect());
  }
  // ------------------------------------------------------------------
  
  // 第二种触发情况
  // If the estimated time to complete a cms collection (cms_duration())
  // is less than the estimated time remaining until the cms generation
  // is full, start a collection.
  if (!UseCMSInitiatingOccupancyOnly) {
    if (stats().valid()) {
      if (stats().time_until_cms_start() == 0.0) {
        return true;
      }
    } else {
      // We want to conservatively collect somewhat early in order
      // to try and "bootstrap" our CMS/promotion statistics;
      // this branch will not fire after the first successful CMS
      // collection because the stats should then be valid.
      if (_cmsGen->occupancy() >= _bootstrap_occupancy) {
        if (Verbose && PrintGCDetails) {
          gclog_or_tty->print_cr(
            " CMSCollector: collect for bootstrapping statistics:"
            " occupancy = %f, boot occupancy = %f", _cmsGen->occupancy(),
            _bootstrap_occupancy);
        }
        return true;
      }
    }
  }
  
  // 第三种触发情况
  // Otherwise, we start a collection cycle if
  // old gen want a collection cycle started. Each may use
  // an appropriate criterion for making this decision.
  // XXX We need to make sure that the gen expansion
  // criterion dovetails well with this. XXX NEED TO FIX THIS
  if (_cmsGen->should_concurrent_collect()) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print_cr("CMS old gen initiated");
    }
    return true;
  }

  // 第四种触发情况
  // We start a collection if we believe an incremental collection may fail;
  // this is not likely to be productive in practice because it's probably too
  // late anyway.
  GenCollectedHeap* gch = GenCollectedHeap::heap();
  assert(gch->collector_policy()->is_two_generation_policy(),
         "You may want to check the correctness of the following");
  if (gch->incremental_collection_will_fail(true /* consult_young */)) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print("CMSCollector: collect because incremental collection will fail ");
    }
    return true;
  }
  
  // 第五种触发情况
  if (MetaspaceGC::should_concurrent_collect()) {
      if (Verbose && PrintGCDetails) {
      gclog_or_tty->print("CMSCollector: collect for metadata allocation ");
      }
      return true;
    }

  return false;
}
```
上述代码可知，从大类上分， background collector 一共有 5 种触发情况：

* 是否是并行 Full GC

```
// 第一种触发情况
  if (_full_gc_requested) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print_cr("CMSCollector: collect because of explicit "
                             " gc request (or gc_locker)");
    }
    return true;
  }
```
指的是在 GC cause 是 _gc_locker 且配置了 GCLockerInvokesConcurrent 参数，或者 GC cause 是_java_lang_system_gc（就是 System.gc()调用）and 且配置了 ExplicitGCInvokesConcurrent 参数，这时会触发一次 background collector。

* 根据统计数据动态计算（仅未配置 UseCMSInitiatingOccupancyOnly 时）


未配置 UseCMSInitiatingOccupancyOnly 时，会根据统计数据动态判断是否需要进行一次 CMS GC。

判断逻辑是，如果预测 CMS GC 完成所需要的时间大于预计的老年代将要填满的时间，则进行 GC。
这些判断是需要基于历史的 CMS GC 统计指标，然而，第一次 CMS GC 时，统计数据还没有形成，是无效的，这时会跟据 Old Gen 的使用占比来进行判断是否要进行 GC。
```
// 第二种触发情况
if (!UseCMSInitiatingOccupancyOnly) {
    if (stats().valid()) {
      if (stats().time_until_cms_start() == 0.0) {
        return true;
      }
    } else {
      // We want to conservatively collect somewhat early in order
      // to try and "bootstrap" our CMS/promotion statistics;
      // this branch will not fire after the first successful CMS
      // collection because the stats should then be valid.
      if (_cmsGen->occupancy() >= _bootstrap_occupancy) {
        if (Verbose && PrintGCDetails) {
          gclog_or_tty->print_cr(
            " CMSCollector: collect for bootstrapping statistics:"
            " occupancy = %f, boot occupancy = %f", _cmsGen->occupancy(),
            _bootstrap_occupancy);
        }
        return true;
      }
    }
  }
```
那占多少比率，开始回收呢？（也就是 _bootstrap_occupancy 的值是多少呢？）

答案是 50%。或许你已经遇到过类似案例，在没有配置 UseCMSInitiatingOccupancyOnly 时，发现老年代占比到 50% 就进行了一次 CMS GC，当时的你或许还一头雾水呢。
```
 _bootstrap_occupancy = ((double)CMSBootstrapOccupancy)/(double)100;
 //参数默认值
 product(uintx, CMSBootstrapOccupancy, 50,
          "Percentage CMS generation occupancy at which to initiate CMS collection for bootstrapping collection stats")  
```
  
* 根据 Old Gen 情况判断

```
  // 第三种触发情况
  // Otherwise, we start a collection cycle if
  // old gen want a collection cycle started. Each may use
  // an appropriate criterion for making this decision.
  // XXX We need to make sure that the gen expansion
  // criterion dovetails well with this. XXX NEED TO FIX THIS
  if (_cmsGen->should_concurrent_collect()) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print_cr("CMS old gen initiated");
    }
    return true;
  }
```
```
bool ConcurrentMarkSweepGeneration::should_concurrent_collect() const {
  assert_lock_strong(freelistLock());
  if (occupancy() > initiating_occupancy()) {
    if (PrintGCDetails && Verbose) {
      gclog_or_tty->print(" %s: collect because of occupancy %f / %f  ",
        short_name(), occupancy(), initiating_occupancy());
    }
    return true;
  }
  if (UseCMSInitiatingOccupancyOnly) {
    return false;
  }
  if (expansion_cause() == CMSExpansionCause::_satisfy_allocation) {
    if (PrintGCDetails && Verbose) {
      gclog_or_tty->print(" %s: collect because expanded for allocation ",
        short_name());
    }
    return true;
  }
  if (_cmsSpace->should_concurrent_collect()) {
    if (PrintGCDetails && Verbose) {
      gclog_or_tty->print(" %s: collect because cmsSpace says so ",
        short_name());
    }
    return true;
  }
  return false;
}
```
从源码上看，这里主要分成两类：

 (1) Old Gen 空间使用占比情况与阈值比较，如果大于阈值则进行 CMS GC

也就是 "occupancy() > initiating_occupancy()"这个判断，其中 occupancy 毫无疑问是 Old Gen 当前空间的使用占比，而 initiating_occupancy 是多少呢？
 ```
 _cmsGen ->init_initiating_occupancy(CMSInitiatingOccupancyFraction, CMSTriggerRatio);
 ...
 void ConcurrentMarkSweepGeneration::init_initiating_occupancy(intx io, uintx tr) {
  assert(io <= 100 && tr <= 100, "Check the arguments");
  if (io >= 0) {
    _initiating_occupancy = (double)io / 100.0;
  } else {
    _initiating_occupancy = ((100 - MinHeapFreeRatio) +
                             (double)(tr * MinHeapFreeRatio) / 100.0)
                            / 100.0;
  }
}
 ```
可以看到当 CMSInitiatingOccupancyFraction 参数配置值大于 0，就是 “io / 100.0”；

当 CMSInitiatingOccupancyFraction 参数配置值小于 0 时（注意，默认是 -1），是 “((100 - MinHeapFreeRatio) + (double)(tr * MinHeapFreeRatio) / 100.0) / 100.0”，这到底是多少呢？

是 92%，这里就不贴出具体的计算过程了，或许你已经在某些书或者博客中了解过，CMSInitiatingOccupancyFraction 没有配置，就是 92，但是其实 CMSInitiatingOccupancyFraction 没有配置是 -1，所以阈值取后者 92%，并不是 CMSInitiatingOccupancyFraction 的值是 92。

(2) 接下来没有配置 UseCMSInitiatingOccupancyOnly 的情况

这里也分成有两小类情况：

a. Old Gen 刚因为对象分配空间而进行扩容，且成功分配空间，这时会考虑进行一次 CMS GC; 

b. 根据 CMS Gen 空闲链判断，这里有点复杂，目前也没整清楚，好在按照默认配置其实这里返回的是 false，所以默认是不用考虑这种触发条件了。

* 根据增量 GC 是否可能会失败（悲观策略）

```
  // 第四种触发情况
  // We start a collection if we believe an incremental collection may fail;
  // this is not likely to be productive in practice because it's probably too
  // late anyway.
  GenCollectedHeap* gch = GenCollectedHeap::heap();
  assert(gch->collector_policy()->is_two_generation_policy(),
         "You may want to check the correctness of the following");
  if (gch->incremental_collection_will_fail(true /* consult_young */)) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print("CMSCollector: collect because incremental collection will fail ");
    }
    return true;
  }
```

什么意思呢？两代的 GC 体系中，主要指的是 Young GC 是否会失败。如果 Young GC 已经失败或者可能会失败，JVM 就认为需要进行一次 CMS GC。
```
  bool incremental_collection_will_fail(bool consult_young) {
    // Assumes a 2-generation system; the first disjunct remembers if an
    // incremental collection failed, even when we thought (second disjunct)
    // that it would not.
    assert(heap()->collector_policy()->is_two_generation_policy(),
           "the following definition may not be suitable for an n(>2)-generation system");
    return incremental_collection_failed() ||
           (consult_young && !get_gen(0)->collection_attempt_is_safe());
  }
```
我们看两个判断条件，“incremental_collection_failed()” 和 “!get_gen(0)->collection_attempt_is_safe()”，其中任何一个通过，都会进行一次 CMS GC。

incremental_collection_failed() 这里指的是 Young GC 已经失败，至于为什么会失败一般是因为 Old Gen 没有足够的空间来容纳晋升的对象。

!get_gen(0)->collection_attempt_is_safe() 指的是 Young GC 晋升是否安全。
通过判断当前 Old Gen 剩余的空间大小是否足够容纳 Young GC 晋升的对象大小。
Young GC 到底要晋升多少是无法提前知道的，因此，这里通过统计平均每次 Young GC 晋升的大小和当前 Young GC 可能晋升的最大大小来进行比较。
```
//av_promo 是平均每次 YoungGC 晋升的大小，max_promotion_in_bytes 是当前可能的最大晋升大小（ eden+from 当前使用空间的大小）
bool   res = (available >= av_promo) || (available >= max_promotion_in_bytes);
```

* 根据 meta space 情况判断

```
  // 第五种触发情况
  if (MetaspaceGC::should_concurrent_collect()) {
      if (Verbose && PrintGCDetails) {
      gclog_or_tty->print("CMSCollector: collect for metadata allocation ");
      }
      return true;
    }
```
这里主要看 metaspace 的 _should_concurrent_collect 标志，这个标志在 meta space 进行扩容前如果配置了 CMSClassUnloadingEnabled 参数时，会进行设置。
这种情况下就会进行一次 CMS GC。因此经常会有应用启动不久，Old Gen 空间占比还很小的情况下，进行了一次 CMS GC，让你很莫名其妙，其实就是这个原因导致的。

# 总结
本文梳理了 CMS GC 的 foreground collector 和 background collector 的触发条件，foreground collector 的触发条件相对来说比较简单，而 background collector 的触发条件比较多，分成 5 大种情况，各大种情况种还有一些小的触发分支。尤其是在没有配置 UseCMSInitiatingOccupancyOnly 参数的情况下，会多出很多种触发可能，一般在生产环境是强烈建议配置 UseCMSInitiatingOccupancyOnly 参数，以便于能够比较确定的执行 CMS GC，另外，也方便排查 GC 原因。


******
> 涤生的博客。

> 转载请注明原创出处，谢谢！

> 欢迎关注我的微信公众号：「涤生的博客」，获取更多技术分享。

![涤生-微信公共号](/img/main/officialAccount.jpg)

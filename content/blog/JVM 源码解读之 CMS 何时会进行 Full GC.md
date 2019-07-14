+++
author = "涤生"
categories = ["Java","GC","JVM"]
tags = ["Java", "JVM", "GC", "CMS GC", "Full GC","foreground collector"]
date = "2019-06-15"
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "JVM 源码解读之 CMS 何时会进行 Full GC"
type = "post"
+++


# 前言
在文章 [JVM 源码解读之 CMS GC 触发条件](/blog/jvm-源码解读之-cms-gc-触发条件) 中分析了 CMS GC 触发的五类情况，并且提到 CMS GC 分为 foreground collector 和 background collector。
不管是 foreground collector 还是 background collector 使用的都是 mark-sweep 算法，分阶段进行标记清理，优点很明显-低延时，但最大的缺点是存在碎片，内存空间利用率低。因此，CMS 为了解决这个问题，在每次进行 foreground collector 之前，判断是否需要进行一次压缩式 GC。

此压缩式 GC，CMS 使用的是跟 Serial Old GC 一样的 LISP2 算法，其使用 mark-compact 来做 Full GC，一般称之为 MSC（mark-sweep-compact），它收集的范围是 Java 堆的 Young Gen 和 Old Gen，以及  metaspace（元空间）。

本文不涉及具体的收集过程，只分析 CMS 在什么情况下会进行 compact 的 Full GC。

> 本文内容是基于 JDK 8

> 什么情况下会进行一次压缩式 Full GC 呢？

# 何时会进行 FullGC？

下面这段代码就是 CMS 进行判断是进行 mark-sweep 的 foreground collector，还是进行 mark-sweep-compact 的 Full GC。主要的判断依据就是是否进行压缩，即代码中的 should_compact。

```
// Check if we need to do a compaction, or if not, whether
// we need to start the mark-sweep from scratch.
bool should_compact    = false;
bool should_start_over = false;
decide_foreground_collection_type(clear_all_soft_refs,
    &should_compact, &should_start_over);
...
if (should_compact) {
    ...
    // mark-sweep-compact
    do_compaction_work(clear_all_soft_refs);
    ...
} else {
    // mark-sweep
    do_mark_sweep_work(clear_all_soft_refs, first_state,
      should_start_over);
}
```
接下来我们就来分析下在什么情况下会进行 compact，
来看 decide_foreground_collection_type 函数，主要分为 4 种情况：

1. GC（包含 foreground collector 和 compact 的 Full GC）次数
2. GCCause 是否是用户请求式触发导致的
3. 增量 GC 是否可能会失败（悲观策略）
4. 是否清理所有 SoftReference

```
void CMSCollector::decide_foreground_collection_type(
  bool clear_all_soft_refs, bool* should_compact,
  bool* should_start_over) {
  ...
  
  // 判断是否压缩的逻辑
  
  *should_compact =
    UseCMSCompactAtFullCollection &&
    ((_full_gcs_since_conc_gc >= CMSFullGCsBeforeCompaction) ||
     GCCause::is_user_requested_gc(gch->gc_cause()) ||
     gch->incremental_collection_will_fail(true /* consult_young */));
  *should_start_over = false;
  
  if (clear_all_soft_refs && !*should_compact) {
  
    if (CMSCompactWhenClearAllSoftRefs) {
      *should_compact = true;
    } else {
    
        if (_collectorState > FinalMarking) {
        _collectorState = Resetting; // skip to reset to start new cycle
        reset(false /* == !asynch */);
        *should_start_over = true;
      } 
    }
  }
}
```


> 接下来我们具体看每种情况


## 1. GC（包含 foreground collector 和 compact 的 Full GC）次数
```
// UseCMSCompactAtFullCollection 参数值默认是 true
UseCMSCompactAtFullCollection &&
    ((_full_gcs_since_conc_gc >= CMSFullGCsBeforeCompaction)
```
这里说的 GC 次数 _full_gcs_since_conc_gc，指的是从上次 background collector 后，foreground collector 和 compact 的 Full GC 的次数，只要次数大于等于 CMSFullGCsBeforeCompaction 参数阈值，就表示可以进行一次压缩式的 Full GC。
（CMSFullGCsBeforeCompaction 参数默认是 0，意味着默认是要进行压缩式的 Full GC。）


## 2. GCCause 是否是用户请求式触发导致
```
 inline static bool is_user_requested_gc(GCCause::Cause cause) {
    return (cause == GCCause::_java_lang_system_gc ||
            cause == GCCause::_jvmti_force_gc);
  }
```
用户请求式触发导致的 GCCause 指的是 _java_lang_system_gc（即 System.gc()）或者 _jvmti_force_gc（即 JVMTI 方式的强制 GC）
意味着只要是 System.gc（前提没有配置 ExplicitGCInvokesConcurrent 参数）调用或者 JVMTI 方式的强制 GC 都会进行一次压缩式的 Full GC。

## 3. 增量 GC 是否可能会失败（悲观策略）
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
[JVM 源码解读之 CMS GC 触发条件](/blog/jvm-源码解读之-cms-gc-触发条件) 文章中也提到了这块内容，
指的是两代的 GC 体系中，主要指的是 Young GC 是否会失败。如果 Young GC 已经失败或者可能会失败，CMS 就认为可能存在碎片导致的，需要进行一次压缩式的 Full GC。

“incremental_collection_failed()” 这里指的是 Young GC 已经失败，至于为什么会失败一般是因为 Old Gen 没有足够的空间来容纳晋升的对象，比如常见的 “promotion failed” 。

“!get_gen(0)->collection_attempt_is_safe()” 指的是 Young Gen 存活对象晋升是否可能会失败。
通过判断当前 Old Gen 剩余的空间大小是否足够容纳 Young GC 晋升的对象大小。
Young GC 到底要晋升多少是无法提前知道的，因此，这里通过统计平均每次 Young GC 晋升的大小和当前 Young GC 可能晋生的最大大小来进行比较。

下面展示的就是 collection_attempt_is_safe 函数的代码：
```
bool DefNewGeneration::collection_attempt_is_safe() {
  if (!to()->is_empty()) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print(" :: to is not empty :: ");
    }
    return false;
  }
  if (_next_gen == NULL) {
    GenCollectedHeap* gch = GenCollectedHeap::heap();
    _next_gen = gch->next_gen(this);
  }
  return _next_gen->promotion_attempt_is_safe(used());
}

```

## 4. 是否清理所有 SoftReference
```
if (clear_all_soft_refs && !*should_compact) {
  
    if (CMSCompactWhenClearAllSoftRefs) {
      *should_compact = true;
    } 
    ...
```
SoftReference 软引用，你应该了解它的特性，一般是在内存不够的时候，GC 会回收相关对象内存。这里说的就是需要回收所有软引用的情况，在配置了 CMSCompactWhenClearAllSoftRefs 参数的情况下，会进行一次压缩式的 Full GC。

> JDK 1.9 有变更：
> 彻底去掉了 CMS forground collector 的功能，也就是说除了 background collector，就是压缩式的 Full GC。自然（UseCMSCompactAtFullCollection、CMSFullGCsBeforeCompaction 这两个参数也已经不在支持了。

# 总结
本文着重介绍了 CMS 在以下 4 种情况：

* GC（包含 foreground collector 和 compact 的 Full GC）次数
* GCCause 是否是用户请求式触发导致
* 增量 GC 是否可能会失败（悲观策略）
* 是否清理所有 SoftReference

会进行压缩式的 Full GC，并且详细介绍了每种情况下的触发条件。
我们在 GC 调优时应该尽可能的避免压缩式的 Full GC，因为其使用的是 Serial Old GC 类似算法，它是单线程对全堆以及 metaspace 进行回收，STW 的时间会特别长，对业务系统的可用性影响比较大。


******
> 涤生的博客。

> 转载请注明原创出处，谢谢！

> 欢迎关注我的微信公众号：「涤生的博客」，获取更多技术分享。

![涤生-微信公共号](/img/main/officialAccount.jpg)

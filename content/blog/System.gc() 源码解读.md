+++
author = "涤生"
categories = ["Java","GC","JVM"]
tags = ["Java", "JVM", "GC", "System.gc()"]
date = "2018-01-14"
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "System.gc() 源码解读"
type = "post"
+++

# 介绍
System.gc()，大家应该也有所了解，是 JDK 提供的触发 Full GC 的一种方式，会触发 Full GC，其间会 stop the world，对业务影响较大，一般情况下不会直接使用。

> 那它是如何实现的呢？
> 另外有哪些参数可以进行优化呢？
> 我们带着问题来对相关源码解读一下。

# 实现
## JDK实现

```
/**
     * Runs the garbage collector.
     * <p>
     * Calling the <code>gc</code> method suggests that the Java Virtual
     * Machine expend effort toward recycling unused objects in order to
     * make the memory they currently occupy available for quick reuse.
     * When control returns from the method call, the Java Virtual
     * Machine has made a best effort to reclaim space from all discarded
     * objects.
     * <p>
     * The call <code>System.gc()</code> is effectively equivalent to the
     * call:
     * <blockquote><pre>
     * Runtime.getRuntime().gc()
     * </pre></blockquote>
     *
     * @see     java.lang.Runtime#gc()
     */
    public static void gc() {
        Runtime.getRuntime().gc();
    }
```
其实就是调用了 Runtime 类的 gc 方法。

```
/**
     * Runs the garbage collector.
     * Calling this method suggests that the Java virtual machine expend
     * effort toward recycling unused objects in order to make the memory
     * they currently occupy available for quick reuse. When control
     * returns from the method call, the virtual machine has made
     * its best effort to recycle all discarded objects.
     * <p>
     * The name <code>gc</code> stands for "garbage
     * collector". The virtual machine performs this recycling
     * process automatically as needed, in a separate thread, even if the
     * <code>gc</code> method is not invoked explicitly.
     * <p>
     * The method {@link System#gc()} is the conventional and convenient
     * means of invoking this method.
     */
    public native void gc();
```
Runtime 类的 gc 方法是个 native 方法，所以只能进入 JVM 代码去看其真正的实现了。

## JVM实现
打开 openjdk 的源码，Runtime 类的 gc 方法在 Runtime.c 文件中有具体的实现

```
JNIEXPORT void JNICALL
Java_java_lang_Runtime_gc(JNIEnv *env, jobject this)
{
    JVM_GC();
}
```
可以看到直接调用了 JVM_GC() 方法，这个方法的实现在 jvm.cpp 中

```
JVM_ENTRY_NO_ENV(void, JVM_GC(void))
  JVMWrapper("JVM_GC");
  if (!DisableExplicitGC) {
    Universe::heap()->collect(GCCause::_java_lang_system_gc);
  }
JVM_END
```
很明显最终调用的是 heap 的 collect 方法，gcCause 为 _java_lang_system_gc。这里有个注意点就是 DisableExplicitGC，如果是 true 就不会执行 collect 方法，也就是使得 System.gc() 无效，DisableExplicitGC 这个参数对应到的配置就是 -XX:+DisableExplicitGC，默认是 false，如果配置了，就是 true。

heap 有几种，具体是哪种 heap，需要看 gc 算法，如常用的 CMS GC 的对应的 heap 是 GenCollectedHeap，所以我们再看看 GenCollectedHeap.cpp 对应的 collect 方法

```
void GenCollectedHeap::collect(GCCause::Cause cause) {
  if (should_do_concurrent_full_gc(cause)) {
#ifndef SERIALGC
    // mostly concurrent full collection
    collect_mostly_concurrent(cause);
#else  // SERIALGC
    ShouldNotReachHere();
#endif // SERIALGC
  } else {
#ifdef ASSERT
    if (cause == GCCause::_scavenge_alot) {
      // minor collection only
      collect(cause, 0);
    } else {
      // Stop-the-world full collection
      collect(cause, n_gens() - 1);
    }
#else
    // Stop-the-world full collection
    collect(cause, n_gens() - 1);
#endif
  }
}
```
这个方法首先通过 should_do_concurrent_full_gc 方法判断是不是进行一次并发 Full GC，如果是则调用 collect_mostly_concurrent 方法，进行并发 Full GC；如果不是则一般会走到 collect(cause, n_gens() - 1) 这段逻辑，进行 Stop the world Full GC，我们就称之为一般 Full GC。

我们先看看 should_do_concurrent_full_gc 到底有哪些条件

```
bool GenCollectedHeap::should_do_concurrent_full_gc(GCCause::Cause cause) {
  return UseConcMarkSweepGC &&
         ((cause == GCCause::_gc_locker && GCLockerInvokesConcurrent) ||
          (cause == GCCause::_java_lang_system_gc && ExplicitGCInvokesConcurrent));
}
```
很明显如果是 CMS GC，则判断 GCCause，如果是 _java_lang_system_gc 并且  ExplicitGCInvokesConcurrent 为 true 则返回 true，这里又引出了另一个参数 ExplicitGCInvokesConcurrent，如果配置了 -XX:+ExplicitGCInvokesConcurrent，则返回 true，进行并行 Full GC，默认为 false。

### 并发 Full GC 
我们接着先来看 collect_mostly_concurrent，是如何进行并行 Full GC。

```
void GenCollectedHeap::collect_mostly_concurrent(GCCause::Cause cause) {
  assert(!Heap_lock->owned_by_self(), "Should not own Heap_lock");

  MutexLocker ml(Heap_lock);
  // Read the GC counts while holding the Heap_lock
  unsigned int full_gc_count_before = total_full_collections();
  unsigned int gc_count_before      = total_collections();
  {
    MutexUnlocker mu(Heap_lock);
    VM_GenCollectFullConcurrent op(gc_count_before, full_gc_count_before, cause);
    VMThread::execute(&op);
  }
}

```

最终通过 VMThread 来进行 VM_GenCollectFullConcurrent 中的 void VM_GenCollectFullConcurrent::doit() 方法来进行回收，这里代码有点多就不展开了，这里最终执行了一次 Young GC 来回收 Young Gen，另外执行了下面这个方法

```
void CMSCollector::request_full_gc(unsigned int full_gc_count, GCCause::Cause cause) {
  GenCollectedHeap* gch = GenCollectedHeap::heap();
  unsigned int gc_count = gch->total_full_collections();
  if (gc_count == full_gc_count) {
    MutexLockerEx y(CGC_lock, Mutex::_no_safepoint_check_flag);
    _full_gc_requested = true;
    _full_gc_cause = cause;
    CGC_lock->notify();   // nudge CMS thread
  } else {
    assert(gc_count > full_gc_count, "Error: causal loop");
  }
}
```
这里主要看 _full_gc_requested 设置成 true 以及唤醒 CMS background 回收线程。
大家可能了解过 CMS GC 有个后台线程一直在扫描，是否进行一次 CMS GC，这个线程默认 2s 进行一次扫描，其中有个判断条件 _full_gc_requested 是否为 true，如果为 true，进行一次 CMS GC，对 Old Gen 和 Perm Gen 进行一次回收。

### 一般 Full GC

一般的 Full GC 会走下面逻辑

```
void GenCollectedHeap::collect_locked(GCCause::Cause cause, int max_level) {
  if (_preloading_shared_classes) {
    report_out_of_shared_space(SharedPermGen);
  }
  // Read the GC count while holding the Heap_lock
  unsigned int gc_count_before      = total_collections();
  unsigned int full_gc_count_before = total_full_collections();
  {
    MutexUnlocker mu(Heap_lock);  // give up heap lock, execute gets it back
    VM_GenCollectFull op(gc_count_before, full_gc_count_before,
                         cause, max_level);
    VMThread::execute(&op);
  }
}
```
通过 VMThread 来调用 VM_GenCollectFull 中的 void VM_GenCollectFull::doit() 方法来进行回收。

```
void VM_GenCollectFull::doit() {
  SvcGCMarker sgcm(SvcGCMarker::FULL);

  GenCollectedHeap* gch = GenCollectedHeap::heap();
  GCCauseSetter gccs(gch, _gc_cause);
  gch->do_full_collection(gch->must_clear_all_soft_refs(), _max_level);
}
```
这里最终会通过 GenCollectedHeap 的 do_full_collection 方法进行一次 Full GC，会回收 Young、Old、Perm 区，并且即使 Old Gen 使用的是 CMS GC，也会对 Old Gen 进行 compact，也就是 MSC，标记-清除-压缩。

# 总结
System.gc() 会触发 Full GC，可以通过 -XX:+DisableExplicitGC 参数屏蔽 System.gc()，在使用 CMS GC 的前提下，也可以使用 -XX:+ExplicitGCInvokesConcurrent 参数来进行并行 Full GC，提升性能。
不过，一般不推荐使用 System.gc()，因为 Full GC 耗时比较长，对应用影响较大，如前段时间的一个案例：[依赖包滥用 System.gc() 导致的频繁 Full GC](/blog/依赖包滥用-system.gc-导致的-full-gc/)。
并且也不建议设置 -XX:+DisableExplicitGC，特别是在有使用堆外内存的情况下，如果堆外内存申请不到足够的空间，jdk 会触发一次 System.gc()，来进行回收，如果屏蔽了，最后一根救命稻草也就失效了，自然就 OOM了。


******
> 涤生的博客。

> 转载请注明原创出处，谢谢！

> 欢迎关注我的微信公众号：「涤生的博客」，获取更多技术分享。

![涤生-微信公共号](/img/main/officialAccount.jpg)
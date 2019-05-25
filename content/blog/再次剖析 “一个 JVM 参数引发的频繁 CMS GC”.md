+++
author = "涤生"
categories = ["Java", "GC"]
tags = ["Java", "JVM", "GC", "CMS GC", "优化"]
date = "2019-04-10"
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "再次剖析 “一个 JVM 参数引发的频繁 CMS GC”"
type = "post"

+++

## 前言
前几天这篇《[一个 JVM 参数引发的频繁 CMS GC](/blog/一个-jvm-参数引发的频繁-cms-gc)》文章发出之后，反应比较激烈，因为这可能与同学们通常 GC 优化经验相悖，通常有很多业务都通过添加 -XX:+CMSScavengeBeforeRemark 参数，来降低 CMS-remark 的时间，进而提升业务的性能以及可用性。

## 背景
这里给出几位同学比较典型的的想法和建议：
“同学1”：“-XX:+CMSScavengeBeforeRemark 参数引发的频繁 CMS GC 有失偏颇，其实根本原因是第一次 CMS GC 过程中的 Young GC 发生了 ‘promotion failed’ 导致了 to space 不为空”。

“同学2”：“我们线上也用了这个参数，没有出现频繁 CMS GC 的现象，我猜测你那种是特殊场景导致的，并不是用了那个参数就一定会导致频繁 CMS GC，大部分情况下加上这个参数是有好处的”。

“同学3”：“是不是可以降低 -XX:CMSInitiatingOccupancyFraction 参数的值来，比如70，让 Young GC成功”。

针对这几位同学提到的想法和建议，我又重新思考这个案例，来解答下这些的问题。因此，这篇文章是《[一个 JVM 参数引发的频繁 CMS GC](/blog/一个-jvm-参数引发的频繁-cms-gc)》的进阶版本。

## 主要内容
本文主要讲解：
> 1. 纠正《[一个 JVM 参数引发的频繁 CMS GC](/blog/一个-jvm-参数引发的频繁-cms-gc)》文章中的一个分析错误
> 2. -XX:+CMSScavengeBeforeRemark 参数到底是不是引起频繁 CMS GC 的原因
> 3. 什么场景会出现这种问题
> 4. 有哪些优化策略

## 内容
### 纠正《[一个 JVM 参数引发的频繁 CMS GC](/blog/一个-jvm-参数引发的频繁-cms-gc)》文章中的一个错误
这里纠正《[一个 JVM 参数引发的频繁 CMS GC](/blog/一个-jvm-参数引发的频繁-cms-gc)》文中提到的关于 “OldGen 的使用占比情况都没有达到 80%，什么原因导致的 CMS GC” 问题原因分析中的一个错误。

引用下《[一个 JVM 参数引发的频繁 CMS GC](/blog/一个-jvm-参数引发的频繁-cms-gc)》文章的相关内容：
“我们再看看 collection_attempt_is_safe() 函数的实现，会让你豁然开朗，if (!to()->is_empty()) return false，刚好满足了每次 Young GC to space 不为空。因此，是在这里 _incremental_collection_failed 被设置成 true，导致每隔 2s 触发一次 CMS GC，这就解释了为什么 OldGen 的使用占比情况都没有达到 80%，也会触发 CMS GC。”

```
bool DefNewGeneration::collection_attempt_is_safe() {
  if (!to()->is_empty()) {
    log_trace(gc)(":: to is not empty ::");
    return false;
  }
  if (_old_gen == NULL) {
    GenCollectedHeap* gch = GenCollectedHeap::heap();
    _old_gen = gch->old_gen();
  }
  return _old_gen->promotion_attempt_is_safe(used());
}
```

这里说的是 CMS GC 过程中由于配置了 -XX:+CMSScavengeBeforeRemark 参数引起了 Young GC，而 Young GC 过程中由于 to space 不为空，导致 _incremental_collection_failed 被设置成 true，致使 CMS 的 background collector 每隔 2s 检查一次 GC 触发条件，发现 _incremental_collection_failed 被设置成 true，所以触发了下一次的 CMS GC。

这段内容是错误的，因为 -XX:+CMSScavengeBeforeRemark 参数引起了 Young GC 发生在 Final Remark 阶段，而在 CMS 的 Sweep 阶段，是会将 _incremental_collection_failed 重置成 false 的，因此，这不是这个原因导致下一次的 CMS GC。
```
// Now that sweeping has been completed, we clear
  // the incremental_collection_failed flag,
  // thus inviting a younger gen collection to promote into
  // this generation. If such a promotion may still fail,
  // the flag will be set again when a young collection is
  // attempted.
  GenCollectedHeap* gch = GenCollectedHeap::heap();
  gch->clear_incremental_collection_failed();  // Worth retrying as fresh space may have been freed up
  gch->update_full_collections_completed(_collection_count_start);
```
那究竟是什么原因导致的下一次的 CMS GC 呢？
我们再看看 CMS background collector 中影响 incremental_collection_will_fail 函数返回值的另一个条件 “consult_young && !_young_gen->collection_attempt_is_safe()”，consult_young 传参设置的是 true 因此我们只需看 _young_gen->collection_attempt_is_safe()。这其实就是上面提到的 Young GC 过程中调用的 collection_attempt_is_safe() 函数，因为 to space 一直不为空，所以这个方法的返回值一直是 false，而 “consult_young && !_young_gen->collection_attempt_is_safe()” 的返回值就是true，导致下一次 CMS GC的触发。
```
// Returns true if an incremental collection is likely to fail.
  // We optionally consult the young gen, if asked to do so;
  // otherwise we base our answer on whether the previous incremental
  // collection attempt failed with no corrective action as of yet.
  bool incremental_collection_will_fail(bool consult_young) {
    // The first disjunct remembers if an incremental collection failed, even
    // when we thought (second disjunct) that it would not.
    return incremental_collection_failed() ||
           (consult_young && !_young_gen->collection_attempt_is_safe());
  }
  bool incremental_collection_failed() const {
    return _incremental_collection_failed;
  }
```

### -XX:+CMSScavengeBeforeRemark 参数到底是不是引起频繁 CMS GC 的原因
从上面对于下一次 CMS GC 触发原因分析，其实主要原因是 YoungGen 的 to space 不为空，导致后续持续的 CMS GC，因为我们得定位下什么原因导致了 to space 不为空。
我们来看看第一次 CMS GC 的日志
#### 第一次 CMS GC日志
```
2019-03-28T20:05:06.906+0800: 3644459.373: [GC (CMS Initial Mark) [1 CMS-initial-mark: 2935428K(3354624K)] 3160044K(5242112K), 0.0586708 secs] [Times: user=0.22 sys=0.00, real=0.06 secs]
2019-03-28T20:05:06.965+0800: 3644459.432: Total time for which application threads were stopped: 0.0616049 seconds, Stopping threads took: 0.0001381 seconds
2019-03-28T20:05:06.965+0800: 3644459.432: [CMS-concurrent-mark-start]
2019-03-28T20:05:08.066+0800: 3644460.533: [CMS-concurrent-mark: 1.101/1.101 secs] [Times: user=1.57 sys=0.05, real=1.10 secs]
2019-03-28T20:05:08.066+0800: 3644460.533: [CMS-concurrent-preclean-start]
2019-03-28T20:05:08.076+0800: 3644460.543: [CMS-concurrent-preclean: 0.010/0.010 secs] [Times: user=0.01 sys=0.01, real=0.01 secs]
2019-03-28T20:05:08.076+0800: 3644460.543: [CMS-concurrent-abortable-preclean-start]
2019-03-28T20:05:10.177+0800: 3644462.645: Application time: 3.2124140 seconds
{Heap before GC invocations=18476 (full 731):
 par new generation   total 1887488K, used 1887488K [0x0000000673400000, 0x00000006f3400000, 0x00000006f3400000)
  eden space 1677824K, 100% used [0x0000000673400000, 0x00000006d9a80000, 0x00000006d9a80000)
  from space 209664K, 100% used [0x00000006e6740000, 0x00000006f3400000, 0x00000006f3400000)
  to   space 209664K,   0% used [0x00000006d9a80000, 0x00000006d9a80000, 0x00000006e6740000)
 concurrent mark-sweep generation total 3354624K, used 2935428K [0x00000006f3400000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 90225K, capacity 91504K, committed 91776K, reserved 1130496K
  class space    used 9517K, capacity 9806K, committed 9856K, reserved 1048576K
2019-03-28T20:05:10.179+0800: 3644462.647: [GC (Allocation Failure) 3644462.647: [ParNew: 1887488K->201195K(1887488K), 0.4228807 secs] 4822916K->3279195K(5242112K), 0.4231546 secs] [Times: user=1.54 sys=0.00, real=0.42 secs] 
Heap after GC invocations=18477 (full 731):
 par new generation   total 1887488K, used 201195K [0x0000000673400000, 0x00000006f3400000, 0x00000006f3400000)
  eden space 1677824K,   0% used [0x0000000673400000, 0x0000000673400000, 0x00000006d9a80000)
  from space 209664K,  95% used [0x00000006d9a80000, 0x00000006e5efae68, 0x00000006e6740000)
  to   space 209664K,   0% used [0x00000006e6740000, 0x00000006e6740000, 0x00000006f3400000)
 concurrent mark-sweep generation total 3354624K, used 3078000K [0x00000006f3400000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 90225K, capacity 91504K, committed 91776K, reserved 1130496K
  class space    used 9517K, capacity 9806K, committed 9856K, reserved 1048576K
}
2019-03-28T20:05:10.603+0800: 3644463.070: Total time for which application threads were stopped: 0.4258929 seconds, Stopping threads took: 0.0001722 seconds
2019-03-28T20:05:10.904+0800: 3644463.372: [CMS-concurrent-abortable-preclean: 2.397/2.828 secs] [Times: user=6.22 sys=0.10, real=2.83 secs]
2019-03-28T20:05:10.904+0800: 3644463.372: Application time: 0.3012271 seconds
2019-03-28T20:05:10.907+0800: 3644463.374: [GC (CMS Final Remark) [YG occupancy: 434406 K (1887488 K)]{Heap before GC invocations=18477 (full 731):
 par new generation   total 1887488K, used 434406K [0x0000000673400000, 0x00000006f3400000, 0x00000006f3400000)
  eden space 1677824K,  13% used [0x0000000673400000, 0x00000006817bed10, 0x00000006d9a80000)
  from space 209664K,  95% used [0x00000006d9a80000, 0x00000006e5efae68, 0x00000006e6740000)
  to   space 209664K,   0% used [0x00000006e6740000, 0x00000006e6740000, 0x00000006f3400000)
 concurrent mark-sweep generation total 3354624K, used 3078000K [0x00000006f3400000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 90225K, capacity 91504K, committed 91776K, reserved 1130496K
  class space    used 9517K, capacity 9806K, committed 9856K, reserved 1048576K
2019-03-28T20:05:10.907+0800: 3644463.375: [GC (CMS Final Remark) 3644463.375: [ParNew (promotion failed): 434406K->315478K(1887488K), 5.8407801 secs] 3512406K->3486710K(5242112K), 5.8410096 secs] [Times: user=6.84 sys=1.31, real=5.84 secs]
Heap after GC invocations=18478 (full 731):
 par new generation   total 1887488K, used 315478K [0x0000000673400000, 0x00000006f3400000, 0x00000006f3400000)
  eden space 1677824K,  13% used [0x0000000673400000, 0x00000006817bed10, 0x00000006d9a80000)
  from space 209664K,  39% used [0x00000006e6740000, 0x00000006eb796e60, 0x00000006f3400000)
  to   space 209664K,  95% used [0x00000006d9a80000, 0x00000006e5efae68, 0x00000006e6740000)
 concurrent mark-sweep generation total 3354624K, used 3171231K [0x00000006f3400000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 90225K, capacity 91504K, committed 91776K, reserved 1130496K
  class space    used 9517K, capacity 9806K, committed 9856K, reserved 1048576K
}
3644469.216: [Rescan (parallel) , 0.3096135 secs]3644469.525: [weak refs processing, 0.0009228 secs]3644469.526: [class unloading, 0.0797710 secs]3644469.606: [scrub symbol table, 0.0229535 secs]3644469.629: [scrub string table, 0.0020416 secs][1 CMS-remark: 3171231K(3354624K)] 3486710K(5242112K), 6.2593934 secs] [Times: user=8.10 sys=1.36, real=6.26 secs]
2019-03-28T20:05:17.166+0800: 3644469.634: Total time for which application threads were stopped: 6.2622888 seconds, Stopping threads took: 0.0002099 seconds
2019-03-28T20:05:17.167+0800: 3644469.634: [CMS-concurrent-sweep-start]
2019-03-28T20:05:17.176+0800: 3644469.644: Application time: 0.0100218 seconds
2019-03-28T20:05:17.179+0800: 3644469.647: Total time for which application threads were stopped: 0.0025500 seconds, Stopping threads took: 0.0001934 seconds
2019-03-28T20:05:18.179+0800: 3644470.647: Application time: 1.0001731 seconds
2019-03-28T20:05:18.182+0800: 3644470.649: Total time for which application threads were stopped: 0.0026811 seconds, Stopping threads took: 0.0001358 seconds
2019-03-28T20:05:21.000+0800: 3644473.468: Application time: 2.8185985 seconds
2019-03-28T20:05:21.003+0800: 3644473.471: Total time for which application threads were stopped: 0.0029238 seconds, Stopping threads took: 0.0001172 seconds
2019-03-28T20:05:21.013+0800: 3644473.481: Application time: 0.0097451 seconds
2019-03-28T20:05:21.019+0800: 3644473.487: Total time for which application threads were stopped: 0.0060990 seconds, Stopping threads took: 0.0002775 seconds
2019-03-28T20:05:21.734+0800: 3644474.201: Application time: 0.7144315 seconds
2019-03-28T20:05:21.736+0800: 3644474.204: Total time for which application threads were stopped: 0.0026804 seconds, Stopping threads took: 0.0001238 seconds
2019-03-28T20:05:22.203+0800: 3644474.671: [CMS-concurrent-sweep: 5.019/5.037 secs] [Times: user=5.28 sys=0.27, real=5.03 secs]
2019-03-28T20:05:22.204+0800: 3644474.671: [CMS-concurrent-reset-start]
2019-03-28T20:05:22.211+0800: 3644474.678: [CMS-concurrent-reset: 0.007/0.007 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
2019-03-28T20:05:22.238+0800: 3644474.706: Application time: 0.5016696 seconds
2019-03-28T20:05:22.241+0800: 3644474.708: Total time for which application threads were stopped: 0.0026876 seconds, Stopping threads took: 0.0001305 seconds
2019-03-28T20:05:22.438+0800: 3644474.905: Application time: 0.1970764 seconds
2019-03-28T20:05:22.440+0800: 3644474.908: Total time for which application threads were stopped: 0.0027034 seconds, Stopping threads took: 0.0001344 seconds
2019-03-28T20:05:23.441+0800: 3644475.908: Application time: 1.0001304 seconds
2019-03-28T20:05:23.443+0800: 3644475.911: Total time for which application threads were stopped: 0.0024875 seconds, Stopping threads took: 0.0001316 seconds
2019-03-28T20:05:24.210+0800: 3644476.678: Application time: 0.7671567 seconds
```
通过日志可知，由于配置 -XX:+CMSScavengeBeforeRemark 参数，CMS Final Remark 阶段触发了一次 Young GC，而这时 OldGen 所剩无几没有足够的空闲空间放置 Young GC 晋升的对象，导致 Young GC 回收停止，致使 to space 不为空。
因此 to space 不为空的直接导火索是 Young GC “promotion failed”，那跟 -XX:+CMSScavengeBeforeRemark 参数有没有关系呢？当然有关系，因为这次 Young GC 就是由于配置了 -XX:+CMSScavengeBeforeRemark 参数，而触发的。
但是有人要说 “如果没有配置 -XX:+CMSScavengeBeforeRemark 参数，CMS GC 过程中也可能会发生 Young GC 的，如果 Young GC 晋升失败，一样会导致 to space 不为空”。
这种说法乍一看是合理的，CMS GC 过程中有几个阶段是并发，确实会出现由于Allocation Failure 而导致 Young GC，但是 Allocation Failure 触发的 Young GC，如果出现晋升失败大部分情况下都是会直接触发 Full GC，来直接对整个堆进行回收，不会出现后续的频繁 CMS GC。
因此，-XX:+CMSScavengeBeforeRemark 参数是可能会触发频繁 CMS GC 的。

### 什么场景会出现这种问题
有些同学可能要问，我们线上很多业务都配置了 -XX:+CMSScavengeBeforeRemark 参数，为什么都没出现过频繁 CMS GC 的问题，该问题在什么场景下比较易触发呢？该问题的出现条件是什么？
要了解在什么场景下比较容易触发，我们先来看看什么条件下会触发这个问题。
主要有两个条件：
* -XX:+CMSScavengeBeforeRemark 参数引起的 “promotion failed”  的 Young GC，致使 to space 不为空；
* 长时间没有 Allocation Failure 引起的 Young GC。

了解了这两个条件，就能解释为什么一般业务配置了这个参数，但是没有出现过类似频繁 CMS GC 的情况。
大部分业务很少出现因为 -XX:+CMSScavengeBeforeRemark 参数引起的 “promotion failed” 的 Young GC。
另外，一般业务的 Allocation Failure 引起的 Young GC 比较频繁，即使发生第一个条件，也很快会出现 Allocation Failure 引起的 “promotion failed” 的 Young GC，进而触发 Full GC 恢复，因此，即使出现时间持续时间也很短。

文章中的案例，就是符合两个条件，进而导致频繁 CMS GC。
* 晋升对象比较大，很容易导致 “-XX:+CMSScavengeBeforeRemark 参数引起的 “promotion failed”  的 Young GC” 这种情况；
* 长时间才会出现一次 Young GC， 又满足了 “长时间没有 Allocation Failure 引起的 Young GC” 条件。

### 有哪些优化策略

* 去掉 -XX:+CMSScavengeBeforeRemark 参数
CMS GC 过程中不强制触发 Young GC，自然降低了 “promotion failed” Young GC 的出现次数，也不会导致 to space 不为空，进而降低了导致频繁 CMS GC 的可能。

* 降低 promotion failed 发生的概率
可以通过降低 -XX:CMSInitiatingOccupancyFraction 参数的阈值，来降低 CMS GC 过程中因为 -XX:+CMSScavengeBeforeRemark 参数导致的 Young GC 出现 “promotion failed”  的概率。

* 提高 Young GC 的频率
可以通过减小 YoungGen 的大小，来加快因 Allocation Failure 而触发 Young GC，这样即使出现该问题，也能尽快触发 Full GC 恢复。

## 总结
针对上述分析可知，-XX:+CMSScavengeBeforeRemark 参数确实是与案例中频繁 CMS GC 是相关的，只是出现的该问题的条件比较苛刻，不太容易出现；另外，针对问题出现的条件，是有些办法可以避免该问题或者降低该问题的出现概率。


******
欢迎关注我的微信公众号：「涤生的博客」，获取更多技术分享。

![涤生-微信公共号](/img/main/officialAccount.jpg)

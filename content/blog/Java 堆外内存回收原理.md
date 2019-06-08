+++
author = "涤生"
categories = ["Java","GC","JVM"]
tags = ["Java", "GC", "JVM", "堆外内存"]
date = "2017-04-13"
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "Java 堆外内存回收原理"
type = "post"

+++

# DirectByteBuffer 简介
DirectByteBuffer 这个类是 JDK 提供使用堆外内存的一种途径，当然常见的业务开发一般不会接触到，即使涉及到也可能是框架（如 Netty、RPC 等）使用的，对框架使用者来说也是透明的。

# 堆外内存的优势
堆外内存优势在 IO 操作上，对于网络 IO，使用 Socket 发送数据时，能够节省堆内存到堆外内存的数据拷贝，所以性能更高。看过 Netty 源码的同学应该了解，Netty 使用堆外内存池来实现零拷贝技术。对于磁盘 IO 时，也可以使用内存映射，来提升性能。
另外，更重要的几乎不用考虑堆内存烦人的 GC 问题。

# 堆外内存的创建
我们直接来看代码，首先向 Bits 类申请额度，Bits 类内部维护着当前已经使用的堆外内存值，会 check 当前申请的大小与已经使用的内存大小是否超过总的堆外内存大小（默认大小与堆内存差不多，其实是有细微区别的，拿 CMS GC 来举例，它的大小是新生代的最大值 - 一个 survivor 的大小 + 老生代的最大值），可以使用 -XX:MaxDirectMemorySize 参数指定堆外内存最大大小。
```java
   //
    DirectByteBuffer(int cap) {                   // package-private

        super(-1, 0, cap, cap);
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);

        long base = 0;
        try {
            base = unsafe.allocateMemory(size);
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        unsafe.setMemory(base, size, (byte) 0);
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;

    }

```
如果 check 不通过，会主动执行 System.gc()，然后 sleep 100 毫秒，再进行 check，如果内存还是不足，就抛出 OOM Error。

如果 check 通过，就会调用 unsafe.allocateMemory 真正分配内存，返回内存地址，然后再将内存清 0。题外话，这个 unsafe 命名看着是不是很吓人，这个 unsafe 不是说不安全，而是 JDK 内部使用的类，不推荐外部使用，所以叫 unsafe，Netty 源码内部也有类似命名。

由于申请内存前可能会调用 System.gc()，所以谨慎设置 -XX:+DisableExplicitGC 这个选项，这个参数作用是禁止代码中显示触发的 Full GC。

# 堆外内存的回收
```java
cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
```
看到这段代码从成员的命名上就应该知道，是用来回收堆外内存的。确实，但是它是如何工作的呢？接下来我们看看 Cleaner 类。
```java
public class Cleaner extends PhantomReference {
    private static final ReferenceQueue dummyQueue = new ReferenceQueue();
    private static Cleaner first = null;
    private Cleaner next = null;
    private Cleaner prev = null;
    private final Runnable thunk;

    private static synchronized Cleaner add(Cleaner var0) {
       ...
    }

    private static synchronized boolean remove(Cleaner var0) {
        ...
    }

    private Cleaner(Object var1, Runnable var2) {
        super(var1, dummyQueue);
        this.thunk = var2;
    }

    public static Cleaner create(Object var0, Runnable var1) {
        return var1 == null?null:add(new Cleaner(var0, var1));
    }

    public void clean() {
        if(remove(this)) {
            try {
                this.thunk.run();
            } catch (final Throwable var2) {
                AccessController.doPrivileged(new PrivilegedAction() {
                    public Void run() {
                        if(System.err != null) {
                            (new Error("Cleaner terminated abnormally", var2)).printStackTrace();
                        }

                        System.exit(1);
                        return null;
                    }
                });
            }

        }
    }
}

```
Cleaner 类，内部维护了一个 Cleaner 对象的链表，通过 create(Object, Runnable) 方法创建 cleaner 对象，调用自身的 add 方法，将其加入到链表中。
更重要的是提供了 clean 方法，clean 方法首先将对象自身从链表中删除，保证只调用一次，然后执行 this.thunk 的 run 方法，thunk 就是由创建时传入的 Runnable 参数，也就是说 clean 只负责触发 Runnable 的 run 方法，至于 Runnable 做什么任务它不关心。

那 DirectByteBuffer 传进来的 Runnable是什么呢？

```java
 private static class Deallocator
        implements Runnable
    {

        private static Unsafe unsafe = Unsafe.getUnsafe();

        private long address;
        private long size;
        private int capacity;

        private Deallocator(long address, long size, int capacity) {
            assert (address != 0);
            this.address = address;
            this.size = size;
            this.capacity = capacity;
        }

        public void run() {
            if (address == 0) {
                // Paranoia
                return;
            }
            unsafe.freeMemory(address);
            address = 0;
            Bits.unreserveMemory(size, capacity);
        }

    }
```

Deallocator 类的对象就是 DirectByteBuffer 中的 cleaner 传进来的 Runnable 参数类，我们直接看 run 方法 unsafe.freeMemory 释放内存，然后更新 Bits 里已使用的内存数据。

接下来我们关注各个环节是如何串起来的？这里主要讲两种回收方式：一种是自动回收，一种是手动回收。

## 如何自动回收？
Java 是不用用户去管理内存的，所以 Java 对堆外内存 默认是自动回收的。
它是 由 GC 模块负责的，在 GC 时会扫描 DirectByteBuffer 对象是否有有效引用指向该对象，如没有，在回收 DirectByteBuffer 对象的同时且会回收其占用的堆外内存。但是 JVM 如何释放其占用的堆外内存呢？如何跟 Cleaner 关联起来呢？

这得从 Cleaner 继承了 PhantomReference（虚引用） 说起。说到 Reference，还有 SoftReference、WeakReference、FinalReference 他们作用各不相同，这里就不展开说了。

简单介绍 PhantomReference，首先虚引用是不会影响 JVM 去回收其指向的对象，当 GC 某个对象时，如果有此对象上还有虚引用对其引用，会将 PhantomReference 对象插入 ReferenceQueue 队列。

PhantomReference插入到哪个队列呢？
看 PhantomReference 类代码，其继承自 Reference，Reference 对象有个 ReferenceQueue 成员，这个也就是 PhantomReference 对象插入的 ReferenceQueue 队列，此成员如果不由外部传入就是 ReferenceQueue.NULL。如果需要通过 queue 拿到 PhantomReference 对象，这个 ReferenceQueue 对象还是必须由外部传入。

```java
private static final ReferenceQueue dummyQueue = new ReferenceQueue();

private Cleaner(Object var1, Runnable var2) {
        super(var1, dummyQueue);
        this.thunk = var2;
}

```

```java
public class PhantomReference<T> extends Reference<T> {
```
Reference 类内部 static 静态块会启动 ReferenceHandler 线程，线程优先级很高，这个线程是用来处理 JVM 在 GC 过程中交接过来的 reference。想必经常用 jstack 命令，看线程堆栈的同学应该见到过这个线程。

```java

public abstract class Reference<T> {

   
    private T referent;         /* Treated specially by GC */

    ReferenceQueue<? super T> queue;

    Reference next;
    transient private Reference<T> discovered;  /* used by VM */

    static private class Lock { };
    private static Lock lock = new Lock();
    private static Reference pending = null;
    
    ...

    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();
    }

    public T get() {
        return this.referent;
    }

    public void clear() {
        this.referent = null;
    }

    public boolean isEnqueued() {
            synchronized (this) {
            return (this.queue != ReferenceQueue.NULL) && (this.next != null);
        }
    }

  
    public boolean enqueue() {
        return this.queue.enqueue(this);
    }


    /* -- Constructors -- */

    Reference(T referent) {
        this(referent, null);
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }

}

```
我们来看看 ReferenceHandler 是如何处理的？
直接看 run 方法，首先是个死循环，一直在那不停的干活，synchronized 块内的这段主要是交接 JVM 扔过来的 reference（就是 pending），再往下看，很明显，调用了 cleaner 的 clean 方法。调完之后直接 continue 结束此次循环，这个 reference 并没有进入 queue，也就是说 Cleaner 虚引用是不放入 ReferenceQueue。


```java
/* High-priority thread to enqueue pending References
     */
    private static class ReferenceHandler extends Thread {

        ReferenceHandler(ThreadGroup g, String name) {
            super(g, name);
        }

        public void run() {
            for (;;) {

                Reference r;
                synchronized (lock) {
                    if (pending != null) {
                        r = pending;
                        Reference rn = r.next;
                        pending = (rn == r) ? null : rn;
                        r.next = r;
                    } else {
                        try {
                            lock.wait();
                        } catch (InterruptedException x) { }
                        continue;
                    }
                }

                // Fast path for cleaners
                if (r instanceof Cleaner) {
                    ((Cleaner)r).clean();
                    continue;
                }

                ReferenceQueue q = r.queue;
                if (q != ReferenceQueue.NULL) q.enqueue(r);
            }
        }
    }
```

这块有点想不通，既然不放入 ReferenceQueue，为什么 Cleaner 类还是初始化了这个 ReferenceQueue。

## 如何手动回收？
手动回收，就是由开发手动调用 DirectByteBuffer 的 cleaner 的 clean 方法来释放空间。由于 cleaner 是 private 反问权限，所以自然想到使用反射来实现。

```java
public static void clean(final ByteBuffer byteBuffer) {  
 if (byteBuffer.isDirect()) { 
        Field cleanerField = byteBuffer.getClass().getDeclaredField("cleaner");
        cleanerField.setAccessible(true);
        Cleaner cleaner = (Cleaner) cleanerField.get(byteBuffer);
        cleaner.clean();
    }
}
```

还有另一种方法，DirectByteBuffer 实现了 DirectBuffer 接口，这个接口有 cleaner 方法可以获取 cleaner 对象。

```java
public static void clean(final ByteBuffer byteBuffer) {  
    if (byteBuffer.isDirect()) {  
        ((DirectBuffer)byteBuffer).cleaner().clean();  
    }  
}
```
Netty 中的堆外内存池就是使用反射来实现手动回收方式进行回收的。

******
> 涤生的博客。

> 转载请注明原创出处，谢谢！

> 欢迎关注我的微信公众号：「涤生的博客」，获取更多技术分享。

![涤生-微信公共号](/img/main/officialAccount.jpg)



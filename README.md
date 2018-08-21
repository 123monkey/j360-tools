j360-tools Java知识点持续更新

[![License](https://img.shields.io/badge/license-Apache%202-4EB1BA.svg)](https://www.apache.org/licenses/LICENSE-2.0.html)
[![Build Status](https://travis-ci.org/xuminwlt/j360-tools.svg?branch=master)](https://travis-ci.org/xuminwlt/j360-tools)

## 基础篇

1. 在String中使用KMP算法计算部分操作
2. JMH验证
3. 线程池如何确定Queue的使用,依据是什么
4. 二叉堆实现有序队列
5. 解读ThreadLocal
6. Java中的排序场景,Collections.sort, TreeMap
7. 线程生命周期各状态在jstack中的解读
8. String.intern, Long, Integer等对象池在jvm中使用
9. finalize,phantomReference使用
10. 验证hash、一致性hash的分布
11. shutdown pool时的规则, shutdown() && shutdownnow()
12. 算法,sort/search/rate limit/sliding window
13. 索引,B-tree、invert index


## 提高篇

1. Innodb事务,分布式事务
2. 一致性hash分片、扩容与缩服
3. CAP
4. redis & cluster
5. JVM,Hotspot/JPDA/JVMTI相关 参考https://www.ibm.com/developerworks/cn/java/j-lo-jpda3/index.html
6. TSDB
7. elasticsearch


## 问题笔记

1. KMP算法+JMH验证

```
# Measurement: 2 iterations, 1 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Throughput, ops/time
# Benchmark: me.j360.tools.ToolsBenchmark.indexOfKMPLongText

...
Result "me.j360.tools.ToolsBenchmark.indexOfStringLongText":
  1171034.703 ops/ms


# Run complete. Total time: 00:00:18

Benchmark                              Mode  Cnt        Score   Error   Units
ToolsBenchmark.indexOfKMP             thrpt    2    22165.252          ops/ms
ToolsBenchmark.indexOfKMPLongText     thrpt    2     1343.742          ops/ms
ToolsBenchmark.indexOfString          thrpt    2  2957587.750          ops/ms
ToolsBenchmark.indexOfStringLongText  thrpt    2  1171034.703          ops/ms
```

3. 理解线程池,用好线程池

从使用到深入了解自定义线程池ThreadPoolExecutor,先了解最完整的构造方法参数,一共7个参数

```
/**
     * Creates a new {@code ThreadPoolExecutor} with the given initial
     * parameters.
     *
     * @param corePoolSize the number of threads to keep in the pool, even
     *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
     * @param maximumPoolSize the maximum number of threads to allow in the
     *        pool
     * @param keepAliveTime when the number of threads is greater than
     *        the core, this is the maximum time that excess idle threads
     *        will wait for new tasks before terminating.
     * @param unit the time unit for the {@code keepAliveTime} argument
     * @param workQueue the queue to use for holding tasks before they are
     *        executed.  This queue will hold only the {@code Runnable}
     *        tasks submitted by the {@code execute} method.
     * @param threadFactory the factory to use when the executor
     *        creates a new thread
     * @param handler the handler to use when execution is blocked
     *        because the thread bounds and queue capacities are reached
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue}
     *         or {@code threadFactory} or {@code handler} is null
     */
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
- 参数列表
 核心线程数:默认参照cpu的数量
 最大线程数:线程池所能容纳的最大线程数
 空闲存活时间:非核心线程的空闲存活时间
 时间单位
 队列实例:任务队列,其中常用的有三种队列，SynchronousQueue,LinkedBlockingDeque,ArrayBlockingQueue
 线程工厂:尽量自定义线程工厂,默认DefaultThreadFactory
 拒绝策略:默认丢弃,选择jdk自带的四选一策略。
 
- 内部属性
 ctl:记录每个线程的状态字段,使用29+3位的Integer类型记录,当数量更大时,未来会考虑使用AtomicLong类型
 

- 参数大小特别说明
    1. ** 当线程池小于corePoolSize时，新提交任务将创建一个新线程执行任务，即使此时线程池中存在空闲线程。 **
    2. 当线程池达到corePoolSize时，新提交任务将被放入workQueue中，等待线程池中任务调度执行 
    3. 当workQueue已满，且maximumPoolSize>corePoolSize时，新提交任务会创建新线程执行任务 
    4. 当提交任务数超过maximumPoolSize时，新提交任务由RejectedExecutionHandler处理 
    5. 当线程池中超过corePoolSize线程，空闲时间达到keepAliveTime时，关闭空闲线程 
    6. 当设置allowCoreThreadTimeOut(true)时，线程池中corePoolSize线程空闲时间达到keepAliveTime也将关闭

- 线程池生命周期
 1. 创建:构造方法,核心线程此时还是空
 2. 运行:提交任务,根据线程数量等情况创建线程或拒绝
 3. 停止:shutdown()->拒绝新提交线程->阻塞一直等待线程池中线程+队列中任务全部完成->销毁线程池,
    shutdownnow()->尝试停止正在运行的线程,丢弃当前等待的所有线程和队列任务,直接销毁线程池,当尝试停止的线程忽略interrupt操作时,可能永远无法真正停止
 
- gc相关

```
/**
     * Invokes {@code shutdown} when this executor is no longer
     * referenced and it has no threads.
     */
    protected void finalize() {
        shutdown();
    }
```

这里覆写了finalize方法,在垃圾回收时还会再次进行shutdown操作,具体参见《9. Finalize》

延伸:ForkJoinPool 1.7+

ForkJoinPool使用了 @sun.misc.Contended 注解,表示此处使用了CPU缓存管理的行填充方式
ForkJoinPool使用work-stealing工作方式,由事件驱动,并支持1.8+lambda函数表达式
在JDK异步执行的某些多态api中,如未使用线程池参数,默认使用ForkJoinPool.commonPool作为默认的线程池,该默认线程池使用jvm参数配置方式进行部分自定义操作

完整的构造方法
```
/**
     * Creates a {@code ForkJoinPool} with the given parameters.
     *
     * @param parallelism the parallelism level. For default value,
     * use {@link java.lang.Runtime#availableProcessors}.
     * @param factory the factory for creating new threads. For default value,
     * use {@link #defaultForkJoinWorkerThreadFactory}.
     * @param handler the handler for internal worker threads that
     * terminate due to unrecoverable errors encountered while executing
     * tasks. For default value, use {@code null}.
     * @param asyncMode if true,
     * establishes local first-in-first-out scheduling mode for forked
     * tasks that are never joined. This mode may be more appropriate
     * than default locally stack-based mode in applications in which
     * worker threads only process event-style asynchronous tasks.
     * For default value, use {@code false}.
     * @throws IllegalArgumentException if parallelism less than or
     *         equal to zero, or greater than implementation limit
     * @throws NullPointerException if the factory is null
     * @throws SecurityException if a security manager exists and
     *         the caller is not permitted to modify threads
     *         because it does not hold {@link
     *         java.lang.RuntimePermission}{@code ("modifyThread")}
     */
    public ForkJoinPool(int parallelism,
                        ForkJoinWorkerThreadFactory factory,
                        UncaughtExceptionHandler handler,
                        boolean asyncMode) {
        this(checkParallelism(parallelism),
             checkFactory(factory),
             handler,
             asyncMode ? FIFO_QUEUE : LIFO_QUEUE,
             "ForkJoinPool-" + nextPoolId() + "-worker-");
        checkPermission();
    }
```

4. 二叉堆

二叉堆是有序队列场景常用的数据结构,Java的有序队列使用的就是二叉堆PriorityQueue

>
    SortedList使用常规的归并排序进行sort操作(mergeSort),同时也是Collections.sort操作的排序方式
    Array.sort使用TimSort进行排序,除非用户指定使用mergeSort
    TimSort,来源于Python,使用优化过的二分插入排序

5. ThreadLocal

ThreadLocal是本地线程对象,在几乎所有的中间件框架中都有定义,通常用于存取当前线程的一些本地状态,比如slf4j中的MDC,Trace中的Tracer,各种容器中的Context上下文对象
ThreadLocal是如何做到的?

ThreadLocal只有一个构造方法new ThreadLocal(),泛型用于指定该ThreadLocal存储的对象类型,常用方法get(),set()
 **面试的大部分都知道但是细节都说错了,注意key并不是当前线程,getMap(t)仅仅是从当前线程中拿到ThreadLocalMap**
ThreadLocal使用一个Map对象存储所有当前线程的所有ThreadLocal对象,其中key=this->当前ThreadLocal,value=存储的对象<T>
当前线程对象中绑定了一个ThreadLocalMap对象称为threadLocals, 初始化时默认为空,当且仅当该线程为子线程需要继承父线程的threadLocal状态时会初始化inheritableThreadLocals
ThreadLocal在初始化时会去给当前Thread对象绑定上threadLocals,从而在任何地方都可以通过

```
Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
```

拿到ThreadLocalMap,从而可以拿到ThreadLocal对象

```
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
ThreadLocal对象既然可以贯穿整个线程生命周期,那会不会存在gc问题,一般情况下不需要特别考虑手动回收,但是极端情况下会存在gc问题
Thread对象是gc的root根节点,gc时根据可达性分析来判断是否对对象进行回收
当ThreadLocal中存储的对象未当前Thread中的对象时,发生gc时会回收掉所有相关对象,如果当前线程是线程池中的核心时,该线程并不会被销毁。

ThreadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal没有外部强引用来引用它，那么系统 GC 的时候，这个ThreadLocal势必会被回收，这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value永远无法回收，造成内存泄漏。
其实，ThreadLocalMap的设计中已经考虑到这种情况，也加上了一些防护措施：在ThreadLocal的get(),set(),remove()的时候都会清除线程ThreadLocalMap里所有key为null的value。

但是这些被动的预防措施并不能保证不会内存泄漏：

 1. 使用static的ThreadLocal，延长了ThreadLocal的生命周期，可能导致的内存泄漏（参考ThreadLocal 内存泄露的实例分析）。
 2. 分配使用了ThreadLocal又不再调用get(),set(),remove()方法，那么就会导致内存泄漏。

6. 排序相关

Collections.sort:参考4
TreeMap/TreeSet中的Tree是红黑树的实现的排序方式,参考红黑树原理,红黑树在1.8+ HashMap/ConcurrentHashMap中的某些条件下链表会转化成红黑树,提升性能

7. jstack
   
   - 1. synchronize锁下拿到锁的线程占据了cpu控制权,其他等待锁的block阻塞
   - 2. 在1情况下的线程如果执行wait或者wait(time),则有条件出让锁和cpu资源,竞争2线程拿到锁,,1线程wait状态,等待2线程notify
   - 3. 1线程情况下,如果线程sleep仅仅是线程sleep,进入wait状态,但是不出让cpu资源,等待sleep结束自我唤醒
   - 4. wait、wait_time发生的情况有2中,1自己拿到锁,自己睡眠sleep,2自己拿到锁,自己wait
   - 5. 其他线程执行完并notify时,wait线程收到通知进入竞争状态,生产者消费者场景是该问题排查的常用场景
   
   ```
   min-xufpdeMacBook-Pro% jstack 73952
   2018-07-05 14:41:24
   Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.101-b13 mixed mode):
   
   "Attach Listener" #14 daemon prio=9 os_prio=31 tid=0x00007f838a061800 nid=0xd07 waiting on condition [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "DestroyJavaVM" #13 prio=5 os_prio=31 tid=0x00007f83890a0800 nid=0x2603 waiting on condition [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "pool-1-thread-3" #12 prio=5 os_prio=31 tid=0x00007f83890a0000 nid=0x3b03 waiting on condition [0x0000700003fb9000]
      java.lang.Thread.State: TIMED_WAITING (sleeping)
           at java.lang.Thread.sleep(Native Method)
           at me.j360.tools.ref.JstackCase$Task3.run(JstackCase.java:63)
           - locked <0x00000007956f70f0> (a java.lang.Object)
           at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
           at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
           at java.lang.Thread.run(Thread.java:745)
   
   "pool-1-thread-2" #11 prio=5 os_prio=31 tid=0x00007f838a060800 nid=0x3e03 waiting for monitor entry [0x0000700003eb6000]
      java.lang.Thread.State: BLOCKED (on object monitor)
           at java.lang.Object.wait(Native Method)
           - waiting on <0x00000007956f70f0> (a java.lang.Object)
           at me.j360.tools.ref.JstackCase$Task.run(JstackCase.java:39)
           - locked <0x00000007956f70f0> (a java.lang.Object)
           at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
           at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
           at java.lang.Thread.run(Thread.java:745)
   
   "pool-1-thread-1" #10 prio=5 os_prio=31 tid=0x00007f838a02b000 nid=0x3a03 waiting for monitor entry [0x0000700003db3000]
      java.lang.Thread.State: BLOCKED (on object monitor)
           at java.lang.Object.wait(Native Method)
           - waiting on <0x00000007956f70f0> (a java.lang.Object)
           at me.j360.tools.ref.JstackCase$Task.run(JstackCase.java:39)
           - locked <0x00000007956f70f0> (a java.lang.Object)
           at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
           at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
           at java.lang.Thread.run(Thread.java:745)
   
   "Monitor Ctrl-Break" #9 daemon prio=5 os_prio=31 tid=0x00007f8389897000 nid=0x3903 runnable [0x0000700003cb0000]
      java.lang.Thread.State: RUNNABLE
           at java.net.PlainSocketImpl.socketAccept(Native Method)
           at java.net.AbstractPlainSocketImpl.accept(AbstractPlainSocketImpl.java:409)
           at java.net.ServerSocket.implAccept(ServerSocket.java:545)
           at java.net.ServerSocket.accept(ServerSocket.java:513)
           at com.intellij.rt.execution.application.AppMain$1.run(AppMain.java:90)
           at java.lang.Thread.run(Thread.java:745)
   
   "Service Thread" #8 daemon prio=9 os_prio=31 tid=0x00007f838a844000 nid=0x3803 runnable [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "C1 CompilerThread2" #7 daemon prio=9 os_prio=31 tid=0x00007f838a058000 nid=0x3703 waiting on condition [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "C2 CompilerThread1" #6 daemon prio=9 os_prio=31 tid=0x00007f838a057000 nid=0x3503 waiting on condition [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "C2 CompilerThread0" #5 daemon prio=9 os_prio=31 tid=0x00007f838a055800 nid=0x4603 waiting on condition [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "Signal Dispatcher" #4 daemon prio=9 os_prio=31 tid=0x00007f838a055000 nid=0x3303 runnable [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "Finalizer" #3 daemon prio=8 os_prio=31 tid=0x00007f838901a000 nid=0x4f03 in Object.wait() [0x000070000359b000]
      java.lang.Thread.State: WAITING (on object monitor)
           at java.lang.Object.wait(Native Method)
           - waiting on <0x0000000795588ee0> (a java.lang.ref.ReferenceQueue$Lock)
           at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
           - locked <0x0000000795588ee0> (a java.lang.ref.ReferenceQueue$Lock)
           at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)
           at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)
   
   "Reference Handler" #2 daemon prio=10 os_prio=31 tid=0x00007f8389017800 nid=0x2d03 in Object.wait() [0x0000700003498000]
      java.lang.Thread.State: WAITING (on object monitor)
           at java.lang.Object.wait(Native Method)
           - waiting on <0x0000000795586b50> (a java.lang.ref.Reference$Lock)
           at java.lang.Object.wait(Object.java:502)
           at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
           - locked <0x0000000795586b50> (a java.lang.ref.Reference$Lock)
           at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)
   
   "VM Thread" os_prio=31 tid=0x00007f838a040800 nid=0x5203 runnable 
   
   "GC task thread#0 (ParallelGC)" os_prio=31 tid=0x00007f838981c000 nid=0x1d07 runnable 
   
   "GC task thread#1 (ParallelGC)" os_prio=31 tid=0x00007f838a000800 nid=0x1e03 runnable 
   
   "GC task thread#2 (ParallelGC)" os_prio=31 tid=0x00007f838a001000 nid=0x2b03 runnable 
   
   "GC task thread#3 (ParallelGC)" os_prio=31 tid=0x00007f838a002000 nid=0x5303 runnable 
   
   "VM Periodic Task Thread" os_prio=31 tid=0x00007f8389831000 nid=0x4203 waiting on condition 
   
   JNI global references: 21
   
   ```
   
8. 对象缓存池



9. Finalize

- RunFinalize.java
- PhantomReference: 在垃圾回收时收到一个系统通知

复写
- vm 
-verbose:gc  -Xloggc:gc_%p.log -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -XX:+PrintGCTimeStamps -XX:+PrintTenuringDistribution -XX:+PrintGCApplicationStoppedTime


只有在finalize中再次引用对象本身才会发生对象复活1次的行为,在执行System.gc时因为对象回收的不确定性,所以根据gc日子判断回收问题,并不能非常准确的描述出事件发生节点
结论如下:
对象复活的行为必须是对象本身在垃圾回收时,在finalize引用一次
同时再次执行垃圾回收时,该对象必被回收,这是回收时间点不确定
根据gc日志的节点只能作为参考,不能作为实验判断依据

- jdk7+ 终止方法中在try withresource使用场景,在finalize中实现非内存资源的释放, file/ inputstream/ socket等等
- Finalizer类和FinalizerThread最终实现垃圾回收的工作,并因为其优先级的原因,无法做到准确的预测时间
- 虚拟引用比其他引用可以更安全的重写finalize导致的问题,作为垃圾回收清理时的通知机制的特殊需求
参考:
https://www.jianshu.com/p/9d2788fffd5f
http://www.cnblogs.com/jqyp/archive/2010/11/27/1889414.html
http://zhang-xzhi-xjtu.iteye.com/blog/413159


```
Java HotSpot(TM) 64-Bit Server VM (25.101-b13) for bsd-amd64 JRE (1.8.0_101-b13), built on Jun 22 2016 02:42:15 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 8388608k(104744k free)

/proc/meminfo:

CommandLine flags: -XX:InitialHeapSize=134217728 -XX:MaxHeapSize=2147483648 -XX:+PrintGC -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC -XX:+PrintTenuringDistribution -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC 
{Heap before GC invocations=1 (full 0):
 PSYoungGen      total 38400K, used 2662K [0x0000000795580000, 0x0000000798000000, 0x00000007c0000000)
  eden space 33280K, 8% used [0x0000000795580000,0x0000000795819a28,0x0000000797600000)
  from space 5120K, 0% used [0x0000000797b00000,0x0000000797b00000,0x0000000798000000)
  to   space 5120K, 0% used [0x0000000797600000,0x0000000797600000,0x0000000797b00000)
 ParOldGen       total 190464K, used 102400K [0x0000000740000000, 0x000000074ba00000, 0x0000000795580000)
  object space 190464K, 53% used [0x0000000740000000,0x0000000746400010,0x000000074ba00000)
 Metaspace       used 2998K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 331K, capacity 388K, committed 512K, reserved 1048576K
2018-07-05T13:28:43.683-0800: 0.382: [GC (System.gc()) 
Desired survivor size 5242880 bytes, new threshold 7 (max 15)
[PSYoungGen: 2662K->496K(38400K)] 105062K->102904K(228864K), 0.0023376 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap after GC invocations=1 (full 0):
 PSYoungGen      total 38400K, used 496K [0x0000000795580000, 0x0000000798000000, 0x00000007c0000000)
  eden space 33280K, 0% used [0x0000000795580000,0x0000000795580000,0x0000000797600000)
  from space 5120K, 9% used [0x0000000797600000,0x000000079767c010,0x0000000797b00000)
  to   space 5120K, 0% used [0x0000000797b00000,0x0000000797b00000,0x0000000798000000)
 ParOldGen       total 190464K, used 102408K [0x0000000740000000, 0x000000074ba00000, 0x0000000795580000)
  object space 190464K, 53% used [0x0000000740000000,0x0000000746402010,0x000000074ba00000)
 Metaspace       used 2998K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 331K, capacity 388K, committed 512K, reserved 1048576K
}
{Heap before GC invocations=2 (full 1):
 PSYoungGen      total 38400K, used 496K [0x0000000795580000, 0x0000000798000000, 0x00000007c0000000)
  eden space 33280K, 0% used [0x0000000795580000,0x0000000795580000,0x0000000797600000)
  from space 5120K, 9% used [0x0000000797600000,0x000000079767c010,0x0000000797b00000)
  to   space 5120K, 0% used [0x0000000797b00000,0x0000000797b00000,0x0000000798000000)
 ParOldGen       total 190464K, used 102408K [0x0000000740000000, 0x000000074ba00000, 0x0000000795580000)
  object space 190464K, 53% used [0x0000000740000000,0x0000000746402010,0x000000074ba00000)
 Metaspace       used 2998K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 331K, capacity 388K, committed 512K, reserved 1048576K
2018-07-05T13:28:43.685-0800: 0.384: [Full GC (System.gc()) [PSYoungGen: 496K->0K(38400K)] [ParOldGen: 102408K->102800K(190464K)] 102904K->102800K(228864K), [Metaspace: 2998K->2998K(1056768K)], 0.0080959 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
Heap after GC invocations=2 (full 1):
 PSYoungGen      total 38400K, used 0K [0x0000000795580000, 0x0000000798000000, 0x00000007c0000000)
  eden space 33280K, 0% used [0x0000000795580000,0x0000000795580000,0x0000000797600000)
  from space 5120K, 0% used [0x0000000797600000,0x0000000797600000,0x0000000797b00000)
  to   space 5120K, 0% used [0x0000000797b00000,0x0000000797b00000,0x0000000798000000)
 ParOldGen       total 190464K, used 102800K [0x0000000740000000, 0x000000074ba00000, 0x0000000795580000)
  object space 190464K, 53% used [0x0000000740000000,0x0000000746464098,0x000000074ba00000)
 Metaspace       used 2998K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 331K, capacity 388K, committed 512K, reserved 1048576K
}

.... 

2018-07-05T13:28:44.714-0800: 1.412: Total time for which application threads were stopped: 0.0071747 seconds, Stopping threads took: 0.0000322 seconds
Heap
 PSYoungGen      total 38400K, used 665K [0x0000000795580000, 0x0000000798000000, 0x00000007c0000000)
  eden space 33280K, 2% used [0x0000000795580000,0x00000007956267d8,0x0000000797600000)
  from space 5120K, 0% used [0x0000000797b00000,0x0000000797b00000,0x0000000798000000)
  to   space 5120K, 0% used [0x0000000797600000,0x0000000797600000,0x0000000797b00000)
 ParOldGen       total 190464K, used 379K [0x0000000740000000, 0x000000074ba00000, 0x0000000795580000)
  object space 190464K, 0% used [0x0000000740000000,0x000000074005ef78,0x000000074ba00000)
 Metaspace       used 3006K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 332K, capacity 388K, committed 512K, reserved 1048576K

```




12. 海量处理处理结构

分而治之/hash映射 + hash统计 + 堆/快速/归并排序；
双层桶划分
Bloom filter/Bitmap；
Trie树(字典树)/数据库/倒排索引；
外排序；
分布式处理之Hadoop/Mapreduce。
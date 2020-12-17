---
title: "Java线程池"
date: 2020-12-16T12:55:05+08:00
draft: false
original: true
categories: 
  - Java
tags: 
  - 线程池
---

#### 线程池是什么？

线程池用于多线程处理中，它可以根据系统的情况，可以有效控制线程执行的数量，优化运行效果。线程池做的工作主要是控制运行的线程的数量，处理过程中将任务放入队列，然后在线程创建后启动这些任务，如果线程数量超过了最大数量超出数量的线程排队等候，等其它线程执行完毕，再从队列中取出任务来执行。

<!--more-->

#### 线程池的作用

在面向对象的编程过程中，创建对象和销毁对象是非常消耗时间和资源的。因此想要最小化这种消耗的一种思想就是『池化资源』。线程池就是这样的一种思想。我们通过重用线程池中的资源来减少创建和销毁线程所需要耗费的时间和资源。

线程池的一个作用是创建和销毁线程的次数，每个工作线程可以多次使用；另一个作用是可根据系统情况调整执行的线程数量，防止消耗过多内存。另外，通过线程池，能有效的控制线程的最大并发数，提高系统资源利用率，同时避免过多的资源竞争，避免堵塞。

线程池的优点总结如下几个方面：

*   线程复用
*   控制最大并发数
*   管理线程

#### 线程池的组成

一般的线程池主要分为以下4个组成部分：

1.  线程池管理器：用于创建并管理线程池
2.  工作线程：线程池中的线程
3.  任务接口：每个任务必须实现的接口，用于工作线程调度其运行
4.  任务队列：用于存放待处理的任务，提供一种缓冲机制

#### 线程池的常见应用场景

许多服务器应用常常需要处理大量而短小的请求（例如，Web 服务器，数据库服务器等等），通常它们收到的请求数量很大，一个简单的模型是，当服务器收到来自远程的请求时，为每一个请求开启一个线程，在请求完毕之后再对线程进行销毁。这样处理带来的问题是，创建和销毁线程所消耗的时间往往比任务本身所需消耗的资源要大得多。那么应该怎么办呢？

线程池为线程生命周期开销问题和资源不足问题提供了解决方案。我们可以通过线程池做到线程复用，不需要频繁的创建和销毁线程，让线程池中的线程一直存在于线程池中，然后线程从任务队列中取得任务来执行。而且这样做的另一个好处有，通过适当地调整线程池中的线程数目，也就是当请求的数目超过某个阈值时，就强制其它任何新到的请求一直等待，直到获得一个线程来处理为止，从而可以防止资源不足。

#### Java线程池的简介

Java中提供了实现线程池的框架Executor，并且提供了许多种类的线程池，接下来的文章中将会做详细介绍。

#### Java线程池框架

Java中的线程池是通过Executor框架实现的，该框架中用到了Executor，Executors，ExecutorService，ThreadPoolExecutor ，Callable和Future、FutureTask这几个类。

![Java线程池框架](/Java线程池/p798.jpg)

*   Executor：所有线程池的接口，只有一个方法
*   Executors：Executor 的工厂类，提供了创建各种不同线程池的方法，返回的线程池都实现了ExecutorService 接口
*   ThreadPoolExecutor：线程池的具体实现类，一般所有的线程池都是基于这个类实现的

其中ThreadPoolExecutor的构造方法如下：

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
}
```

其中：

*   corePoolSize：线程池的核心线程数
*   maximumPoolSize：线程池中允许的最大线程数
*   keepAliveTime：空闲线程结束的超时时间
*   unit：是一个枚举，它表示的是 keepAliveTime 的单位
*   workQueue：工作队列，用于任务的存放

#### Java线程池的工作过程

Java线程池的工作过程如下：

1.  线程池刚创建时，里面没有一个线程。任务队列是作为参数传进来的。不过，就算队列里面有任务，线程池也不会马上执行它们。
2.  当调用 execute() 方法添加一个任务时，线程池会做如下判断：
    *   如果正在运行的线程数量小于 corePoolSize，那么马上创建线程运行这个任务；
    *   如果正在运行的线程数量大于或等于 corePoolSize，那么将这个任务放入队列；
    *   如果这时候队列满了，而且正在运行的线程数量小于 maximumPoolSize，那么还是要创建非核心线程立刻运行这个任务；
    *   如果队列满了，而且正在运行的线程数量大于或等于 maximumPoolSize，那么线程池会抛出异常RejectExecutionException。
3.  当一个线程完成任务时，它会从队列中取下一个任务来执行。
4.  当一个线程无事可做，超过一定的时间（keepAliveTime）时，线程池会判断，如果当前运行的线程数大于 corePoolSize，那么这个线程就被停掉。所以线程池的所有任务完成后，它最终会收缩到 corePoolSize 的大小。

#### 常见的Java线程池

生成线程池使用的是Executors的工厂方法，以下是常见的 Java 线程池：

##### SingleThreadExecutor

SingleThreadExecutor是单个线程的线程池，即线程池中每次只有一个线程在运行，单线程串行执行任务。如果这个唯一的线程因为异常结束，那么会有一个新的线程来替代它。此线程池保证所有任务的执行顺序按照任务的提交顺序执行。

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

##### FixedThreadPool

FixedThreadPool是固定数量的线程池，只有核心线程，每提交一个任务就是一个线程，直到达到线程池的最大数量，然后后面进入等待队列，直到前面的任务完成才继续执行。线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

####  CachedThreadPool

CachedThreadPool是可缓存线程池。如果线程池的大小超过了处理任务所需要的线程，那么就会回收部分空闲（60秒不执行任务）的线程，当任务数增加时，此线程池又可以智能的添加新线程来处理任务。此线程池不会对线程池大小做限制，线程池大小完全依赖于操作系统（或者说JVM）能够创建的最大线程大小。其中，SynchronousQueue是一个是缓冲区为1的阻塞队列。

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

####  ScheduledThreadPool

ScheduledThreadPool是核心线程池固定，大小无限制的线程池，支持定时和周期性的执行线程。创建一个周期性执行任务的线程池。如果闲置,非核心线程池会在DEFAULT_KEEPALIVEMILLIS时间内回收。

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
```

####  Java 线程池的创建和使用

我们可以通过Executors的工厂方法来创建一个线程池。但是我们该如何让线程池执行任务呢？

线程池最常用的提交任务的方法有两种：

*   execute：

```java
ExecutorService.execute(Runnable runable)；
```

*   submit：

```java
FutureTask task = ExecutorService.submit(Runnable runnable);
FutureTask<T> task = ExecutorService.submit(Runnable runnable,T Result);
FutureTask<T> task = ExecutorService.submit(Callable<T> callable);
```

可以看出submit开启的是有返回结果的任务，会返回一个FutureTask对象，这样就能通过get()方法得到结果。submit最终调用的也是execute(Runnable runable)，submit只是将Callable对象或Runnable封装成一个FutureTask对象，因为FutureTask是个Runnable，所以可以在execute中执行。

下面的示例代码演示了如何创建一个线程池，并且使用它管理线程：

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " is running.");
    }
}
 
public class TestSingleThreadExecutor {
    public static void main(String[] args) {
        //创建一个可重用固定线程数的线程池
        ExecutorService pool = Executors.newFixedThreadPool(2);
        //创建实现了Runnable接口对象
        Thread tt1 = new MyThread();
        Thread tt2 = new MyThread();
        Thread tt3 = new MyThread();
        Thread tt4 = new MyThread();
        Thread tt5 = new MyThread();
        //将线程放入池中并执行
        pool.execute(tt1);
        pool.execute(tt2);
        pool.execute(tt3);
        pool.execute(tt4);
        pool.execute(tt5);
        //关闭
        pool.shutdown();
    }
}
```

运行结果：

```
pool-1-thread-1 is running.
pool-1-thread-2 is running.
pool-1-thread-1 is running.
pool-1-thread-2 is running.
pool-1-thread-1 is running.

```

### Java线程池原理

这篇文章会分别从这三个方面，结合具体的代码实现来剖析 Java 线程池的原理以及它的具体实现。

#### 线程复用

我们知道线程池的一个作用是创建和销毁线程的次数，每个工作线程可以多次使用。这个功能就是线程复用。想要了解 Java 线程池是如何进行线程复用的，我们首先需要了解线程的生命周期。

#### 线程生命周期

下图描述了线程完整的生命周期：

![线程生命周期](/Java线程池/p799.jpg)


在一个线程完整的生命周期中，它可能经历五种状态：新建（New）、就绪（Runnable）、运行（Running）、阻塞（Blocked）、终止（Zombie）。

在 Java中，Thread 通过new来新建一个线程，这个过程是是初始化一些线程信息，如线程名、id、线程所属group等，可以认为只是个普通的对象。调用Thread的start()后Java虚拟机会为其创建方法调用栈和程序计数器，同时将hasBeenStarted为true，之后如果再次调用start()方法就会有异常。

处于这个状态中的线程并没有开始运行，只是表示该线程可以运行了。至于该线程何时开始运行，取决于 JVM 里线程调度器的调度。当线程获取CPU后，run()方法会被调用。不要自己去调用Thread的run()方法。之后根据CPU的调度，线程就会在就绪—运行—阻塞间切换，直到run()方法结束或其他方式停止线程，进入终止状态。

因此，如果要实现线程的复用，我们必须要保证线程池中的线程保持存活状态（就绪、运行、阻塞）。接下来，我们就来看看ThreadPoolExecutor是如何实现线程复用的。

#### Worker 类

ThreadPoolExecutor主要是通过一个类来控制线程复用的：Worker 类。

我们来看一下简化后的 Worker 类代码：

```java
private final class Worker implements Runnable {
 
    final Thread thread;
 
    Runnable firstTask;
 
    Worker(Runnable firstTask) {
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }
 
    public void run() {
        runWorker(this);
    }
 
    final void runWorker(Worker w) {
        Runnable task = w.firstTask;
        w.firstTask = null;
        while (task != null || (task = getTask()) != null){
             task.run();
        }
    }
  ……
}
```

从代码中，我们可以看到 Worker 实现了 Runnable 接口，并且它还有一个 Thread成员变量 thread，这个 thread 就是要开启运行的线程。我们看到 Worker 的构造方法中传递了一个 Runnable 参数，同时它把自己作为参数传入 newThread()，这样的话，当 Thread 的start()方法得到调用时，执行的其实是 Worker 的run()方法，即runWorker()方法。

runWorker()方法之中有一个 while 循环，使用 getTask()来获取任务，并执行。接下来，我们将会看到getTask()是如何获取到 Runnable 对象的。

#### getTask()

我们来看一下简化后的getTask()代码：

```java
private Runnable getTask() {
  if(一些特殊情况) {
    return null;
  }
  Runnable r = workQueue.take();
  return r;
}
```

我们可以看到任务是从 workQueue中获取的，这个 workQueue 就是我们初始化 ThreadPoolExecutor 时存放任务的 BlockingQueue队列，这个队列里的存放的都是将要执行的 Runnable任务。因为 BlockingQueue 是个阻塞队列，BlockingQueue.take()返回的是空，则进入等待状态直到 BlockingQueue 有新的对象被加入时唤醒阻塞的线程。所以一般情况下，Thread的run()方法不会结束，而是不断执行workQueue里的Runnable任务，这就达到了线程复用的目的了。

#### 控制最大并发数

我们现在已经知道了 Java 线程池是如何做到线程复用的了，但是Runnable 是什么时候被放入 workQueue 队列中的呢，Worker里的Thread的又是什么时候调用start()开启新线程来执行Worker的run()方法的呢？从上面的分析中我们可以看出Worker里的runWorker()执行任务时是一个接一个，串行进行的，那并发是怎么体现的呢？它又是如何做到控制最大并发数的呢？

#### execute()

通过查看 execute()就能解答上述的一些问题，同样是简化后的代码：

```java
public void execute(Runnable command) {
    if (command == null) throw new NullPointerException();
    int c = ctl.get();
    // 当前线程数 < corePoolSize
    if (workerCountOf(c) < corePoolSize) {
        // 直接启动新的线程。
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 活动线程数 >= corePoolSize
    // runState为RUNNING && 队列未满
    if (isRunning(c) && workQueue.offer(command)) {
    int recheck = ctl.get();
    // 再次检验是否为RUNNING状态
    // 非RUNNING状态 则从workQueue中移除任务并拒绝
    if (!isRunning(recheck) && remove(command))
      reject(command);
    // 采用线程池指定的策略拒绝任务
    // 两种情况：
    // 1.非RUNNING状态拒绝新的任务
    // 2.队列满了启动新的线程失败（workCount > maximumPoolSize）
    } else if (!addWorker(command, false))
        reject(command);
}
```

####  addWorker()

我们再来看一下addWorker()的简化代码：

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    int wc = workerCountOf(c);
    if (wc >= (core ? corePoolSize : maximumPoolSize)) {
        return false;
    }
    w = new Worker(firstTask);
    final Thread t = w.thread;
    t.start();
}
```

根据上面的代码，线程池工作过程中是如何添加任务的就很清晰了：

*   如果正在运行的线程数量小于 corePoolSize，那么马上创建线程运行这个任务；
*   如果正在运行的线程数量大于或等于 corePoolSize，那么将这个任务放入队列；
*   如果这时候队列满了，而且正在运行的线程数量小于 maximumPoolSize，那么还是要创建非核心线程立刻运行这个任务；
*   如果队列满了，而且正在运行的线程数量大于或等于 maximumPoolSize，那么线程池会抛出异常RejectExecutionException

如果通过addWorker()成功创建新的线程，则通过start()开启新线程，同时将firstTask作为这个Worker里的run()中执行的第一个任务。虽然每个Worker的任务是串行处理，但如果创建了多个Worker，因为共用一个workQueue，所以就会并行处理了。所以可以根据corePoolSize和maximumPoolSize来控制最大并发数。

过程如下图所示：

![线程池执行过程](/Java线程池/p797.jpg)


#### 管理线程

上边的文章已经讲了，通过线程池可以很好的管理线程的复用，控制并发数，以及销毁等过程，而线程的管理过程已经穿插在其中了，也很好理解。

在 ThreadPoolExecutor 有个AtomicInteger变量 ctl，这一个变量保存了两个内容：

*   所有线程的数量
*   每个线程所处的状态

其中低29位存线程数，高3位存runState，通过位运算来得到不同的值。

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

//得到线程的状态
private static int runStateOf(int c) { return c & ~CAPACITY; }

//得到Worker的的数量
private static int workerCountOf(int c) { return c & CAPACITY; }

// 判断线程是否在运行
private static boolean isRunning(int c) { return c < SHUTDOWN; }
```

这里主要通过shutdown和shutdownNow()来分析线程池的关闭过程。首先线程池有五种状态来控制任务添加与执行。主要介绍以下三种：

*   RUNNING状态：线程池正常运行，可以接受新的任务并处理队列中的任务；
*   SHUTDOWN状态：不再接受新的任务，但是会执行队列中的任务；
*   STOP状态：不再接受新任务，不处理队列中的任务

shutdown()这个方法会将runState置为SHUTDOWN，会终止所有空闲的线程，而仍在工作的线程不受影响，所以队列中的任务人会被执行；shutdownNow()方法将runState置为STOP。和shutdown()方法的区别是，这个方法会终止所有的线程，所以队列中的任务也不会被执行了。

### Java线程池框架源码分析

前面的文章中已经给出了Java线程池框架中几个重要类的关系图:

![Java线程池框架](/Java线程池/p798.jpg)


现在我们基于这张图来逐步分析。

#### Executor

```
public interface Executor {
    void execute(Runnable command);
}
```

这个接口表示向线程池中提交一个任务。

#### ExecutorService

```
public interface ExecutorService extends Executor {
    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
       throws InterruptedException;
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                      long timeout, TimeUnit unit)
            throws InterruptedException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
                throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                                long timeout, TimeUnit unit)
                    throws InterruptedException, ExecutionException, TimeoutException;
}
```

可以看到 ExecutorService 扩展了 Executor 接口，在 Executor 基础上提供了更多的提交任务的方式和管理线程池的一些方法。

#### AbstractExecutorService

```java
public abstract class AbstractExecutorService implements ExecutorService {
 
}
```

#### ThreadPoolExecutor

ThreadPoolExecutor 是线程池框架的关键类. 首先来看一下 ThreadPoolExecutor 中几个重要的属性.

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    /**
     * 整个线程池的状态控制类 ctl, 是一个 AtomicInteger ，封装了下面2个部分:
     * workerCount 表示有效的线程数
     * runState 表示线程池当前状态，是running, shutdown还是其他状态
     *
     * runState表示了线程池的整个整个生命周期，可以取以下值:
     * RUNNING: 接受新的task，处理队列中的task
     * SHUTDOWN: 不接受新的 task 但会处理队列中的 task
     * STOP: 不接受新的 task, 不处理队列中的 task, 中断正在执行的 task
     * TIDYING: 所有 task 执行结束, workerCount 是 0, 线程过渡到TIDYING会调用 terminated()钩子方法
     * TERMINATED: terminated()执行完成
     *
     * 状态转换关系:
     *
     * RUNNING -> SHUTDOWN(调用shutdown())
     * On invocation of shutdown(), perhaps implicitly in finalize()
     * (RUNNING or SHUTDOWN) -> STOP(调用shutdownNow())
     * On invocation of shutdownNow()
     * SHUTDOWN -> TIDYING(queue和pool均empty)
     * When both queue and pool are empty
     * TIDYING -> TERMINATED(调用terminated())
     * When the terminated() hook method has completed
     */
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
 
    /**
     * task 的队列
     */
    private final BlockingQueue<Runnable> workQueue;
 
    /**
     * worker 线程的 Set, 只有持有 mainlock 时才能访问
     */
    private final ReentrantLock mainLock = new ReentrantLock();
    private final HashSet<Worker> workers = new HashSet<Worker>();
 
    private final Condition termination = mainLock.newCondition();
    private int largestPoolSize;
    private long completedTaskCount;
 
    private volatile ThreadFactory threadFactory;
    private volatile RejectedExecutionHandler handler;
 
    /**
     * idle 线程waiting for work的超时时间
     */
    private volatile long keepAliveTime;
    /**
     * 如果是 false,core threads将会保持 alive 即使处于 idel 状态
     * 如果是 true,core threads会keepAliveTime作为超时时间 wait for work
     */
    private volatile boolean allowCoreThreadTimeOut;
 
    private volatile int corePoolSize;
    private volatile int maximumPoolSize;
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();
}
```

workerCount, runState 使用一个 AtomicInteger 进行了封装, runState用 int 的高3位标书,低位表示 workerCount, 所以我们能看到 ThreadPoolExecutor 中和 ctl 相关的常量和解析方法.

```java
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
 
// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
 
// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

ThreadPoolExecutor 最主要的构造函数,设置上面说的重要属性.

```java
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

现在来看一下 execute() 方法, execute()方法有3个处理步骤:

1.  线程数小于 corePoolSize 时,则试图创建一个新的 worker 线程
2.  如果上面一步失败了，则试图将任务添加到阻塞队列中，并且要再一次判断需要不需要回滚队列，或者说创建线程
3.  如果上面两步都失败了，则会试图强行创建一个线程来执行这个任务，如果还是失败，扔掉这个任务

```
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    // 1.判断有效线程数是否小于核心线程数
    if (workerCountOf(c) < corePoolSize) {
        //创建新线程
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 2.分开来看，首先判断当前的池子是否是处于 running 状态
    // 因为只有 running 状态才可以接收新任务
    // 接下来判断能否成功添加到队列中，如果队列满了或者其他情况则会跳到下一步
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 再次检查池子的状态，如果进入了非 running 状态，回滚队列，扔掉这个任务
        if (! isRunning(recheck) && remove(command))
            reject(command);
        //如果处于 running 状态则检查当前的有效线程，如果没有则创建一个线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 3.前两步失败了，就强行创建线程，成功会返回true，如果失败扔掉这个任务
    else if (!addWorker(command, false))
        reject(command);
}
```

解释一下第二步，为什么要recheck

当这个任务被添加到了阻塞队列前，池子处于 RUNNING 状态，但如果在添加到队列成功后，池子进入了 SHUTDOWN 状态或者其他状态，这时候是不应该再接收新的任务的，所以需要把这个任务从队列中移除，并且 reject

同样，在没有添加到队列前，可能有一个有效线程，但添加完任务后，这个线程闲置超时或者因为异常被干掉了，这时候需要创建一个新的线程来执行任务

为了更直观的理解一个任务的执行过程，可以参考下面这张图

![任务的执行过程](/Java线程池/p796.jpg)

#### addWorker()

前一步把 execute 的流程捋了一遍，里面多次出现了 addWorker() 方法，前文说到这是个创建线程的方法，来看看 addWorker 做了些什么，这个方法代码比较长，我们拆开来一点一点看.

*   第一部分 — 判断各种基础异常

```
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
 
        // Check if queue empty only if necessary.
        // 检查线程池状态，队列状态，以及 firstask ，拆开来看
        // 这段代码看起来异常的蛋疼,转换一下逻辑即
        // rs>= SHUTDOWN && (rs != SHUTDOWN || firstTask != null ||workQueue.isEmpty())
        // 总结起来就是 当前处于非 Running 状态,并且这三种情况
        // 1. 不是处于 SHUTDOWN 状态，不能再创建线程
        // 2. 有新的任务 (因为不能再接收新的任务)
        // 3. 阻塞队列中已经没有任务 (不需要再创建线程)
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;
 
        for (;;) {
            //当前有效线程数目
            int wc = workerCountOf(c);
            // 根据传入的参数确定以核心线程数还是最大线程数作为判断条件
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                // 大于容量 或者指定的线程数，不允许创建
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
}
```

*   第二部分 — 试图创建线程

创建一个Worker

```java
boolean workerStarted = false; //标记 worker 开启状态
boolean workerAdded = false; //标记 worker 添加状态
Worker w = null;
try {
    w = new Worker(firstTask); //将这个任务作为 worker 的第一个任务传入
    final Thread t = w.thread; //通过 worker 获取到一个线程
    if (t != null) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // Recheck while holding lock.
            // Back out on ThreadFactory failure or if
            // shut down before lock acquired.
            int rs = runStateOf(ctl.get());
            // running状态，或者 shutdown 状态但是没有新的任务
            if (rs < SHUTDOWN ||
                (rs == SHUTDOWN && firstTask == null)) {
                if (t.isAlive()) // precheck that t is startable
                    throw new IllegalThreadStateException();
                // 将这个 worker 添加到线程池中
                workers.add(w);
                int s = workers.size();
                if (s > largestPoolSize)
                    largestPoolSize = s;
                // 标记worker添加成功
                workerAdded = true;
            }
        } finally {
            mainLock.unlock();
        }
        // 如果 worker 创建成功，开启线程
        if (workerAdded) {
            t.start();
            workerStarted = true;
        }
    }
} finally {
    if (! workerStarted)
        addWorkerFailed(w);
}
return workerStarted;
```

上面代码从逻辑层面来看不算难懂，到这里一个任务到达后，ThreadPoolExecutor 的处理就结束了，那么任务又是怎么被添加到阻塞队列中，线程是如何从队列中取出任务，上文中的 Worker 又是什么东西？

一个一个来，先来看看 Worker 到底是什么.

#### Worker

Worker 是 ThreadPoolExecutor 的一个内部类，实现了 Runnable 接口，继承自 AbstractQueuedSynchronizer,这又是个什么鬼？？? 这个就是经常见到的 AQS 的全称,这个暂时还没有研究.~~~~

```java
private final class Worker
       extends AbstractQueuedSynchronizer
       implements Runnable {
 
    final Thread thread;
    /** Initial task to run. Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;
}
```


简单来说，Worker实现了 lock 和 unLock 方法来标示当前线程的状态是否为闲置

```java
// Lock methods
//
// The value 0 represents the unlocked state.
// The value 1 represents the locked state.
public void lock()        { acquire(1); }
public boolean tryLock()  { return tryAcquire(1); }
public void unlock()      { release(1); }
public boolean isLocked() { return isHeldExclusively(); }
```

上一节创建线程成功后调用 t.start() 而这个线程又是 Worker 的成员变量

```java
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
```

可以看到这里将 Worker 作为 Runnable 参数创建了一个新的线程，我们知道 Thread 接收一个 Runnable 对象后 start 运行的是 Runnable 的 run 方法，Worker 的 run 方法调用了 runWorker ,这个方法里面就是取出任务执行的逻辑

```java
public void run() {
    runWorker(this);
}
 
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask; // 获取到 worker 的第一个任务
    w.firstTask = null;
    w.unlock(); // 标记为闲置，还没有开始任务 允许打断
    boolean completedAbruptly = true; // 异常退出标记
    try {
        // 循环取出任务，如果第一个任务不为空，或者从队列中拿到了任务
        // 只要这两个条件满足，会一直循环，直到没有任务，正常退出，或者异常退出
        while (task != null || (task = getTask()) != null) {
            w.lock();// 该线程标记为非闲置
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted. This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            // 翻译注释：1.如果线程池STOPPING状态，需要中断线程
            // 2.Thread.interrupted()是一个native方法，返回当前线程是否有被等待中断的请求
            // 3.第二个条件成立时，检查线程池状态，如果为STOP，并且没有被中断，则中断线程
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            // 执行任务
            try {
                beforeExecute(wt, task);// 执行前
                Throwable thrown = null;
                try {
                    task.run(); // 执行任务
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown); // 执行结束
                }
            } finally {
                task = null; // 将 worker 的任务置空
                w.completedTasks++;
                w.unlock(); // 释放锁，进入闲置状态
            }
        }// 循环结束
        completedAbruptly = false; // 标记为正常退出
    } finally {
      // 干掉 worker
        processWorkerExit(w, completedAbruptly);
    }
}
```

这里弄清楚了一件事情，进入循环准备执行任务时，worker 加锁标记为非闲置，任务执行完毕或者出现异常，worker 释放锁，进入闲置状态。

也就是当一个 worker 执行任务前或者执行完任务，到取出下一个任务期间，都是闲置状态可以被打断

上面取出任务调用了 getTask() ，诶～为什么有一个死循环，别着急，慢慢看来。上面的代码可以知道如果 getTask 返回任务则执行，如果返回为 null 则 worker 需要被回收

```java
private Runnable getTask() {
    // 标记取任务是否超时
    boolean timedOut = false; // Did the last poll() time out?
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        // Check if queue empty only if necessary.
        // 如果线程池状态为 STOP 或者 SHUTDOWN 并且队列已经为空，回收 wroker
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
        //获取当前有效线程数
        int wc = workerCountOf(c);
        // Are workers subject to culling?
        // timed 用来标记当前的 worker 是否设置超时时间，
        // 还记得获取线程池的时候 可以设置核心线程超时时间
        //1.允许核心线程超时回收(即所有线程) 2.当前有效线程超过核心线程数(需要回收)
        // 如果timed == false 则该worker不会被回收，如果没有取到任务 会一直阻塞
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
 
        // 回收线程条件
        // 1. 有效线程数已经大于了线程池的最大线程数或者设置了超时回收并且已经超时
        // 2. 有效线程数大于1或者队列任务已经为空
        // 只有当上面1和2 同时满足时 则试图回收线程
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
          // 如果减少workercount成功 直接回收
            if (compareAndDecrementWorkerCount(c))
                return null;
          // 否则重走循环，从第一个判断条件处回收
            continue;
        }
        // 取任务
        try {
          // 根据是否设置超时回收来选择不同的取任务的方式
          // poll 方法取任务会有超时时间，超过时间则返回null
          // take 方法没有超时时间，阻塞式方法
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
          // 如果任务不为空返回任务
            if (r != null)
                return r;
          // 否则标记超时 进入下一次循环等待回收
            timedOut = true;
        } catch (InterruptedException retry) {
          // 如果出现异常，试图重试
            timedOut = false;
        }
    }
}
```

getTask() 方法逻辑也捋得差不多了，这里又出现了两个新的方法，workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) 和 workQueue.take() ，这两个都是阻塞队列的方法，来看看它们又各自是怎么实现的

#### LinkedBlockingQueue — 阻塞队列

ThreadPoolExecutor 使用的是链表结构的阻塞队列，实现了 BlockingQueue 接口，而 BlockingQueue 则是继承自 Queue 接口，再上层就是 Collection 接口。

因为本篇笔记主要是分析 ThreadPoolExecutor 的原理，所以不会详细介绍 LinkedBlockingQueue 中的其它代码，主要介绍这里所用的方法，首先来看一下上文所提到的 take()

```java
public E take() throws InterruptedException {
    E x; // 任务
    int c = -1; // 取出任务后的剩余任务数量
    final AtomicInteger count = this.count; // 当前任务数量
    final ReentrantLock takeLock = this.takeLock; // 加锁防止并发
    takeLock.lockInterruptibly();
    try {
      // 如果队列数量为空，则一直循环，阻塞线程
        while (count.get() == 0) {
            notEmpty.await();
        }
      // 取出任务
        x = dequeue();
      // 任务数量减一
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();// 标记队列非空
    } finally {
        takeLock.unlock(); // 释放锁
    }
    if (c == capacity)
        signalNotFull();//标记队列已满
    return x;// 返回任务
}
```

上面的代码可以知道 take 方法会一直阻塞直到队列有新的任务为止

接下来是 poll 方法，可以看到几乎与 take 方法相同，唯一的区别是在阻塞的循环代码块里面加了时间判断，如果超时则直接返回为空，不会一直阻塞下去

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E x = null; // 存放的任务
    int c = -1;
    long nanos = unit.toNanos(timeout); // 超时时间
    final AtomicInteger count = this.count; // 队列中的数量
    final ReentrantLock takeLock = this.takeLock; // 加锁防止并发
    takeLock.lockInterruptibly();
    try {
        // 如果队列为空，则不断的循环
        while (count.get() == 0) {
            // 如果当倒计时小于0 即超时时间到 则返回空
            if (nanos <= 0)
                return null;
            // 让线程等待
            nanos = notEmpty.awaitNanos(nanos);
        }
        x = dequeue(); // 取出一个任务
        c = count.getAndDecrement(); // 取出后的队列数量
        if (c > 1)
            notEmpty.signal(); // 标记非空
    } finally {
        takeLock.unlock(); // 释放锁
    }
    if (c == capacity)
        signalNotFull(); // 标记队列已满
    return x; // 返回任务
}
```

#### 线程池的回收及终止

前一节分析了任务的执行流程及原理，也留下了一个问题，worker 是如何被回收的呢？线程池该如何管理呢？回到上一节的 runWorker() 方法中，还记得最后调用了一个方法

```java
processWorkerExit(w, completedAbruptly);
```

这个方法传入了两个参数，第一个是当前的 Woker ,第二个是标记异常退出的标识

首先判断是否为异常退出，如果是异常退出的话需要手动调整线程数量，如果是正常回收的，getTask 方法里面已经手动调整过了，不记得的小伙伴可以看看前文的代码，找找 decrementWorkerCount(),

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();
    // 加锁
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    // 记录线程池完成的任务总数，从 workers 中移除该 worker
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
 
    tryTerminate(); // 尝试关闭池子
    int c = ctl.get();
    // 以下的代码是判断需不需要给线程池创建一个新的线程
    // 如果线程池的状态是 RUNNING 或者 SHUTDOWN 进一步判断需不需要创建
    if (runStateLessThan(c, STOP)) {
        // 如果为异常退出直接创建，如果不是异常退出进入判断
        if (!completedAbruptly) {
            // 获取线程池应该存在的最小线程数 如果设置了超时 则是0，否则是核心线程数
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            // 如果 min 是0 但是队列又不为空，则 min 应该是1
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            //如果当前池中的有效线程数大于等于最小线程数 则不需要创建
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        // 创建线程
        addWorker(null, false);
    }
}
```

上面的代码中调用了 tryTerminate() 方法，这个方法是用于终止线程池的，又是一个 for 循环，从代码结构来看是异常情况的重试机制。还是老方法，慢慢来看总共做了几件事情

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
      // 如果处于这三种情况不需要关闭线程池
      // 1. Running 状态
      // 2. SHUTDOWN 状态并且任务队列不为空，不能终止
      // 3. TIDYING 或者 TERMINATE 状态，说明已经在关闭了 不需要重复关闭
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
        // 进入到关闭线程池的代码，如果线程池中还有线程，则需要打断线程
        if (workerCountOf(c) != 0) { // Eligible to terminate 可以关闭池子
            // 打断闲置线程，只打断一个
            interruptIdleWorkers(ONLY_ONE);
            return;
            // 如果有两个以上怎么办？只打断一个？
            // 这里只打断一个是因为 worker 回收的时候都会进入到该方法中来，可以回去再看看
            // runWorker方法最后的代码
        }
 
        // 线程已经回收完毕，准备关闭线程池
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();// 加锁
        try {
            // 将状态改变为 TIDYING 并且即将调用 terminated
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    terminated(); // 终止线程池
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0)); // 改变状态
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
            // 如果终止失败会重试
        }
        // else retry on failed CAS
    }
}
```

尝试终止线程池的代码分析完了，好像就结束了～但作为好奇宝宝，我们是不是应该看看如何打断闲置线程，以及 terminated 中做了什么呢？来吧，继续装逼

先来看打断线程

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();//加锁～
    try {
      // 遍历线程池中的 wroker
        for (Worker w : workers) {
            Thread t = w.thread;
          // 如果线程没有被中断，并且能够获取到 worker的锁(说明是闲置线程)
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();// 中断线程
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
          // 只中断一个 worker 跳出循环，否则会将所有的闲置线程都中断
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();// 释放锁
    }
```

有同学开始装逼了，说我们是好奇宝宝，t.interrupt() 方法也应该看，嗯～没错，但这里是调用了 native 方法，会 c 的可以去看看装逼，我就算了～

好了，再来看看 terminate, 是不是很坑爹？ terminated 里面神！马！也！没！干！。。。淡定，其实这个方法类似于 Activity 的生命周期方法，允许你在被终止时做一些事情，默认的线程池没有什么要做的事情，当然什么也没写啦～

```java
/**
 * Method invoked when the Executor has terminated. Default
 * implementation does nothing. Note: To properly nest multiple
 * overridings, subclasses should generally invoke
 * {@code super.terminated} within this method.
 */
protected void terminated() { }
```


#### 异常处理

还记得前面讲到，出现各种异常情况，添加队列失败等等，只是笼统的说了一句扔掉，当然代码实现不可能是简单一句扔掉就完了。回到 execute() 方法中找到 reject() 任务，看看究竟是怎么处理的

```java
final void reject(Runnable command) {
    handler.rejectedExecution(command, this);
}
```

还记得在创建线程池的时候，初始化了一个 handler — RejectedExecutionHandler

这是一个接口，只有一个方法,接收两个参数

```java
void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
```

既然是一个接口，那么肯定有他的实现类，我们先不急着看所有实现类，先来看看这里的 handler 可能是什么，记得在使用 Executors 获取线程池调用构造方法的时候并没有传入 handler 参数，那么 ThreadPoolExecutor 应该会有一个默认的 handler

```java
private static final RejectedExecutionHandler defaultHandler =
    new AbortPolicy();
public static class AbortPolicy implements RejectedExecutionHandler {
    public AbortPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                             " rejected from " +
                                             e.toString());
    }
}
```

默认 handler 是 AbortPolicy ,这个类实现了 rejectedExecution() 方法，抛了一个 Runtime 异常，也就是说当任务添加失败，就会抛出异常。这个类在 AsyncTask 引发了一场血案～所以在 API19 以后修改了 AsyncTask 的部分代码逻辑，这里就不细说啦.

实际上，在 ThreadPoolExecutor 中除了 AbortPolicy 外还实现了三种不同类型的 handler

*   CallerRunsPolicy — 在 线程池没有 shutdown 的前提下，会直接在执行 execute 方法的线程里执行这个任务

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {
        r.run();
    }
}
```

*   DiscardPolicy — 啥也不干，默默地丢掉任务～不信你看

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
}
```

*   DiscardOldestPolicy — 丢弃掉队列中未执行的，最老的任务，也就是任务队列排头的任务，然后再试图在执行一次

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {
        e.getQueue().poll();
        e.execute(r);
    }
}
```
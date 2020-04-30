---
title: ThreadPoolExecutor源码介绍
tags: [Java, JUC, 源码, 多线程, 线程池, ThreadPoolExecutor]
layout: post
author: wjf
---


# 目录

-   [介绍](#orgfaf4c4a)
-   [重要组件介绍](#org40ababf)
    -   [ReentrantLock mainLock](#org81eb00e)
    -   [Worker](#orgb335cfa)
    -   [BlockingQueue workQueue](#orgd6de845)
    -   [Condition termination](#org9885238)
    -   [ThreadFactory threadFactory](#orgf9c06f2)
    -   [RejectedExecutionHandler handler](#org4cdee23)
-   [execute 方法原理](#org3fc7eb0)
-   [改变线程池状态的方法](#org5010304)
-   [工作线程控制任务中断的方法](#org8e9f6d3)
-   [工作线程添加、运行和退出等方法](#org99df002)
-   [其他方法的源码](#org5f9d09c)



<a id="orgfaf4c4a"></a>

# 介绍

ThreadPoolExecutor 是是一种 ExecutorService（ExecutorService 的实现），内部维护了一个线程池，并利用线程池中的线程执行提交的任务。Executors 提供了许多固定配置的 ThreadPoolExecutor，快速提供 ThreadPoolExecutor。

ThreadPoolExecutor 提供线程池执行任务的目的是为了解决

1.  减少了为每个任务创建单独线程的开销，提升了执行大量任务的性能。
2.  提供用于管理线程、任务执行等一些列方法
3.  提供了基本的统计方法，比如完成了多少个任务等。

ThreadPoolExecutor 提供了许多可调节执行异步任务策略的参数和一些扩展钩子。虽然官网文档鼓励使用 Executors 提供的竞态方法创建线程池，但是面对具体的应用场景，可能不是最优的选择。尽量还是需要了解 ThreadPoolExecutor 各个参数做用，以便定制化配置线程池。

ThreadPoolExecutor 的线程数会在和核心线程数和最大线程数之间自动调节。如果线程数在少于核心线程数的情况下提交任务，那么不管是否存在其他空闲线程，线程池会创建一个线程来执行该任务；如果在线程数位于核心线程数和最大线程数之间的情况下提交任务，那么任务会先尝试进入阻塞队列，等待线程空闲出来按照顺序执行阻塞队列中的任务。如果阻塞队列已经达到容量上限，那么线程池会创建新的线程来执行该任务；如果在线程池的线程数已经达到最大线程数时，那么尝试进入阻塞队列等待执行。如果阻塞队列也达到上限，那么执行线程池初始化时选择的拒绝策略。如果设置的核心线程数和最大线程数是一样的值，那么相当于设置了一个固定线程数的线程池。核心线程数和最大线程数通常在初始化时就确定，但是可以通过提供的改变数值的方法(setCorePoolSize，setMaximumPoolSize)来修改。

ThreadPoolExecutor 默认是基于请求来创建线程的，即只有当请求来时才会创建线程。ThreadPoolExecutor 允许通过重写 prestartCoreThread 和 prestartAllCoreThreads 来实现初始化即创建线程。这个功能对于哪些配置了没有容量的阻塞队列会比较有用。

ThreadPoolExecutor 利用 ThreadFactory 来创建线程。如果没有做特殊配置，那么利用的是 Executors.defaultThreadFactory(ThreadFactory 是接口)来具体创建线程。Executors.defaultThreadFactory 创建的线程都在一个线程组（ThreadGroup）中，并且拥有同样的优先级别，并且都不是守护线程。可以自定义 ThreadFactory 来改变线程的名称，优先级别，守护线程状态等。如果 ThreadFactory 创建线程失败不会影响任务的提交，但是会出现没有线程执行任务的情况。线程应当要持久 modifyThread 的权限。如果线程没有该权限，服务会被降级，将会导致：配置改变后不会立即产生影响；关闭线程池时，线程池允许关闭但是不会立刻结束。

如果当前的线程数超过核心线程数并且存在线程处于空闲状态（没有执行任务），当空闲时间超过了 keepAliveTime，此类线程会被线程池回收。如果未来有同时超过核心线程数的任务，新的线程会被创建来执行这些任务。keepAliveTime(空闲时间)同样也允许在初始化之后被修改并生效。keepAliveTime 如果设置成 Long.MAX<sub>VALUE</sub> 会当成没有过期时间，空闲线程将不会被自动关闭。keepAliveTime 的策略默认只对超过核心线程数的空闲线程起作用，但是可以设置 allowCoreThreadTimeOut 来对空闲的核心线程起作用。

ThreadPoolExecutor 内部存在一个阻塞队列用于存放没有足够线程执行的任务。如果当前的线程数少于核心线程数的情况下提交任务时，ThreadPoolExecutor 会创建新线程执行该任务；如果执行任务的线程数已经大于核心线程数，那么任务会存到阻塞队列中。阻塞队列能够存储的任务数量取决于阻塞队列的容量大小。

ThreadPoolExecutor 使用的是 BlockingQueue 作为阻塞队列，BlockingQueue 有三个重要实现：

1.  SynchronousQueue: 直接传递。这是一个由提交任务的线程直接传递给消费任务的线程的传递方式。如果没有任何消费线程，那么向 SynchronousQueue 提交任务会失败。所以选择 SynchronousQueue 作为阻塞队列时，每次提交任务如果没有当前没有线程能够处理任务需要创建新线程来及时处理该任务。该策略可以避免如果有大量任务，并且任务内部有依赖关系而导致的锁问题（正在活跃线程中执行的任务依赖由于超过了核心线程数导致在阻塞队列中排队的任务，出现被依赖优先执行的任务没有线程执行，而执行中的任务由于被依赖任务没有执行完毕导致线程不释放）。SynchronousQueue 一般要求提供不设限制的容量，避免出现达到容量上限导致无法创建线程来处理任务而导致的任务提交失败。但是如果消费任务的速度少于任务提交的速度，SynchronousQueue 会出现由于创建的线程过多而导致的内存溢出问题。
2.  LinkedBlockingQueue：无界队列。队列不会在初始化时定义队列的容量限制，最大上限时 Integer.MAX<sub>VALUE</sub>。只要少于这个值，队列将不会拒绝将任务添加到队列中。如果使用 LinkedBlockingQueue，由于一般很能将任务加到 Integer.MAX<sub>VALUE</sub>，所以会出现只要线程数达到核心线程数，线程将不会被创建（任务都被存到 LinkedBlockingQueue 中）。LinkedBlockingQueue 适合任务处理非常快的场景，但是如果任务处理不及时，和可能会导致任务大量在 LinkedBlockingQueue 中堆积，导致大量超时。由于 LinkedBlockingQueue 很难达到队列上限，所以也会导致线程池的最大线程数的配置不生效。
3.  ArrayBlockingQueue：有界队列。初始化时需要指定队列的容量限制。有界队列可以帮助避免资源耗尽，但是也更加难以控制。有界队列的容量限制和线程池线程数的配置会相互影响。如果配置了一个大容量的阻塞队列和比较小的线程数上限，可以减少线程上下文切换的开销提高 CPU 的使用效率，但是可能会降低请求的吞吐量。而且如果线程数过少并且过多的阻塞在 IO 操作等等待或者耗时比较长的任务上，那么会导致大量任务没有线程执行堆积在阻塞队列上，长时间得不到执行甚至会导致请求方等待超时。如果使用一个比较小的队列配置一个较大的线程数上限，可以提高 CPU 的效率。但是如果频繁的在多个线程之间来回切换也可能会降低吞吐量。

ThreadPoolExecutor 在队列已经满了并且线程已经全部执行并且线程数已经到达限制的上限或者已经关闭 ThreadPoolExecutor 对任何提交的任务都会执行拒绝策略。默认使用 AbortPolicy 拒绝策略。

1.  AbortPolicy：该策略抛出 RejectedExecutionException 异常
2.  CallerRunsPolicy：该策略会让提交任务的线程计算提交的任务。
3.  DiscardPolicy：直接丢弃任务。
4.  DiscardOldestPolicy：丢弃队列第一个任务，重新执行提交逻辑。不断重复以上步骤知道提交成功。

如果以上的拒绝策略不满足业务需求，可以自定义一个拒绝策略。但是在设计拒绝策略是在 ThreadPoolExecutor 达到拒绝条件下才会被触发。

ThreadPoolExecutor 提供了一些钩子方法（某种场景下触发，如果有特殊需求可以重写这些钩子方法达到业务目的），比如 beforeExecute 和 afterExecute。他们分别在任务执行前和任务执行结束后执行。如果需要统计、日志等可以重写这俩方法实现。如果 ThreadPoolExecutor 关闭需要做二外的事情可以重写 terminated 方法来实现。

如果钩子方法，比如 afterExecute，执行失败，虽然具体的任务内容已经执行，但是总体上该任务执行失败。

ThreadPoolExecutor 如果不被手动关闭，那么默认情况下 ThreadPoolExecutor 之前创建过的不多于核心线程数的线程。如果担心忘记手动关闭而导致此类资源无法回收，可以设置 allowCoreThreadTimeOut，那么核心线程数如果空闲时间超过 getKeepAliveTime，线程池也会将此类线程资源回收。


<a id="org40ababf"></a>

# 重要组件介绍


<a id="org81eb00e"></a>

## ReentrantLock mainLock

ThreadPoolExecutor 内部持有一个 ReentrantLock，为关闭 ThreadPoolExecutor，等待关闭 ThreadPoolExecutor，中断空闲线程，添加工作线程等需要保证线程安全的操作提供同步执行的保证。


<a id="orgb335cfa"></a>

## Worker

Worker 对象表示一个工作线程。Worker 本身继承自 AbstractQueuedSynchronizer（同步器的实现以及大部分 JUC 锁的父类），这表示 Worker 本身可以绑定一个线程（exclusiveOwnerThread 属性）；内部维护了一个单向链表可以按照某个策略存储一系列任务；天然支持锁的机制。

Worker 在执行任务前后会上锁和解锁的操作。利用 AbstractQueuedSynchronizer 的上锁和释放锁而不是使用 ReentrantLock 做上锁和释放锁的原因是：Worker 在等待任务执行结束时被终止。如果其他线程调用 setCorePoolSize，那么会执行一些列操作 setCorePoolSize->interruptIdleWorkers->tryLock() (here success!) -> Thread.interrupt (thread of this worker)。所以 Worker 不能和 ReentrantLock 公用一把锁（[Why Worker extends AbstractQueuedSynchronizer](https://stackoverflow.com/questions/42189195/why-threadpoolexecutorworker-extends-abstractqueuedsynchronizer)）。也因此 Worker 被设计为是独占模式（一次只能一个线程持有）非重入锁。

```java
    private final class Worker
          extends AbstractQueuedSynchronizer
          implements Runnable
      {
          /**
           * This class will never be serialized, but we provide a
           * serialVersionUID to suppress a javac warning.
           */
          private static final long serialVersionUID = 6138294804551838833L;
    
          // 具体的工作线程。如果线程工厂创建线程失败则指向 null
          final Thread thread;
          // 初始化时指定的任务。一般情况下初始化时指定的是 null
          Runnable firstTask;
          // 执行过的任务数量
          volatile long completedTasks;
    
          /**
           * 构造方法。根据指定的任务初始化 Worker。
           * @param firstTask 第一个任务。如果没有则 null
           */
          Worker(Runnable firstTask) {
              // 初始化时将 state 的值设置为-1，如此除非调用释放锁，否则无法获取锁。如此该 Worker 无法被直接中断。
              setState(-1);
              this.firstTask = firstTask;
              this.thread = getThreadFactory().newThread(this);
          }
    
          // 执行任务。调用 ThreadPoolExecutor.runWorker
          public void run() {
              runWorker(this);
          }
    
          // 配置成为排它锁
          protected boolean isHeldExclusively() {
              return getState() != 0;
          }
    
          // 获取锁
          protected boolean tryAcquire(int unused) {
              // CAS 比较 state 的值，如果能够将其从 0 修改成 1 则表示能够获得锁。
              if (compareAndSetState(0, 1)) {
                  setExclusiveOwnerThread(Thread.currentThread());
                  return true;
              }
              return false;
          }
    
          // 释放锁
          protected boolean tryRelease(int unused) {
              // 持有同步器的线程置空
              setExclusiveOwnerThread(null);
              // 同步器状态置为 0
              setState(0);
              return true;
          }
    
          public void lock()        { acquire(1); }
          public boolean tryLock()  { return tryAcquire(1); }
          public void unlock()      { release(1); }
          public boolean isLocked() { return isHeldExclusively(); }
    
          // 关闭当前任务
          void interruptIfStarted() {
              Thread t;
              if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                  try {
                      t.interrupt();
                  } catch (SecurityException ignore) {
                  }
              }
          }
      }
```

<a id="orgd6de845"></a>

## BlockingQueue workQueue

workQueue 阻塞队列，是为来不及及时消费的任务提供临时存储的缓冲池。在提交任务时（execute(Runnable command)）,如果当前执行任务的线程数已经大于或者等于核心线程数，那么会将任务存储到阻塞队列中，等待线程空闲出来执行队列中的任务。如果此时队列已经满了，那么将会继续添加工作线程（addWorker）。但是如果执行任务的线程数已经到达线程池线程数上限，那么将会拒绝任务（reject(command)）。

workQueue 还在执行 Wroker 过程中（runWorker）用于不断提供任务（while (task != null \|\| (task = getTask()) != null)）。


<a id="org9885238"></a>

## Condition termination

termination 是 mainLock（ReentrentLock）提供的竞态条件，用于提供对 awaitTermination 的支持。如果 ThreadPoolExecutor 被关闭（调用 shutDown 或者 shutDownNow）或者需要被关闭时，都会调用 tryTerminate()触发 termination.signalAll()。而 awaitTermination 方法会调用 termination.awaitNanos(nanos)，阻塞等待指定的时间。如果在指定时间内收到关闭完成的信号（termination.signalAll()）或者等待超时则返回 true 或者 false。

awaitTermination 的使用场景如下：

```java
    // 关闭线程池
    pool.shutdown();
    try {
      // 等待线程池关闭完成。如果线程池没有在指定时间内完成关闭（还有任务在执行），那么使用 shutdownNow（正在执行的任务也会被关闭）关闭线程池。
      if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
        pool.shutdownNow(); // Cancel currently executing tasks
        // 继续等待线程池被关闭，如果在等待时间内还未被关闭，那么记录异常
        if (!pool.awaitTermination(60, TimeUnit.SECONDS))
            System.err.println("Pool did not terminate");
      }
    } catch (InterruptedException ie) {
      // 如果当前调用关闭线程池的线程被中断，那么继续关闭线程池，并将线程中断标识置为中断状态（捕获到 InterruptedException 后线程的中断标志会被清空）。
      pool.shutdownNow();
      Thread.currentThread().interrupt();
    }
```

<a id="orgf9c06f2"></a>

## ThreadFactory threadFactory

ThreadPoolExecutor 中使用的线程都是 ThreadFactory 创建的。Executors 提供了两种实现：

1.  DefaultThreadFactory：能够创建在用一个线程组和拥有同样命名规则的线程。
2.  PrivilegedThreadFactory：继承 DefaultThreadFactory，额外增加了拥有可以访问调用线程的上下文权限（拥有相同的 AccessControlContext 和 ClassLoader）。


<a id="org4cdee23"></a>

## RejectedExecutionHandler handler

handler 是阻塞满了，执行任务的线程数也已经达到配置的上限，那么对新接入的任务应当如何执行的策略。

```java
    public interface RejectedExecutionHandler {
    
        /**
         * 当线程池 ThreadPoolExecutor 的队列无法存储更多任务，并且线程已经达到上限时会调用该方法会调用该方法。
         * 如果队列已经关闭也可能会调用该方法
         *
         * @param r 任务
         * @param executor 线程池
         * @throws RejectedExecutionException 如果没有任何补救措施那么抛出该异常
         */
        void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
    }
```

ThreadPoolExecutor 提供了四种实现：

1.  AbortPolicy：没有不就措施，抛出 RejectedExecutionException 异常
2.  DiscardPolicy：直接丢弃任务，不做任何处理
3.  DiscardOldestPolicy：如果线程池没有关闭，那么丢弃阻塞队列中的第一个元素（最早进入队列中的任务）并提交任务（execute）。如果依然无法添加，继续执行 DiscardOldestPolicy 策略（不断循环直到添加成功）。
4.  CallerRunsPolicy：让提交任务的线程执行任务。


<a id="org3fc7eb0"></a>

# execute 方法原理

ThreadPoolExecutor 利用 AtomicInteger ctl 来记录工作线程数和线程池状态。ThreadPoolExecutor 基于 ctl 维护了两个属性：

1.  workerCount：工作线程数
2.  runState：ThreadPoolExecutor 的状态，包括运行中，关闭等。

为了让 workerCount 和 runState 都维护在一个值上（ctl），限制了 workerCount 上限只有 2<sup>29</sup>-1，而不是 int 的最大范围。如果这个限制太小不满足需求，那么将来可以使用 AtomicLong 代替。使用 int 会使代码简单一些，速度也快一些。

workerCount 表示那些

ThreadPoolExecutor 的执行任务的流程如下：

![execute 流程](/assets/image/ThreadPoolExecutor.execute.png)

源码介绍：

```java
    /**
     * 执行提交的任务。通常情况，线程池会使用维护的线程异步执行提交的任务，并确在满足拒绝执行的情况下执行拒绝线程池初始化时选择的拒绝策略。
     * 如果线程池被关闭，同样也无法执行任务，同样也会执行拒绝策略。
     * @param command 任务
     * @throws RejectedExecutionException 如果满足拒绝执行的条件，并且没有任何补救措施则会抛出该异常
     * @throws NullPointerException 如果提交的任务是 Null，则抛出该异常
     */
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. 如果当前执行任务的线程数少于核心线程数，那么线程池会创建一个新的工作线程来执行当前提交的任务，即创建一个 Worker 的操作（addWorker）。创建 worker 的操作会再次确认线程池的状态，如果线程已经关闭或者已经达到上限，那么创建失败并返回 false。
         *
         * 2. 如果工作线程数大于或者等于核心线程数或者 addWorker 操作失败（返回 false），那么校验线程池未被关闭并且将任务成功提交给阻塞队列。如果校验通过，double-check 线程池状态。如果线程池关闭，则删除任务并执行拒绝策略；否则如果工作线程数是 0 则添加工作线程（addWorker）。
         *
         * 3. 如果将任务添加到阻塞队列中失败，那么执行 addWorker 操作添加工作线程。如果也同样失败则执行拒绝策略（说明已经是阻塞队列满并且工作线程数达到最大线程数的状态）。
         */
    
        // 当前工作线程数
        int c = ctl.get();
        // 如果 worker 的数量小于核心线程数，那么添加 worker
        if (workerCountOf(c) < corePoolSize) {
            // 添加 worker 操作还会做 double-check，如果添加成功则退出，否则后续做入队操作
            if (addWorker(command, true))
                return;
            // 如果添加失败，那么获取最新的 worker 数量
            c = ctl.get();
        }
        // 如果线程未关闭，那么执行入队操作
        if (isRunning(c) && workQueue.offer(command)) {
            // 如果正确入队，再次确认线程池属于未关闭状态。
            // 如果线程池关闭，那么删除刚入队的任务，并执行拒绝策略
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 如果线程池未关闭或者删除任务失败，校验线程池中的工作线程，如果不存在则添加工作线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 入队失败（队列已经满了），再次执行添加 worker 的操作（如果超过核心线程数并且队列已经达到容量上限，在未超过最大线程数的配置下，会创建新线程处理）。
        else if (!addWorker(command, false))
            // 添加 worker 失败，可能是已经达到最大线程数，那么执行拒绝策略。
            reject(command);
    }
```

<a id="org5010304"></a>

# 改变线程池状态的方法

ThreadPoolExecutor的状态信息存储在属性AtomicInteger ctl中，ctl值的范围是在-536870912到1610612736之间。为了保证线程池的状态变更是线程安全的，所以使用AtomicInteger存储状态。其中0表示SHUTDOWN状态。-536870912～0之间表示RUNNING。-536870912表示没有任何工作线程在运行，以后没添加一个工作线程，ctl+1。所以ThreadPoolExecutor最多允许的线程数是536870911个线程。大于0的值只能是536870912，表示停止（STOP）；1073741824表示关闭中（TIDYING）;1610612736表示终止（TERMINATED）。

之所以采用这种方式是方便采用位运算快速转换各个状态的值，比如：

```java
    // 当前线程池的状态 RUNNING（-536870912～0），SHUTDOWN（0~536870912），STOP（536870912~1073741824），TIDYING（1073741824），TERMINATED（1610612736）
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    // 当前工作线程的数量。只有ctl的值在-536870912～0（不包括0）之间才有工作线程，并且数量在0～536870911（包括0）之间
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    // 将对应的状态转成
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```

ThreadPoolExecutor大致有以下的几种转变：

-   RUNNING -> SHUTDOWN：运行的线程池调用shutdown方法
-   (RUNNING or SHUTDOWN) -> STOP：运行的状态或者关闭状态调用shutdownNow()方法
-   SHUTDOWN -> TIDYING：关闭状态下阻塞队列是空队列并且没有任何工作线程会转变成TIDYING状态
-   STOP -> TIDYING：线程池没有工作线程停转状态会转变成TIDYING
-   TIDYING -> TERMINATED：terminated()的钩子方法执行结束后状态变成TERMINATED

以下是转变状态的方法：

```java
    /**
     * 将状态变更成指定的状态
     *
     * @param targetState 指定状态。ThreadPoolExecutor里是SHUTDOWN或者STOP。
     */
    private void advanceRunState(int targetState) {
        //无限循环直到更新成功
        for (;;) {
            // 当前状态码
            int c = ctl.get();
            // 如果当前状态码已经大于给的状态码，那么无需修改跳出循环。
            // CAS修改，将状态码改成状态码和工作线程数或运算后的结果。这是STOP的状态位于536870912~1073741824（不包括1073741824）的原因。即如果当前工作线程数是2，那么变更后的结果是536870914。
            if (runStateAtLeast(c, targetState) ||
                ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
                break;
        }
    }
    
    /**
     * 终止线程池
     */
    final void tryTerminate() {
        for (;;) {
            // 当前状态码
            int c = ctl.get();
            // 如果当前状态还是运行中的状态，那么直接跳出结束。状态变化不允许RUNNING -> TIDYING和RUNNING -> TERMINATED
            // 如果状态已经是TIDYING或者TERMINATED，那么不需要做任何处理，直接退出方法
            // 如果状态是SHUTDOWN，并且阻塞队列还有任务未执行，那么直接退出等待阻塞队列中的任务执行结束。
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            // 说明已经处于关闭线程池的过程中，但是此时还有工作线程。那么关闭一个空闲工作线程（也可能不存在），目的在于通知其他所有阻塞等待获取任务的空闲工作线程退出（发送关闭信号），所以不需要关闭所有的空闲线程。
            // 发送信号的逻辑是：当关闭一个工作线程（interrupt()），会使得Worker执行跳出循环等待获取任务并进入processWorkerExit()方法，该方法会继续调用tryTerminate通知其他空闲的工作线程关闭（见runWorker）
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }
            // 上
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 将线程池CAS变更成TIDYING状态
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        // 变更成功调用terminated()钩子方法
                        terminated();
                    } finally {
                        // 钩子方法执行结束（或者异常中断）后将线程池变更成TERMINATED状态。
                        ctl.set(ctlOf(TERMINATED, 0));
                        // 发送关闭线程池结束信号（awaitTermination方法会阻塞直到该通知或者超时）
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
        }
    }
```

<a id="org8e9f6d3"></a>

# 工作线程控制任务中断的方法

ThreadPoolExecutor会回收那些超长空闲的工作线程或者强制关闭时回收正在进行中的工作线程，所以定义了一些私有方法用于终止这些工作线程

```java
    /**
     * 校验关闭工作线程的权限
     * 如果有安全管理器，校验调用线程是否有中断工作线程（Worker）绑定的线程（Thread）的权限。
     */
    private void checkShutdownAccess() {
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkPermission(shutdownPerm);
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                for (Worker w : workers)
                    security.checkAccess(w.thread);
            } finally {
                mainLock.unlock();
            }
        }
    }
    
    /**
     * 中断所有的工作线程
     */
    private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
    
    /**
     * 中断空闲的工作线程。
     * 空闲的工作线程是指当前阻塞等待获取任务（w.tryLock()拿不到锁）的工作线程。
     * 如果onlyOne为true，那么便利到一个工作线程则退出。
     * onlyOne为true的场景是tryTerminate通过关闭一个空闲的工作线程来通知关闭信号。
     */
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
    
    /**
     * 关闭所有的空闲工作线程
     */
    private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
    }
```

<a id="org99df002"></a>

# 工作线程添加、运行和退出等方法

```java
    /**
     * 添加一个工作线程
     * 如果线程池的状态是RUNNING，并且未超过线程数量限制（如果core是true则不应该超过核心线程数限制，否则是最大线程数限制）并且线程池成功创建线程，那么会添加一个工作线程。工作线程会立即开始工作，并首先执行指定的第一个任务。后续会通过getTask从阻塞队列中获取任务并执行，否则阻塞在getTask中。
     *
     * @param firstTask 工作线程第一个任务。如果不存在则给null。这个设定是为了解决在线程数量少于核心线程数时，任务可以不用进入等待队列执行让工作线程执行；或者队列已经满了，此时无法进入队列。
     *
     * @param core 是否创建核心线程数内的线程
     * @return 如果工作线程创建成功则返回true，否则返回false
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            // 状态吗
            int c = ctl.get();
            // 状态 RUNNING，SHUTDOWN，STOP，TIDYING，TERMINATED中的一种。采用和CAPACITY补码做与运算，0~536870912之间的数值都会得到0（SHUTDOWN）的结果。
            int rs = runStateOf(c);
    
            // 如果状态不是RUNNING，并且不属于是SHUTDOWN并且队列还有任务的情况，那么返回false。无需添加工作线程。如果是SHUTDOWN但是队列还有任务未执行，那么还需要工作线程继续执行队列中的任务，此种情况下允许继续添加工作线程，并且工作线程只用于执行队列中的任务。
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
    
            // 如果是RUNNING状态，那么状态码加一
            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                // 获取最新的状态，再次校验线程池状态是否发生变化
                c = ctl.get();
                if (runStateOf(c) != rs)
                    continue retry;
                // compareAndIncrementWorkerCount 更新失败，重新获取最新的重新走循环更新工作线程数
            }
        }
    
        // 工作线程是否开始工作
        boolean workerStarted = false;
        // 工作线程是否添加成功
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // 线程池最新的状态，用于再次确认
                    int rs = runStateOf(ctl.get());
                    // 再次校验工作线程是否满足启动的条件
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        // 如果线程是已经启动的状态那么抛出异常。刚创建的线程应该是未启动的状态
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                // 如果工作线程添加成功，那么让工作线程开始工作。
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            // 如果工作线程未开始工作，那么执行回退操作
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
    
    /**
     * 添加工作线程失败后的回退操作
     * - 从队列中删除之前添加的工作线程
     * - 工作线程数减一
     * - 执行tryTerminate方法，校验是否需要终止线程池。如果线程池在调用addWorkerFailed前一刻被终止，由于workers中还有当前的工作线程，所以无法变成TIDYING和TERMINATED状态。
     */
    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
    
    /**
     * 工作线程退出方法
     * 如果工作线程在执行过程中遇到异常或者线程被中断或者超长空闲(getTask() == null)，那么执行销毁工作线程任务。
     * 如果是由于异常或者线程中断导致的退出，并且线程池状态是RUNNING或者SHUTDOWN(SHUTDOWN状态依然需要工作线程执行队列中的任务)，那么需要补充一个工作线程。
     *
     * @param w 工作线程
     * @param completedAbruptly 是否由于是异常（包括线程中断）导致的退出
     */
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        // 如果是异常退出导致需要关闭工作线程，那么之前创建工作线程时已经添加过工作线程数量（状态码加一），那么需要现在需要减少工作线程数量
        if (completedAbruptly)
            decrementWorkerCount();
    
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }
        // 校验是否需要关闭线程池
        tryTerminate();
    
        int c = ctl.get();
        // 如果是RUNNING或者SHUTDOWN状态并且当前需要工作线程那么创建一个工作线程
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                // 工作线程数量已经超过所需的工作线程数量，不需要创建新的工作线程
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
    
    /**
     * 获取任务
     * 工作线程在执行完第一个任务（初始化给定的）后，会阻塞等待阻塞队列给出一个新的任务。
     *
     * @return 阻塞队列中的任务。如果线程池已经被关闭或者超时，那么返回null
     */
    private Runnable getTask() {
        boolean timedOut = false; 
    
        for (;;) {
            // 状态码
            int c = ctl.get();
            // 状态
            int rs = runStateOf(c);
    
            // 如果线程池是SHUTDOWN状态并且阻塞队列是空队列（SHUTDOWN状态依然需要处理阻塞队列中的任务）或者线程池是STOP及后续的状态那么表示当前工作线程需要关闭，减少工作线程数量并返回null。
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
            // 工作线程数量
            int wc = workerCountOf(c);
    
            // 是否允许工作线程超长空闲关闭并且线程数量大于核心线程数
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            // 当前工作线程满足超长空闲关闭的条件，那么工作线程数量减一并返回null
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }
            // 从阻塞队列获取任务
            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            // 如果工作线程的线程被中断码，那么重新获取任务
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
    
    /**
     * 工作线程开始执行任务
     * 不断从阻塞队列中获取任务并执行。
     * 如果线程池关闭，那么回收工作线程。
     * 如果是执行任务发送异常或者线程被中断导致工作线程绑定的线程被终止并回收，那么根据情况（当前线程池是否需要一个新的工作线程）创建一个新的工作线程代替原来的工作线程。
     *
     * @param w the worker
     */
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        // 释放锁。工作线程初始化时会将state设置为-1，如此终止空闲工作线程的方法不会关闭这些刚初始化的工作线程。
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 不断从阻塞队列中获取任务并执行。
            // 如果线程池的状态是SHUTDOWN并且阻塞队列是空队列或者状态是SHUTDOWN以后的状态，那么getTask返回null，表示需要关闭当前工作线程。
            // 如果工作线程长时间未能从阻塞队列中获取任务，getTask也可能返回null，也将会关闭工作线程。
            while (task != null || (task = getTask()) != null) {
                // 获取到任务，工作线程上锁，表示并非是空闲的工作线程。
                w.lock();
                // 如果线程池已经是SHUTDOWN后续状态（所有的工作线程都需要被关闭）并且当前工作线程没有被中断，那么中断工作线程绑定的线程。
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    // 满足条件中断工作线程绑定（Worker）的线程(Thread)，执行finally中的processWorkerExit，关闭工作线程
                    wt.interrupt();
                try {
                    // 执行任务开始前的钩子方法
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 执行任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        // 执行钩子方法
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            // 执行任务时未发生异常。这里是getTask返回null，工作线程需要被关闭。
            completedAbruptly = false;
        } finally {
            // 回收工作线程
            processWorkerExit(w, completedAbruptly);
        }
    }
```

<a id="org5f9d09c"></a>

# 其他方法的源码

```java  
    /**
     * 关闭线程池。该方法不会立即停止已经提交的任务，但是会拒绝接受新的任务。如果对已经关闭的线程池调用该方法不会产生任何效果。
     * 该方法不会阻塞，即该方法执行结束后不代表线程池关闭完成。如果希望获得关闭完成的通知，可以使用 awaitTermination。
     *
     * @throws SecurityException {@inheritDoc}
     */
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 如果存在 SecurityManager，那么校验调用线程是否有“modifyThread”的 RuntimePermission
            checkShutdownAccess();
            // while true + cas更新，将线程池状态更新成关闭（shutdown）状态
            advanceRunState(SHUTDOWN);
            // 关闭没有执行任务的工作线程（worker）
            interruptIdleWorkers();
            // 调用关闭线程的钩子方法。一般用于统计类场景。
            onShutdown();
        } finally {
            mainLock.unlock();
        }
        // 如果当前状态不是（shutdown并且worker队列是空队列）并且不是（stop并且worker队列是空队列）,尝试更新成终止状态
        tryTerminate();
    }
    
    /**
     * 关闭所有的工作线程，并终止等待任务，并返回等待执行的任务列表。
     * 终止任务是通过线程的interrupt方法实现的。如果线程不响应中断信号，那么任务可能无法被中断。
     *
     * @throws SecurityException {@inheritDoc}
     */
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 校验线程的访问权限
            checkShutdownAccess();
            // 将线程池状态改成stop状态
            advanceRunState(STOP);
            // 终止所有工作线程
            interruptWorkers();
            // 将阻塞队列中的任务取出并清空阻塞队列
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        // 如果当前状态不是（shutdown并且worker队列是空队列）并且不是（stop并且worker队列是空队列）,尝试更新成终止状态
        tryTerminate();
        return tasks;
    }
    
    // 一切不属于运行中的状态都是关闭状态。
    public boolean isShutdown() {
        return ! isRunning(ctl.get());
    }
    
    /**
     * 如果线程池已经开始关闭，但是还未完成，那么返回true。即状态位于RUNNING和TERMINATED之间
     *
     * @return {@code true} if terminating but not yet terminated
     */
    public boolean isTerminating() {
        int c = ctl.get();
        return ! isRunning(c) && runStateLessThan(c, TERMINATED);
    }
    
    // 如果当前是TERMINATED状态则返回true，否则返回false
    public boolean isTerminated() {
        return runStateAtLeast(ctl.get(), TERMINATED);
    }
    
    /**
     * 等待线程池关闭结束。该方法回阻塞线程知道线程池关闭结束或者超时。
     * 一般用于关闭线程池。线程池shutdown后等待一段时间，如果线程池还未关闭，那么使用shutdownNow关闭线程池。
     * @param timeout 超时时间
     * @param unit 时间单位
     */
    public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (;;) {
                // 比较当前状态是否>=TERMINATED，如果是则返回true
                if (runStateAtLeast(ctl.get(), TERMINATED))
                    return true;
                // 说名当前还未处于TERMINATED状态，那么查看是否超时。如果超时则返回false。
                if (nanos <= 0)
                    return false;
                // 未超时并且不处于TERMINATED状态，那么阻塞等待关闭结束通知。
                nanos = termination.awaitNanos(nanos);
            }
        } finally {
            mainLock.unlock();
        }
    }
    
    /**
     * 重写回收线程池对象的方法，回收时关闭线程池。
     */
    protected void finalize() {
        SecurityManager sm = System.getSecurityManager();
        if (sm == null || acc == null) {
            shutdown();
        } else {
            PrivilegedAction<Void> pa = () -> { shutdown(); return null; };
            AccessController.doPrivileged(pa, acc);
        }
    }
    
    /**
     * Sets the thread factory used to create new threads.
     *
     * @param threadFactory the new thread factory
     * @throws NullPointerException if threadFactory is null
     * @see #getThreadFactory
     */
    public void setThreadFactory(ThreadFactory threadFactory) {
        if (threadFactory == null)
            throw new NullPointerException();
        this.threadFactory = threadFactory;
    }
    
    /**
     * Returns the thread factory used to create new threads.
     *
     * @return the current thread factory
     * @see #setThreadFactory(ThreadFactory)
     */
    public ThreadFactory getThreadFactory() {
        return threadFactory;
    }
    
    /**
     * Sets a new handler for unexecutable tasks.
     *
     * @param handler the new handler
     * @throws NullPointerException if handler is null
     * @see #getRejectedExecutionHandler
     */
    public void setRejectedExecutionHandler(RejectedExecutionHandler handler) {
        if (handler == null)
            throw new NullPointerException();
        this.handler = handler;
    }
    
    /**
     * Returns the current handler for unexecutable tasks.
     *
     * @return the current handler
     * @see #setRejectedExecutionHandler(RejectedExecutionHandler)
     */
    public RejectedExecutionHandler getRejectedExecutionHandler() {
        return handler;
    }
    
    /**
     * 调整核心线程数
     *
     * @param corePoolSize 核心线程数
     * @throws IllegalArgumentException 如果数值小于0，那么抛出该异常
     * @see #getCorePoolSize
     */
    public void setCorePoolSize(int corePoolSize) {
        if (corePoolSize < 0)
            throw new IllegalArgumentException();
        int delta = corePoolSize - this.corePoolSize;
        this.corePoolSize = corePoolSize;
        // 如果工作线程数超过了新设置的核心线程数，那么关闭空闲工作线程。
        if (workerCountOf(ctl.get()) > corePoolSize)
            interruptIdleWorkers();
        // 如果工作线程数小于或者等于核心线程数，那么尝试添加新的工作线程
        else if (delta > 0) {
            // 需要新增的工作线程数量。取差值和阻塞队列的长度的最小值
            int k = Math.min(delta, workQueue.size());
            // 添加工作线程
            while (k-- > 0 && addWorker(null, true)) {
                if (workQueue.isEmpty())
                    break;
            }
        }
    }
    
    /**
     * 返回核心线程数
     *
     * @return the core number of threads
     * @see #setCorePoolSize
     */
    public int getCorePoolSize() {
        return corePoolSize;
    }
    
    /**
     * 启动一个核心线程。只有当工作线程数小于核心线程数时才能够启动成功
     *
     * @return {@code true} if a thread was started
     */
    public boolean prestartCoreThread() {
        return workerCountOf(ctl.get()) < corePoolSize &&
            addWorker(null, true);
    }
    
    /**
     * 启动一个工作线程。只有当工作线程数小于最大线程数时才会启动成功
     */
    void ensurePrestart() {
        int wc = workerCountOf(ctl.get());
        if (wc < corePoolSize)
            addWorker(null, true);
        else if (wc == 0)
            addWorker(null, false);
    }
    
    /**
     * 将线程启动到核心线程数的个数
     *
     * @return 返回本次操作启动工作线程的个数
     */
    public int prestartAllCoreThreads() {
        int n = 0;
        while (addWorker(null, true))
            ++n;
        return n;
    }
    
    /**
     * 如果允许核心线程数内的工作线程由于长时间空闲而关闭，那么返回true，否则返回false
     *
     * @return 如果允许长时间空闲而关闭则返回true，否则返回false
     *
     * @since 1.6
     */
    public boolean allowsCoreThreadTimeOut() {
        return allowCoreThreadTimeOut;
    }
    
    /**
     * 设置是否允许核心线程数内的工作线程空闲时间超过指定的最大空闲时间后关闭该工作线程。
     *
     * @param value true表示允许核心线程数内的工作线程超时空闲关闭
     * @throws IllegalArgumentException 如果允许关闭但是未设置空闲时间，那么抛出该异常
     *
     * @since 1.6
     */
    public void allowCoreThreadTimeOut(boolean value) {
        if (value && keepAliveTime <= 0)
            throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
        if (value != allowCoreThreadTimeOut) {
            allowCoreThreadTimeOut = value;
            // 如果允许超长空闲关闭，那么立即清理空闲的工作线程
            if (value)
                interruptIdleWorkers();
        }
    }
    
    /**
     * 设置最大线程数
     *
     * @param maximumPoolSize 最大线程数
     * @throws IllegalArgumentException 如果最大线程数小于等于0或者小于核心线程数，那么抛出该异常
     * @see #getMaximumPoolSize
     */
    public void setMaximumPoolSize(int maximumPoolSize) {
        if (maximumPoolSize <= 0 || maximumPoolSize < corePoolSize)
            throw new IllegalArgumentException();
        this.maximumPoolSize = maximumPoolSize;
        // 如果当前的工作线程数已经超过最大线程数，那么关闭空闲的工作线程
        if (workerCountOf(ctl.get()) > maximumPoolSize)
            interruptIdleWorkers();
    }
    
    /**
     * 返回最大线程数
     *
     * @return 最大线程数
     * @see #setMaximumPoolSize
     */
    public int getMaximumPoolSize() {
        return maximumPoolSize;
    }
    
    /**
     * 设置空闲时间。
     * 超过核心线程数的工作线程如果空闲时间超过设置的空闲时间，那么此类工作线程将会被关闭。
     * 如果设置了允许核心线程数内的工作线程超长空闲时间关闭（allowCoreThreadTimeOut == true），
     * 那么所有工作线程空闲时间超过设置的空闲时间（keepAliveTime）都将会被关闭。
     *
     * @param time 空闲时间
     * @param unit 时间单位
     * @throws IllegalArgumentException 如果小于等于0则抛出该异常
     * @see #getKeepAliveTime(TimeUnit)
     */
    public void setKeepAliveTime(long time, TimeUnit unit) {
        if (time < 0)
            throw new IllegalArgumentException();
        if (time == 0 && allowsCoreThreadTimeOut())
            throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
        long keepAliveTime = unit.toNanos(time);
        long delta = keepAliveTime - this.keepAliveTime;
        this.keepAliveTime = keepAliveTime;
        if (delta < 0)
            interruptIdleWorkers();
    }
    
    /**
     * 返回设置的空闲时间
     *
     * @param unit 返回时间的时间单位
     * @return 空闲时间
     * @see #setKeepAliveTime(long, TimeUnit)
     */
    public long getKeepAliveTime(TimeUnit unit) {
        return unit.convert(keepAliveTime, TimeUnit.NANOSECONDS);
    }
```
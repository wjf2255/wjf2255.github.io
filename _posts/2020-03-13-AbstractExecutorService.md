---
title: AbstractExecutorService源码介绍
tags: [Java, JUC, 源码, 多线程, 线程池, AbstractExecutorService]
layout: post
author: wjf
---

<div id="table-of-contents">
<h2>目录</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#org8eee5fc">介绍</a></li>
<li><a href="#orga3197d6">Executor</a></li>
<li><a href="#org99c409d">ExecutorService</a>
<ul>
<li><a href="#org7257a5a">方法定义</a></li>
</ul>
</li>
<li><a href="#orge423e0a">CompletionService</a></li>
<li><a href="#orgf00ec56">ExecutorCompletionService</a></li>
<li><a href="#orgcdf46bd">AbstractExecutorService</a></li>
</ul>
</div>
</div>


<a id="org8eee5fc"></a>

# 介绍

AbstractExecutorService 是 JDK 线程池 ThreadPoolExecutor，ForkJoinPool 等的父类，提供了许多非常有用的功能。

AbstractExecutorService 的继承关系如下：

![继承关系](/assets/image/AbstractExecutorService.png)


<a id="orga3197d6"></a>

# Executor

Executor 是执行 Runnable 任务的接口，用于解耦任务创建和任务执行，包括线程的使用和执行的排期等。在需要创建多个线程执行任务时，Executor 还可以代替 new Thread(new(RunnableTask())).start()，而使用

```java
    Executor executor = anExecutor;
    executor.execute(new RunnableTask1());
    executor.execute(new RunnableTask2());
```

然而 Executor 并不要求任务按照异步的方式执行，也可以只是使用当前线程执行任务，比如：

```java
    class DirectExecutor implements Executor {
      public void execute(Runnable r) {
        r.run();
      }
    }
```

但是 Executor 的实现类一般来说都是使用异步的方式执行提交的任务，如下：

```java
    class ThreadPerTaskExecutor implements Executor {
      public void execute(Runnable r) {
        new Thread(r).start();
      }
    }
```

Executor 接口方法定义如下：

```java
    /**
     *
     * 在接下来的某个时刻执行任务。根据实现类的不同，任务可能在调用方的线程中执行，也可能创建一个新的线程执行，或者在一个线程池中执行。
     *
     * @param command the runnable task 可执行的任务（实现 Runnable 接口的对象）
     * @throws RejectedExecutionException if this task cannot be
     * accepted for execution 如果 Executor 不接受任务则抛出该异常
     * @throws NullPointerException if command is null 如果任务为为空则抛出该异常
     */
    void execute(Runnable command);
```

<a id="org99c409d"></a>

# ExecutorService

ExecutorService 提供一些方法用于管理服务终止以及提供一些返回计算结果（Future） 的执行任务的方法。Future(计算结果)可以用于跟踪异步任务的执行结果，查看是否完成，取消未完成的任务等。

ExecutorService 可以被关闭。被关闭的 ExecutorService 无法再执行异步任务。ExecutorService 提供了两个关闭 ExecutorService 的方法：

1.  shutdown：该方法会继续执行在调用该方法前提交的任务，但是会拒绝之后提交的任务。在之前提交的任务全部结束后关闭 ExecutorService。
2.  shutdownNow: 该方法会暂停正在进行中和等待中的任务。

ExecutorService 如果未在使用，那么应该关闭回收计算机资源。

ExecutorService 提供 submit 方法，用于扩展 execute。submit 能够返回一个 Future。invokeAny 和 invokeAll 能够处理一批的任务，并等待一个或者所有的异步任务完成。

ExecutorService 使用例子：

```java
    class NetworkService implements Runnable {
        private final ServerSocket serverSocket;
        private final ExecutorService pool;
    
        public NetworkService(int port, int poolSize)
            throws IOException {
          serverSocket = new ServerSocket(port);
          pool = Executors.newFixedThreadPool(poolSize);
        }
    
        public void run() { // run the service
          try {
            for (;;) {
              pool.execute(new Handler(serverSocket.accept()));
            }
          } catch (IOException ex) {
            pool.shutdown();
          }
        }
      }
    
      class Handler implements Runnable {
        private final Socket socket;
        Handler(Socket socket) { this.socket = socket; }
        public void run() {
          // read and service request on socket
        }
      }
```

关停 ExecutorService 的例子：

```java
    void shutdownAndAwaitTermination(ExecutorService pool) {
        //  拒接接受新任务
        pool.shutdown();
        try {
          // 如果服务在 60 秒还未关停则做更多的处理
          if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
            //立即关停所有任务
            pool.shutdownNow();
            // 如果服务在 60 秒还未关停则输出提示
            if (!pool.awaitTermination(60, TimeUnit.SECONDS))
                System.err.println("Pool did not terminate");
          }
        } catch (InterruptedException ie) {
          // 如果遇到线程中断异常那么立即关停服务并将线程置为中断状态
          pool.shutdownNow();
          Thread.currentThread().interrupt();
        }
      }
```

ExecutorService 提供的内存一致性保证：提交任务操作一定在线程处理任务之前完成，并且线程处理任务一定在 Future.get()获取到异步任务结果前完成。


<a id="org7257a5a"></a>

## 方法定义

```java
    /**
     * 开始关停 ExecutorService。
     * 关停前提交的任务会继续执行，但是不会接受新任务。如果是已经关停的 ExecutorService 调用该方法不会发生任何变换。
     *
     * @throws SecurityException 如果存在安全管理器并且关闭线程没有权限修改线程状态，那么会抛出该异常。没有权限修改的原因是没有获得 RuntimePermission 或者是安全管理器检查访问权限时校验未通过。
     */
    void shutdown();
    
    /**
     * 立即关停 ExecutorService
     * 尝试关停所有进行中的任务，同时拒绝任何新任务，停止处理等待中的任务，并且返回等待任务的列表。
     * 正在进行中的任务没法保证一定成功停止，比如通过线程中断方法（Thread.interrupt）中断进行中的任务，而任务没有响应线程中断，那么任务无法终止。
     *
     * @return 正在进行中的任务列表
     * @throws SecurityException 如果存在安全管理器并且关闭线程没有权限修改线程状态，那么会抛出该异常。没有权限修改的原因是没有获得 RuntimePermission 或者是安全管理器检查访问权限时校验未通过。
     */
    List<Runnable> shutdownNow();
    
    /**
     * 返回关闭状态：如果关闭则返回 true，否则返回 false
     *
     * @return 关闭状态：如果关闭则返回 true，否则返回 false
     */
    boolean isShutdown();
    
    /**
     * 如果关闭后所有任务都完成那么返回 true
     *
     * @return 如果关闭后所有任务都完成那么返回 true
     */
    boolean isTerminated();
    
    /**
     * 当关闭 ExecutorService 之后用于阻塞等待给定的时间，让正在进行中的任务完成，直到任意条件满足：
     * 1. 所有任务都已经完成
     * 2. 等待超时
     * 3. 线程被中断
     *
     * @param 等待的时间
     * @param unit the time unit of the timeout argument 时间单位
     * @return 如果 ExecutorService 在时间内终止了那么返回 true，否则返回 false
     * @throws InterruptedException 当前线程被中断抛出中断异常
     */
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    
    /**
     * 提交一个可执行的并且由返回值的任务，并返回一个任务执行结果（Future）。
     * 如果希望立即阻塞直到获取异步任务结果可以使用 result = exec.submit(aCallable).get()。Future.get()或阻塞直到获取到异步任务结果。
     * ExecutorService 提供了很多方法用于将闭包对象转成被执行的方法。
     *
     * @param 可被 ExecutorService 执行的任务
     * @param <T> 任务的执行结束后返回的结果的对象类型
     * @return 可以跟踪异步任务执行结果的对象（Future）
     * @throws RejectedExecutionException 如果 ExecutorService 拒绝任务则抛出
     * @throws NullPointerException if the task is null
     */
    <T> Future<T> submit(Callable<T> task);
    
    /**
     * 提交一个可执行的任务，并指定任务返回的对象类型，该方法返回一个可以跟踪任务执行结果（Future）
     * @param task 可执行的任务
     * @param Result 该任务执行后返回的类型
     * @param <T>任务的执行结束后返回的结果的对象类型
     * @return 任务执行结果（Future）
     * @throws RejectedExecutionException ExecutorService 拒绝任务后抛出该异常
     * @throws NullPointerException 如果任务(task)是 null 抛出该异常
     */
    <T> Future<T> submit(Runnable task, T result);
    
    /**
     * 提交一个可执行的任务，并返回一个可以跟踪任务执行结果的对象（Future）
     *
     * @param task 可执行的任务
     * @return 任务执行结果（Future）
     * @throws RejectedExecutionException 如果 ExecutorService 拒绝该任务则抛出该异常
     * @throws NullPointerException 如果提交了一个空任务则抛出该异常
     */
    Future<?> submit(Runnable task);
    
    /**
     * 执行批量任务并返回跟踪这批任务执行结果的对象集合（List<Future>）
     * 该方法会阻塞等待任务结束后才将结果返回，所以调用方拿到结果时，任务已经结束（或者任务执行过程中断导致的异常结束）。
     * 如果在 invokeAll 执行期间改变任务集合可能会引发未定义方法说明上（throws 定义的异常列表）的异常。
     * 比如 invokeAll 使用 for 循环遍历集合时集合被删除元素抛出 ConcurrentModificationException 异常。
     *
     * @param tasks 任务集合
     * @param <T> 任务类型
     * @return 任务执行结果（Future）集合
     * @throws InterruptedException 如果在等待结果过程中线程被中断则抛出该异常
     * @throws NullPointerException 如果任务结合是空对象则抛出该异常
     * @throws RejectedExecutionException 如果任务提交被拒绝则抛出该异常
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    
    /**
     * 在上一个 invokeAll 基础上增加了等待时间。
     *
     * @param tasks 任务集合
     * @param timeout 等待时间
     * @param unit 时间单位
     * @param <T> 任务完成后返回类型
     * @throws InterruptedException 如果在等待过程中线程被中断则抛出该异常
     * @throws NullPointerException 如果提交的任务集合是空对象则抛出该异常
     * @throws RejectedExecutionException 如果本次提交被拒绝则抛出该异常
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;
    
    /**
     * 执行给定任务列表中的任一任务。
     * 如果有任一任务执行完成则将该任务执行结果返回，并且取消其他任务。
     *
     * @param tasks 任务集合
     * @param <T> 任务返回的类型
     * @return 任务执行结果（Future）
     * @throws InterruptedException 如果等待过程中线程被中断则抛出中断异常
     * @throws NullPointerException 如果提交的任务
     * @throws IllegalArgumentException 如果任务是空列表则抛出该异常
     * @throws ExecutionException 如果没有任何任务执行成功则抛出该异常
     * @throws RejectedExecutionException 如果任务被拒绝则抛出该异常
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
    
    /**
     * 在上一个 invokeAny 基础上增加等待时间
     *
     * @param tasks the collection of tasks
     * @param timeout the maximum time to wait
     * @param unit the time unit of the timeout argument
     * @param <T> the type of the values returned from the tasks
     * @return the result returned by one of the tasks
     * @throws InterruptedException if interrupted while waiting
     * @throws NullPointerException if tasks, or unit, or any element
     *         task subject to execution is {@code null}
     * @throws TimeoutException if the given timeout elapses before
     *         any task successfully completes
     * @throws ExecutionException if no task successfully completes
     * @throws RejectedExecutionException if tasks cannot be scheduled
     *         for execution
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
```

<a id="orge423e0a"></a>

# CompletionService

CompletionService 用于给创建异步任务和消费异步任务结果解耦。CompletionService 依赖 Executor 执行异步任务，而自己管理完成队列，维护异步任务执行结果。

```java
    /**
     * 提交任务
     *
     * @param task 任务
     * @return 任务执行结果（Future）
     * @throws RejectedExecutionException 如果拒绝任务则抛出该异常
     * @throws NullPointerException 如果任务是空对象则抛出该异常
     */
    Future<V> submit(Callable<V> task);
    
    /**
     * 同上
     *
     * @param task the task to submit
     * @param result the result to return upon successful completion
     * @return a Future representing pending completion of the task,
     *         and whose {@code get()} method will return the given
     *         result value upon completion
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     * @throws NullPointerException if the task is null
     */
    Future<V> submit(Runnable task, V result);
    
    /**
     * 获取计算结果并在队列中删除该计算结果。如果没有任务任务完成，那么该方法将会阻塞线程等待任务计算结果
     *
     * @return 计算结果
     * @throws InterruptedException 在等待过程中线程被中断则抛出该异常
     */
    Future<V> take() throws InterruptedException;
    
    /**
     * 同 take()，不一样的地方时该方法不会阻塞，即如果没有任务完成则返回 null。
     *
     * @return 任务计算结果
     */
    Future<V> poll();
    
    /**
     * 通 take()，只是增加了等待超时时间的设置。
     *
     * @param timeout how long to wait before giving up, in units of
     *        {@code unit}
     * @param unit a {@code TimeUnit} determining how to interpret the
     *        {@code timeout} parameter
     * @return the Future representing the next completed task or
     *         {@code null} if the specified waiting time elapses
     *         before one is present
     * @throws InterruptedException if interrupted while waiting
     */
    Future<V> poll(long timeout, TimeUnit unit) throws InterruptedException;
```

<a id="orgf00ec56"></a>

# ExecutorCompletionService

ExecutorCompletionService 是 CompletionService 实现类。ExecutorCompletionService 利用 AbstractExecutorService 构建 RunnableFuture(一个即表示任务又表示计算结果的对象)，通过 Executor 提交任务，并在任务直接结束后将计算结果存到阻塞队列中。

```java
    public class ExecutorCompletionService<V> implements CompletionService<V> {
      //任务执行者，用于提交任务
      private final Executor executor;
      // 任务执行服务，用户构建 RunnableFuture
      private final AbstractExecutorService aes;
      // 阻塞队列。用于存储任务的计算结果
      private final BlockingQueue<Future<V>> completionQueue;
    
      /**
       * 自定义一个计算结果，主要实现完成异步任务时，将计算结果加到阻塞队列中。
       */
      private class QueueingFuture extends FutureTask<Void> {
          QueueingFuture(RunnableFuture<V> task) {
              super(task, null);
              this.task = task;
          }
          // FutureTask 留了一个钩子，用于完成任务后执行相关功能。本身 FutureTask 没有任务实现，这里将计算结果加到阻塞队列中。
          protected void done() { completionQueue.add(task); }
          private final Future<V> task;
      }
    
      // 自定义构建 RunnableFuture
      private RunnableFuture<V> newTaskFor(Callable<V> task) {
          if (aes == null)
              return new FutureTask<V>(task);
          else
              return aes.newTaskFor(task);
      }
      // 自定义构建 RunnableFuture
      private RunnableFuture<V> newTaskFor(Runnable task, V result) {
          if (aes == null)
              return new FutureTask<V>(task, result);
          else
              return aes.newTaskFor(task, result);
      }
    
      /**
       * 构造方法
       *
       * @param executor the executor to use
       * @throws NullPointerException if executor is {@code null}
       */
      public ExecutorCompletionService(Executor executor) {
          if (executor == null)
              throw new NullPointerException();
          this.executor = executor;
          this.aes = (executor instanceof AbstractExecutorService) ?
              (AbstractExecutorService) executor : null;
          this.completionQueue = new LinkedBlockingQueue<Future<V>>();
      }
    
      /**
       * 构造方法
       *
       * @param executor the executor to use
       * @param completionQueue the queue to use as the completion queue
       *        normally one dedicated for use by this service. This
       *        queue is treated as unbounded -- failed attempted
       *        {@code Queue.add} operations for completed tasks cause
       *        them not to be retrievable.
       * @throws NullPointerException if executor or completionQueue are {@code null}
       */
      public ExecutorCompletionService(Executor executor,
                                       BlockingQueue<Future<V>> completionQueue) {
          if (executor == null || completionQueue == null)
              throw new NullPointerException();
          this.executor = executor;
          this.aes = (executor instanceof AbstractExecutorService) ?
              (AbstractExecutorService) executor : null;
          this.completionQueue = completionQueue;
      }
    
      public Future<V> submit(Callable<V> task) {
          if (task == null) throw new NullPointerException();
          RunnableFuture<V> f = newTaskFor(task);
          executor.execute(new QueueingFuture(f));
          return f;
      }
    
      public Future<V> submit(Runnable task, V result) {
          if (task == null) throw new NullPointerException();
          RunnableFuture<V> f = newTaskFor(task, result);
          executor.execute(new QueueingFuture(f));
          return f;
      }
    
      public Future<V> take() throws InterruptedException {
          return completionQueue.take();
      }
    
      public Future<V> poll() {
          return completionQueue.poll();
      }
    
      public Future<V> poll(long timeout, TimeUnit unit)
              throws InterruptedException {
          return completionQueue.poll(timeout, unit);
      }
```

<a id="orgcdf46bd"></a>

# AbstractExecutorService

AbstractExecutorService 实现了 ExecutorService 定义的执行任务的方法，比如 submit，invokeAll，invokeAny 等。AbstractExecutorService 提供了一个 newTaskFor 方法用于构建 RunnableFuture 对象。执行任务方法返回的跟踪任务执行结果的对象都是通过 newTaskFor 来构建的。如果有需要可以通过自定义 newTaskFor 来构建所需的 RunnableFuture。

```java
    /**
     * 构建用于跟踪任务执行结果的对象。
     * 如果有需要可以重写该方法自定义跟踪任务结果对象
     *
     * @param runnable 可执行任务
     * @param value 任务执行结果类型
     * @param <T> 任务执行结果的类型
     * @return 任务执行结果
     * @since 1.6
     */
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
    
    /**
     * 同上
     *
     * @param callable the callable task being wrapped
     * @param <T> the type of the callable's result
     * @return a {@code RunnableFuture} which, when run, will call the
     * underlying callable and which, as a {@code Future}, will yield
     * the callable's result as its result and provide for
     * cancellation of the underlying task
     * @since 1.6
     */
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
    
    /**
     * 提交任务
     */
    public Future<?> submit(Runnable task) {
        // 校验任务，如果是空对象则抛出空指针异常
        if (task == null) throw new NullPointerException();
        // 构建一个 RunnableFuture 对象。RunnableFuture 本身表示一个任务（extends Runnable），同时也表示一个任务执行结果(extends Future)。
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        // 执行任务
        execute(ftask);
        // 将 RunnableFuture 对象返回
        return ftask;
    }
    
    /**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     */
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }
    
    /**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     */
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
    
    /**
     * 执行批量任务中的任一任务
     */
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                              boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {
        // 如果任务是空对象抛出空指针异常
        if (tasks == null)
            throw new NullPointerException();
        int ntasks = tasks.size();
        // 如果任务集合是空集合抛出参数不合法异常
        if (ntasks == 0)
            throw new IllegalArgumentException();
        // 构建计算结果集合
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(ntasks);
    
        // 任务提交和消费的生产者消费者工具
        ExecutorCompletionService<T> ecs =
            new ExecutorCompletionService<T>(this);
    
        // 每次提交前都先检查一下是否有完成的任务，如果完成则直接返回完成任务的计算结果，否则继续提交剩余任务
    
        try {
            //异常信息。用于最后一次获取的异常信息。如果任务没有正常结束，将该异常信息抛出。
            ExecutionException ee = null;
            // 等待时间
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Iterator<? extends Callable<T>> it = tasks.iterator();
    
           // 先提交一个任务，以后再循环时先检查是否有任务已经完成。
           // 利用生产者消费者工具提交任务。任务如果结束后，会将计算结果存入到工具的阻塞队列中。
            futures.add(ecs.submit(it.next()));
            --ntasks;
            int active = 1;
    
            for (;;) {
                // 检查是否有任务已经完成，如果有那么返回任务结果，否则继续提交任务
                Future<T> f = ecs.poll();
                if (f == null) {
                    // 还有任务未提交完，继续提交任务
                    if (ntasks > 0) {
                        --ntasks;
                        futures.add(ecs.submit(it.next()));
                        ++active;
                    }
                    // 所有任务已执行结束，那么跳出循环。如果到这应该有执行异常的结果，后续会抛出执行异常
                    else if (active == 0)
                        break;
                    // 如果设置超时时间，那么利用阻塞队列的等待阻塞获取异常结果。如果超时未获取到计算结果，那么抛出超时异常。
                    else if (timed) {
                        f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
                        if (f == null)
                            throw new TimeoutException();
                        // 正常获取到结果，那么继续更新等待剩余时间。以防止 f.get()异常后还需要走这段代码，需要继续阻塞等待。
                        nanos = deadline - System.nanoTime();
                    }
                    // 如果没有设置超时时间，那么无限等待获取计算结果
                    else
                        f = ecs.take();
                }
                // 如果获取到计算结果，那么将活跃任务(active)数减一，并且返回具体的任务结果。
                if (f != null) {
                    --active;
                    // 如果获取计算结果有异常，那么记录异常信息，继续循环获取其他任务的计算结果
                    try {
                        return f.get();
                    } catch (ExecutionException eex) {
                        ee = eex;
                    } catch (RuntimeException rex) {
                        ee = new ExecutionException(rex);
                    }
                }
            }
            // 到此任务已经全部结束，但是没有任何一个任务正常结束。如果记录的异常不为空则抛出记录的异常信息，否则创建一个执行任务异常并抛出。
            if (ee == null)
                ee = new ExecutionException();
            throw ee;
    
        } finally {
            // 取消所有任务。到这或者是有一个任务已经结束（结束任务依然可以取消，只是没有效果）需要将其他任务取消，或者所有任务都非正常结束（取消任务没有效果）。
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
        }
    }
    
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException {
        try {
            return doInvokeAny(tasks, false, 0);
        } catch (TimeoutException cannotHappen) {
            assert false;
            return null;
        }
    }
    
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                           long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        return doInvokeAny(tasks, true, unit.toNanos(timeout));
    }
    
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        // 任务是否全部结束标志
        boolean done = false;
        try {
            // 构建 RunnableFuture
            // 将计算结果存结果集中
            // 执行任务
            for (Callable<T> t : tasks) {
                RunnableFuture<T> f = newTaskFor(t);
                futures.add(f);
                execute(f);
            }
            // 循环遍历计算结果，如果有未完成，那么阻塞等待任务计算完成
            for (int i = 0, size = futures.size(); i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) {
                    try {
                        f.get();
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    }
                }
            }
            done = true;
            return futures;
        } finally {
            // done == false 表示在执行任务时（execute(f)）或者之前已经出现异常情况，比如当前线程被中断导致无法全部提交，那么将任务全部取消释放资源。
            if (!done)
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);
        }
    }
    
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            // 构建计算结果并加到结果集中
            for (Callable<T> t : tasks)
                futures.add(newTaskFor(t));
    
            // 超时时间节点
            final long deadline = System.nanoTime() + nanos;
            final int size = futures.size();
    
            // 循环遍历结果集并执行任务。每次执行后立即检查超时时间。如果超时则直接返回结果集。既然已经超时，后续任务已经没必要开始。
            for (int i = 0; i < size; i++) {
                execute((Runnable)futures.get(i));
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L)
                    return futures;
            }
    
            // 循环遍历结果集，如果没有完成则阻塞等待完成。
            for (int i = 0; i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) {
                    if (nanos <= 0L)
                        return futures;
                    try {
                        f.get(nanos, TimeUnit.NANOSECONDS);
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    } catch (TimeoutException toe) {
                        return futures;
                    }
                    nanos = deadline - System.nanoTime();
                }
            }
            done = true;
            return futures;
        } finally {
            if (!done)
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);
        }
    }
```

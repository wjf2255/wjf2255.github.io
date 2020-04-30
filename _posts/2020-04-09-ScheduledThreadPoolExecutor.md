---
title: ScheduledThreadPoolExecutor源码介绍
tags: [Java, JUC, 源码, 多线程, 线程池, ScheduledThreadPoolExecutor]
layout: post
author: wjf
---


# 目录

-   [介绍](#org2514652)
-   [原理](#org4db8316)
-   [重要组件](#org113ad30)
    -   [ScheduledFutureTask](#orgb467cab)
    -   [DelayedWorkQueue](#orge4cc33d)
-   [主要源码](#org92abd92)



<a id="org2514652"></a>

# 介绍

ScheduledThreadPoolExecutor在ThreadPoolExecutor基础上另外提供了延迟执行任务和定时执行任务的功能。ScheduledThreadPoolExecutor可以用来代替Timer。

Timer是JDK1.3 提供的一个定时执行任务的工具，支持延迟一次执行或者定时执行。但是Timer本身又一些缺陷，比如单线程执行任务，如果其中一个任务执行时间过长，会影响后续任务的启动；Time基于绝对时间，系统时间的改变会影响Timer执行任务；Timer不会捕获任务执行过程中的异常，导致如果发生异常会意外终止Timer等。

延迟执行的任务能够保证不早于指定延迟时间开始，但是无法保证在延迟时间到达那一刻立即开始。任务会按照执行时间的顺序先后开始执行。

如果任务在延迟执行时间结束前任务被取消，那么定时任务将不会被执行（FutureTask NEW -> CANCELLED流程）,但是默认情况下延迟任务依然还会留在队列中。如果希望取消的任务立刻从阻塞队列中删除，可以设置setRemoveOnCancelPolicy为true。

通过scheduleAtFixedRate和scheduleWithFixedDelay重复执行的任务重复执行的时间不会重叠。尽管重复执行的任务可能会被不同的线程执行，但是后一次执行一定在本次执行结束后才开始执行。即如果安排9:30分开始执行，每3分钟执行一次，但是9:30分执行的任务由于特殊情况执行了4分钟，那么原本9:33分开始执行的任务会被延迟到9:34分开始执行。

ScheduledThreadPoolExecutor虽然继承ThreadPoolExecutor，但是由于使用的应用场景更多是在重复执行一些任务，所以在如何使用上有许多不一样的地方。ScheduledThreadPoolExecutor是一种采用固定线程数和无界队列的线程池，调整最大线程数对于ScheduledThreadPoolExecutor没有任何意义。并且将corePoolSize设置为0或者允许核心线程超长空闲关闭是不合适的，因为这很容易导致一旦需要开始执行任务时线程池没有可用线程。

ScheduledThreadPoolExecutor通过ScheduledFuture来控制延迟执行和重复执行，通过重写execute和submit方法，将Runnable和Callable对象转成ScheduledFuture。为了保证ScheduledThreadPoolExecutor功能的完整性，任何ScheduledThreadPoolExecutor的子类在重写这部分方法时，需要执行父类方法。ScheduledThreadPoolExecutor还为需要定制化ScheduledFuture的场景定义了decorateTask，定义如何将Runnable或者Callable转成ScheduledFuture。如果子类有需要可以重写decorateTask。


<a id="org4db8316"></a>

# 原理

ScheduledThreadPoolExecutor继承ThreadPoolExecutor拥有ThreadPoolExecutor的特点，但是对部分功能做了定制，比如：

1.  队列使用的是延迟队列，一种基于堆实现的无界队列，有点类似PriorityBlockingQueue。获取延迟队列中的任务不仅要满足队列中有任务，而且要满足时间需求才能对外提供任务。该功能是ScheduledThreadPoolExecutor能够执行延迟任务的原因。
2.  ScheduledThreadPoolExecutor执行的任务一律是ScheduleFuture。ScheduleFuture重写了run方法，对与重复执行的任务在执行完任务后，重新修改执行时间（满足时间需求后延迟队列才会将任务交给工作线程），并重写放入到延迟队列中。
3.  重写execute和submit方法，调整了实现逻辑实现延迟执行任务和重复执行任务的功能

![添加任务和执行任务的流程](/assets/image/ScheduledThreadPoolExecutor.png)


<a id="org113ad30"></a>

# 重要组件


<a id="orgb467cab"></a>

## ScheduledFutureTask

ScheduledFutureTask继承自FutureTask实现RunnableScheduledFuture接口，主要扩展是否是周期任务，以便实现任务能够周期执行的功能。

```java
    private class ScheduledFutureTask<V>
             extends FutureTask<V> implements RunnableScheduledFuture<V> {
    
         // 堆中的任务是需要能够排序的。sequenceNumber用于如果开始执行时间一致的情况下，那么按照入库的先后顺序执行。
         private final long sequenceNumber;
    
         //任务允许开始执行的时间，单位是纳秒
         private long time;
    
         /**
          * 周期任务的时间间隔，单位是纳秒
          */
         private final long period;
    
         //重新入队的任务。之所以使用outerTask代替自身入队是为了支持使用decorateTask进行定制ScheduledFutureTask。
         RunnableScheduledFuture<V> outerTask = this;
    
         /**
          * 延迟队列的位置，方便做取消操作。堆是基于数组实现的，所以其实是数组的下标。
          */
         int heapIndex;
    
         /**
          * Creates a one-shot action with given nanoTime-based trigger time.
          */
         ScheduledFutureTask(Runnable r, V result, long ns) {
             super(r, result);
             this.time = ns;
             this.period = 0;
             this.sequenceNumber = sequencer.getAndIncrement();
         }
    
         /**
          * Creates a periodic action with given nano time and period.
          */
         ScheduledFutureTask(Runnable r, V result, long ns, long period) {
             super(r, result);
             this.time = ns;
             this.period = period;
             this.sequenceNumber = sequencer.getAndIncrement();
         }
    
         /**
          * Creates a one-shot action with given nanoTime-based trigger time.
          */
         ScheduledFutureTask(Callable<V> callable, long ns) {
             super(callable);
             this.time = ns;
             this.period = 0;
             this.sequenceNumber = sequencer.getAndIncrement();
         }
    
         public long getDelay(TimeUnit unit) {
             return unit.convert(time - now(), NANOSECONDS);
         }
    
         /**
          *
          * 堆需要ScheduledFutureTask能够比较大小以便于找到堆中的位置
          * 如果是周期任务，比较的逻辑是先根据任务允许开始执行的时间戳大小排序，其次根据入队先后顺序sequenceNumber排序
          * 如果是非周期任务，那么根据延迟启动的时间戳和现在时间的差值大小排序
          */
         public int compareTo(Delayed other) {
             if (other == this) // compare zero if same object
                 return 0;
             if (other instanceof ScheduledFutureTask) {
                 ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
                 long diff = time - x.time;
                 if (diff < 0)
                     return -1;
                 else if (diff > 0)
                     return 1;
                 else if (sequenceNumber < x.sequenceNumber)
                     return -1;
                 else
                     return 1;
             }
             long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
             return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
         }
    
         /**
          * Returns {@code true} if this is a periodic (not a one-shot) action.
          *
          * @return {@code true} if periodic
          */
         public boolean isPeriodic() {
             return period != 0;
         }
    
         /**
          * 设置下一次开始的时间戳。周期任务在每次执行完毕后需要重新计算下一次开始的时间。
          */
         private void setNextRunTime() {
             long p = period;
             if (p > 0)
                 time += p;
             else
                 time = triggerTime(-p);
         }
         /**
          * 取消任务
          */
         public boolean cancel(boolean mayInterruptIfRunning) {
             boolean cancelled = super.cancel(mayInterruptIfRunning);
             if (cancelled && removeOnCancel && heapIndex >= 0)
                 remove(this);
             return cancelled;
         }
    
         /**
          * 重写FutureTask的run方法，如果不是周期性任务，那么执行FutureTask.run方法，
          * 否则执行FutureTask.runAndReset，并且变更下一次执行的时间，并将任务重新放入到延迟队列中
          * 
          */
         public void run() {
             // 是否是周期性任务
             boolean periodic = isPeriodic();
             // 如果不需要执行，那么取消任务
             if (!canRunInCurrentRunState(periodic))
                 cancel(false);
             // 如果不是周期性任务，执行FutureTask.run方法
             else if (!periodic)
                 ScheduledFutureTask.super.run();
             // 如果是周期性任务，执行FutureTask.runAndReset（保持FutureTask状态为运行中），更新下一次的执行时间，并将任务重新放入到延迟队列中
             else if (ScheduledFutureTask.super.runAndReset()) {
                 setNextRunTime();
                 reExecutePeriodic(outerTask);
             }
         }
     }
```

<a id="orge4cc33d"></a>

## DelayedWorkQueue

DelayedWorkQueue是一个基于堆数据结构实现，只支持存储RunnableScheduledFuture对象的阻塞队列。为了能够快速从DelayedWorkQueue删除任务（为了保证DelayedWorkQueue变动，任何导致DelayedWorkQueue变动的操作，比如添加元素和删除元素都是上锁排他操作，从而导致并发阻塞）减少并发阻塞的时间，RunnableScheduledFuture会持有延迟队列中的位置（heapIndex），使得算法的复杂度从O(n)降低到O(log n)。DelayedWorkQueue为了保证RunnableScheduledFuture中位置的正确，每次堆操作都需要伴随着维护该值的操作。

    
```java
    static class DelayedWorkQueue extends AbstractQueue<Runnable>
        implements BlockingQueue<Runnable> {
    
        // 初始化时RunnableScheduledFuture数组的容量大小
        private static final int INITIAL_CAPACITY = 16;
        // RunnableScheduledFuture数组，堆存储数据的载体
        private RunnableScheduledFuture<?>[] queue =
            new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
        // 可重入锁，提供DelayedWorkQueue出队入队操作时线程安全的保证
        private final ReentrantLock lock = new ReentrantLock();
        // 阻塞队列大小
        private int size = 0;
    
        /**
         * DelayedWorkQueue采用Leader-Follower 模式来处理出队。由于阻塞队列只能提供线程阻塞等待知道阻塞队列中有元素，而延迟队列还需要满足队列的中任务的延迟执行时间得到满足才能执行，所以线程获取到任务后还需要等待延迟时间结束。但是这会导致线程池里很多线程都在等待这个未到达延迟时间要求的任务，然后只有一个线程能够获取，其他线程被唤醒后又继续等待执行下一个任务。
         * Leader-Follower模式会标记一个leader线程，其他都是follower线程。其他线程都是一直等待直到leader线程得到任务，将自身调整成flower线程，并通知另外一个flower线程成为leader线程继续leader线程的工作。
         */
        private Thread leader = null;
    
        /**
         * available帮助实现Leader-Follower模式：1.帮助通知等待的线程选出下一个leader，2. 有leader存在则全部阻塞直到被通知成为leader而被唤醒。
         * 当往阻塞队列中添加任务时，如果被添加的任务是堆顶，那么available会发消息通知。阻塞等待获取任务的线程会被唤醒，如果当前没有leader，被唤醒的线程会成为leader，并根据堆顶任务的延迟时间等待这个数值的时间后执行任务。如果存在则一直等待直到被唤醒成为leader。
         */
        private final Condition available = lock.newCondition();
    
        /**
         * Sets f's heapIndex if it is a ScheduledFutureTask.
         */
        private void setIndex(RunnableScheduledFuture<?> f, int idx) {
            if (f instanceof ScheduledFutureTask)
                ((ScheduledFutureTask)f).heapIndex = idx;
        }
    
        /**
         * 堆的上浮操作
         * 元素添加到堆中的过程就是一个元素加到堆尾然后不断上浮直到找到合适位置的过程。
         * 堆的上浮操作还会用到删除堆元素的场景（符合某种条件下做上浮操作）
         */
        private void siftUp(int k, RunnableScheduledFuture<?> key) {
            // k 是数组下标，在堆添加元素操作里，k是原来堆元素的数量+1
            // k > 0 说明还没到堆顶，在没到堆顶时需要不断和父节点比较以便确认是否是合适的位置。
            while (k > 0) {
                // 父节点的数组下标位置
                int parent = (k - 1) >>> 1;
                // 父节点任务
                RunnableScheduledFuture<?> e = queue[parent];
                // key.compareTo(e) >= 0说明已经找到合适位置，跳出循环
                if (key.compareTo(e) >= 0)
                    break;
                // 到这说明不是合适位置，需要和父节点对换位置
                queue[k] = e;
                setIndex(e, k);
                k = parent;
            }
            // 到这说明找到合适位置（可能就是堆尾，没有做任何移动）
            queue[k] = key;
            setIndex(key, k);
        }
    
        /**
         * 堆的下沉操作
         * 下沉操作就是从指定位置开始不断匹配子节点中最大的节点，如果比子节点最小节点小，那么交换节点位置的过程。
         * 删除元素时，会讲堆尾元素换到被删除的位置，然后做下沉操作。
         */
        private void siftDown(int k, RunnableScheduledFuture<?> key) {
            // 指定位置的左子结点位置
            int half = size >>> 1;
            // 如果指定位置大于或者等于堆的元素个数的一半是，说明再无子节点，可以退出和子节点比较的逻辑
            while (k < half) {
                // 左子节点位置
                int child = (k << 1) + 1;
                // 左自节点位置上的任务
                RunnableScheduledFuture<?> c = queue[child];
                // 右子节点位置
                int right = child + 1;
                // 比较左右自节点，选择其中延迟时间比较小（延迟时间一致则入堆时间早）的节点
                if (right < size && c.compareTo(queue[right]) > 0)
                    c = queue[child = right];
                // key.compareTo(c) <= 0 表示不需要节点位置，退出循环
                if (key.compareTo(c) <= 0)
                    break;
                //需要交换节点，将找到的自节点移动到父节点位置
                queue[k] = c;
                setIndex(c, k);
                k = child;
            }
            // 将任务放入到找到的位置
            queue[k] = key;
            setIndex(key, k);
        }
    
        /**
         * 堆扩大用户存储堆数据的数组
         * 每次扩大都在原来数组大小的基础上扩大一倍
         */
        private void grow() {
            // 当前数组的容量
            int oldCapacity = queue.length;
            // 扩大后的容量，原来的数组右移一位
            int newCapacity = oldCapacity + (oldCapacity >> 1); // grow 50%
            // newCapacity < 0表示超时Integer能够表示的范围了（最高位表示正负值，已经到最高位）
            if (newCapacity < 0) // overflow
                newCapacity = Integer.MAX_VALUE;
            // 组数拷贝，并指向新数组
            queue = Arrays.copyOf(queue, newCapacity);
        }
    
        /**
         * 查找指定任务在数组上的位置，如果找不到则返回-1
         */
        private int indexOf(Object x) {
            if (x != null) {
                // 如果是ScheduledFutureTask则，位置已经在存储在ScheduledFutureTask.heapIndex
                if (x instanceof ScheduledFutureTask) {
                    int i = ((ScheduledFutureTask) x).heapIndex;
                    // 做一下校验，如果headIndex有效则返回
                    // 被删除的ScheduledFutureTask.heapIndex为-1
                    if (i >= 0 && i < size && queue[i] == x)
                        return i;
                // 如果不是ScheduledFutureTask则便利数组查找
                } else {
                    for (int i = 0; i < size; i++)
                        if (x.equals(queue[i]))
                            return i;
                }
            }
            return -1;
        }
    
        // 查找延迟队列是否包含指定的任务
        public boolean contains(Object x) {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                return indexOf(x) != -1;
            } finally {
                lock.unlock();
            }
        }
    
        // 删除任务
        public boolean remove(Object x) {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                // 如果不存在则直接返回false
                int i = indexOf(x);
                if (i < 0)
                    return false;
    
                // 存在则将任务headIndex设置为-1
                setIndex(queue[i], -1);
                int s = --size;
                // 堆尾任务，用来填充到被删除的位置
                RunnableScheduledFuture<?> replacement = queue[s];
                // 堆尾置空（任务被移走）
                queue[s] = null;
                // 被删除的任务不是堆尾任务，需要做下沉操作
                if (s != i) {
                    // 下沉操作
                    siftDown(i, replacement);
                    // 如果下沉操作没有移动位置，则做上浮操作
                    // 父节点的延迟时间（相同则比较入堆先后）一定比替换前的延迟时间小，如果做了交换，那么被替换上来的节点还是会比父节点小。
                    if (queue[i] == replacement)
                        siftUp(i, replacement);
                }
                return true;
            } finally {
                lock.unlock();
            }
        }
    
        // 阻塞队列长度（堆上有任务的数量）
        public int size() {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                return size;
            } finally {
                lock.unlock();
            }
        }
    
        public boolean isEmpty() {
            return size() == 0;
        }
    
        public int remainingCapacity() {
            return Integer.MAX_VALUE;
        }
        // 返回延迟队列第一个元素（不会删除该元素）
        public RunnableScheduledFuture<?> peek() {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                return queue[0];
            } finally {
                lock.unlock();
            }
        }
    
        /**
         * 延迟队列入队操作
         */
        public boolean offer(Runnable x) {
            if (x == null)
                throw new NullPointerException();
            // 入队任务。继承AbstractQueue，但是延迟队列只支持RunnableScheduledFuture
            RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
            // 上锁保证延迟队列线程安全
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                int i = size;
                // 如果已经达到底层数组的容量上线，那么现将底层数组的容量扩大一倍
                if (i >= queue.length)
                    grow();
                // 延迟队列大小加一
                size = i + 1;
                // 如果延迟队列是空队列，那么直接将任务加到堆顶（数组下标为0的位置）
                if (i == 0) {
                    queue[0] = e;
                    setIndex(e, 0);
                // 如果不是空队列，那么被添加到堆的最后一个位置（堆的元素从根节点开始依层次从左到右依次对应到数组上），然后做上浮操作
                } else {
                    siftUp(i, e);
                }
                // 如果添加到堆顶，那么将leader置空并发送信号通知重新选举leader。
                // 所有available.await阻塞的线程在AbstractQueuedSynchronizer里的单向链表上，available.signal会唤醒链表上的第一个节点的线程，而不是唤醒所有的线程，确保只有一个线程会被唤醒成为leader
                // 如果之前已经存在leader，并阻塞在available.await(delay)上，该线程到达时间后依然会被唤醒，但是由于该线程已经不是leader，之后会再次调用available.await()继续等待直到被唤醒成为leader。
                if (queue[0] == e) {
                    leader = null;
                    available.signal();
                }
            } finally {
                lock.unlock();
            }
            return true;
        }
    
        public void put(Runnable e) {
            offer(e);
        }
    
        public boolean add(Runnable e) {
            return offer(e);
        }
    
        public boolean offer(Runnable e, long timeout, TimeUnit unit) {
            return offer(e);
        }
    
        /**
         * 堆出队操作，需要将堆顶任务删除并选取新的堆顶元素
         * @param f 堆顶任务
         */
        private RunnableScheduledFuture<?> finishPoll(RunnableScheduledFuture<?> f) {
            // 堆最后一个任务在数组上的位置
            int s = --size;
            // 堆最后一个任务
            RunnableScheduledFuture<?> x = queue[s];
            // 清空最后一个位置上的任务
            queue[s] = null;
            // s!=0表示堆不只有一个任务，那么将最一个位置的任务放到堆顶，然后做下沉操作
            if (s != 0)
                siftDown(0, x);
            // 被剔除的任务headIndex更新为-1
            setIndex(f, -1);
            return f;
        }
    
        // 延迟队列队首任务出队
        // 如果没有任务或者延迟时间没到，那么返回null，不会阻塞获取任务的线程
        public RunnableScheduledFuture<?> poll() {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                // 如果队首任务为null（空队列）或者堆顶元任务没有到执行的时间，那么直接返回null
                RunnableScheduledFuture<?> first = queue[0];
                if (first == null || first.getDelay(NANOSECONDS) > 0)
                    return null;
                else
                    // 堆顶出队操作
                    return finishPoll(first);
            } finally {
                lock.unlock();
            }
        }
    
        // 从延迟队列中获取任务
        // 如果延迟队列没有任务或者时间没到则阻塞直到获取到任务
        public RunnableScheduledFuture<?> take() throws InterruptedException {
            // 上锁保证延迟队列的线程安全
            final ReentrantLock lock = this.lock;
            lock.lockInterruptibly();
            try {
                // 循环操作直到成功异常退出
                for (;;) {
                    // 延迟队列第一个任务-总是最先出队的任务
                    RunnableScheduledFuture<?> first = queue[0];
                    // 如果延迟队列是空队列则阻塞等待直到通知该等待线程成为leader
                    if (first == null)
                        available.await();
                    else {
                        // 延迟启动的时间
                        long delay = first.getDelay(NANOSECONDS);
                        // delay <= 0表示该任务可以执行，那么做出队操作返回队首任务
                        if (delay <= 0)
                            return finishPoll(first);
                        // 有任务，但是还未到该任务的延迟执行时间所需需要继续等待
    
                        first = null;
                        // 如果已经存在leader那么一直等待直到被唤醒成为leader
                        if (leader != null)
                            available.await();
                        else {
                            // leader不存在则当前线程成为leader
                            Thread thisThread = Thread.currentThread();
                            leader = thisThread;
                            try {
                                // 当前线程等待延迟时间
                                available.awaitNanos(delay);
                            } finally {
                                // 获取到任务后则将leader置空
                                if (leader == thisThread)
                                    leader = null;
                            }
                        }
                    }
                }
            } finally {
                // 成为leader获取到任务后发送信号通知选举下一个leader
                if (leader == null && queue[0] != null)
                    available.signal();
                // 释放锁
                lock.unlock();
            }
        }
    
        /**
         * 获取任务
         * 在获取到任务钱线程或被阻塞直到获取到任务或者等待超时
         * 
         */
        public RunnableScheduledFuture<?> poll(long timeout, TimeUnit unit)
            throws InterruptedException {
            long nanos = unit.toNanos(timeout);
            final ReentrantLock lock = this.lock;
            lock.lockInterruptibly();
            try {
                for (;;) {
                    // 延迟队列第一个任务-总是最先出队的任务
                    RunnableScheduledFuture<?> first = queue[0];
                    if (first == null) {
                        // 如果队列是空队列并且已经过了指定的等待时间，那么返回null
                        if (nanos <= 0)
                            return null;
                        // 等待指定时间
                        else
                            nanos = available.awaitNanos(nanos);
                    } else {
                        // 距离任务开始执行的时间
                        long delay = first.getDelay(NANOSECONDS);
                        // delay <= 0表示任务可以开始执行，将任务出队
                        if (delay <= 0)
                            return finishPoll(first);
                        // 已经过超时时间，返回null
                        if (nanos <= 0)
                            return null;
                        //后续线程可能会开始等待。由于first任务很可能由于其他功能导出队或者删除，为了能够让first指向的任务被快速回收，将first指向置为null
                        first = null;                   
                        if (nanos < delay || leader != null)
                            nanos = available.awaitNanos(nanos);
                        else {
                            Thread thisThread = Thread.currentThread();
                            // 当前线程设置为leader。除非队首元素发生变化，否则当前线程应该最早获得任务。设置为leader降低其他线程被无效唤醒的概率
                            leader = thisThread;
                            try {
                                // 计算nanos，供下一次进入for循环逻辑使用
                                long timeLeft = available.awaitNanos(delay);
                                nanos -= delay - timeLeft;
                            } finally {
                                // 没有获取到任务，重新抢队首元素，将leader设置为null
                                if (leader == thisThread)
                                    leader = null;
                            }
                        }
                    }
                }
            } finally {
                // leader == null && queue[0] != null 表示需要选举一个leader
                // 当前线程已经获取到任务并且去处理任务了
                if (leader == null && queue[0] != null)
                    available.signal();
                lock.unlock();
            }
        }
    
        // 清空阻塞队列上的任务
        public void clear() {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                for (int i = 0; i < size; i++) {
                    RunnableScheduledFuture<?> t = queue[i];
                    if (t != null) {
                        queue[i] = null;
                        setIndex(t, -1);
                    }
                }
                size = 0;
            } finally {
                lock.unlock();
            }
        }
    }
```

<a id="org92abd92"></a>

# 主要源码

```java
    /**
      * ScheduledThreadPoolExecutor执行任务的方法主体
      * @param task 任务
      */
     private void delayedExecute(RunnableScheduledFuture<?> task) {
         // 如果线程池已经被关闭，那么拒绝任务
         if (isShutdown())
             reject(task);
         else {
             // 将任务添加到延迟队列中
             super.getQueue().add(task);
             // 线程关闭并且是周期任务并且配置了取消任务立即删除，那么从队列中删除并且取消该任务。
             if (isShutdown() &&
                 !canRunInCurrentRunState(task.isPeriodic()) &&
                 remove(task))
                 task.cancel(false);
             // 没有工作线程或这工作线程少于核心线程数数时启动工作线程执行任务
             else
                 ensurePrestart();
         }
     }
    
     /**
      * 执行周日任务的方法主体。
      *
      * @param task 任务
      */
     void reExecutePeriodic(RunnableScheduledFuture<?> task) {
         // 是否是周期任务。如果不是则不做处理
         if (canRunInCurrentRunState(true)) {
             super.getQueue().add(task);
             if (!canRunInCurrentRunState(true) && remove(task))
                 task.cancel(false);
             else
                 ensurePrestart();
         }
     }
    
     /**
      * 将阻塞队列中的任务清空
      */
     @Override void onShutdown() {
         BlockingQueue<Runnable> q = super.getQueue();
         boolean keepDelayed =
             getExecuteExistingDelayedTasksAfterShutdownPolicy();
         boolean keepPeriodic =
             getContinueExistingPeriodicTasksAfterShutdownPolicy();
         if (!keepDelayed && !keepPeriodic) {
             for (Object e : q.toArray())
                 if (e instanceof RunnableScheduledFuture<?>)
                     ((RunnableScheduledFuture<?>) e).cancel(false);
             q.clear();
         }
         else {
             // Traverse snapshot to avoid iterator exceptions
             for (Object e : q.toArray()) {
                 if (e instanceof RunnableScheduledFuture) {
                     RunnableScheduledFuture<?> t =
                         (RunnableScheduledFuture<?>)e;
                     if ((t.isPeriodic() ? !keepPeriodic : !keepDelayed) ||
                         t.isCancelled()) { // also remove if already cancelled
                         if (q.remove(t))
                             t.cancel(false);
                     }
                 }
             }
         }
         tryTerminate();
     }
    
     /**
      * 让子类可以通过重写该方法自定义RunnableScheduledFuture
      *
      * @param runnable the submitted Runnable
      * @param task the task created to execute the runnable
      * @param <V> the type of the task's result
      * @return a task that can execute the runnable
      * @since 1.6
      */
     protected <V> RunnableScheduledFuture<V> decorateTask(
         Runnable runnable, RunnableScheduledFuture<V> task) {
         return task;
     }
    
     /**
      * 让子类可以通过重写该方法自定义RunnableScheduledFuture
      *
      * @param callable the submitted Callable
      * @param task the task created to execute the callable
      * @param <V> the type of the task's result
      * @return a task that can execute the callable
      * @since 1.6
      */
     protected <V> RunnableScheduledFuture<V> decorateTask(
         Callable<V> callable, RunnableScheduledFuture<V> task) {
         return task;
     }
    
     /**
      * 计算距离开始执行任务的时间
      */
     private long triggerTime(long delay, TimeUnit unit) {
         return triggerTime(unit.toNanos((delay < 0) ? 0 : delay));
     }
    
     /**
      * 计算距离开始执行任务的时间
      */
     long triggerTime(long delay) {
         return now() +
             ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
     }
    
     /**
      * 限制延迟时间在Long.MAX_VALUE以内
      */
     private long overflowFree(long delay) {
         Delayed head = (Delayed) super.getQueue().peek();
         if (head != null) {
             long headDelay = head.getDelay(NANOSECONDS);
             if (headDelay < 0 && (delay - headDelay < 0))
                 delay = Long.MAX_VALUE + headDelay;
         }
         return delay;
     }
    
     /**
      * @throws RejectedExecutionException {@inheritDoc}
      * @throws NullPointerException       {@inheritDoc}
      */
     public ScheduledFuture<?> schedule(Runnable command,
                                        long delay,
                                        TimeUnit unit) {
         if (command == null || unit == null)
             throw new NullPointerException();
         RunnableScheduledFuture<?> t = decorateTask(command,
             new ScheduledFutureTask<Void>(command, null,
                                           triggerTime(delay, unit)));
         delayedExecute(t);
         return t;
     }
    
     /**
      * @throws RejectedExecutionException {@inheritDoc}
      * @throws NullPointerException       {@inheritDoc}
      */
     public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                            long delay,
                                            TimeUnit unit) {
         if (callable == null || unit == null)
             throw new NullPointerException();
         RunnableScheduledFuture<V> t = decorateTask(callable,
             new ScheduledFutureTask<V>(callable,
                                        triggerTime(delay, unit)));
         delayedExecute(t);
         return t;
     }
    
     /**
      * @throws RejectedExecutionException {@inheritDoc}
      * @throws NullPointerException       {@inheritDoc}
      * @throws IllegalArgumentException   {@inheritDoc}
      */
     public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                   long initialDelay,
                                                   long period,
                                                   TimeUnit unit) {
         if (command == null || unit == null)
             throw new NullPointerException();
         if (period <= 0)
             throw new IllegalArgumentException();
         ScheduledFutureTask<Void> sft =
             new ScheduledFutureTask<Void>(command,
                                           null,
                                           triggerTime(initialDelay, unit),
                                           unit.toNanos(period));
         RunnableScheduledFuture<Void> t = decorateTask(command, sft);
         sft.outerTask = t;
         delayedExecute(t);
         return t;
     }
    
     /**
      * @throws RejectedExecutionException {@inheritDoc}
      * @throws NullPointerException       {@inheritDoc}
      * @throws IllegalArgumentException   {@inheritDoc}
      */
     public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                      long initialDelay,
                                                      long delay,
                                                      TimeUnit unit) {
         if (command == null || unit == null)
             throw new NullPointerException();
         if (delay <= 0)
             throw new IllegalArgumentException();
         ScheduledFutureTask<Void> sft =
             new ScheduledFutureTask<Void>(command,
                                           null,
                                           triggerTime(initialDelay, unit),
                                           unit.toNanos(-delay));
         RunnableScheduledFuture<Void> t = decorateTask(command, sft);
         sft.outerTask = t;
         delayedExecute(t);
         return t;
     }
```

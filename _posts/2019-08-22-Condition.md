---
title: Java Condition学习笔记
tags: [Java, JUC, 源码, 多线程, 锁]
layout: post
author: wjf
---

<div id="table-of-contents">
<h2>目录</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#orgca2d90f">作用</a></li>
<li><a href="#orga3c7b61">接口定义</a></li>
<li><a href="#org0226b88">ConditionObject 实现类</a>
<ul>
<li><a href="#orgc709f74">AbstractQueuedSynchronizer</a></li>
<li><a href="#orgc8ab98a">Node</a></li>
<li><a href="#org86519a2">ConditionObject</a></li>
</ul>
</li>
</ul>
</div>
</div>


<a id="orgca2d90f"></a>

# 作用

Condition 是 jdk1.5 引入的用于代替监视器方法，比如 wait，notify，notifyAll，提供更加易用和灵活的同步阻塞方法。
Condition 翻译暂时翻译为竞态条件，用作是在其他线程通知唤醒之前阻塞一个线程。在多线程环境下如果出现资源竞争问题，需要保证多线程操作同一资源时，需要顺序执行，以免出现线程安全问题。
线程在使用竞态条件时的阻塞效果和 wait()方法类似，但是本质上静态条件是一种锁。等待满足一种静态条件其实是通过静态条件的 newCondition()方法初始化初始化一个特殊的锁。

一下是一个竞态条件的例子：

     class BoundedBuffer {
      final Lock lock = new ReentrantLock();
      final Condition notFull  = lock.newCondition();
      final Condition notEmpty = lock.newCondition();
    
      final Object[] items = new Object[100];
      int putptr, takeptr, count;
    
      public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
          while (count == items.length)
            notFull.await();
          items[putptr] = x;
          if (++putptr == items.length) putptr = 0;
          ++count;
          notEmpty.signal();
        } finally {
          lock.unlock();
        }
      }
    
      public Object take() throws InterruptedException {
        lock.lock();
        try {
          while (count == 0)
            notEmpty.await();
          Object x = items[takeptr];
          if (++takeptr == items.length) takeptr = 0;
          --count;
          notFull.signal();
          return x;
        } finally {
          lock.unlock();
        }
      }
    }

上面是一个包含 put(存放一个元素)和 take（取走一个元素）并且由有边界的容器的例子。如果容器已经达到容量上限，那么调用 put 方法的线程会被阻塞。如果有其他线程成功取走元素时，调用 put 被阻塞的线程会被唤醒。
竞态条件比监视器的阻塞和唤醒增加了一些功能，比如可以指定唤醒的顺序和通知时不需要持有锁。
竞态条件的实例也是一个普通的 Java 对象，具备 Object.wait 方法和 notify 方法，并且也可以单独使用这些监视器方法。但是由于这些监视器方法和静态条件本身设计的方法是相互独立的，所以除非是实现类有特殊的设计，否则建议不要使用静态条件的监视器方法，以免混淆。
由于平台本身可能会发生[虚假唤醒（Spurious Wakeup)](https://en.wikipedia.org/wiki/Spurious_wakeup),所以在设计等待条件时，需要放在一个循环体中判断。如果放在 if 条件判断中语句中，此时发生虚假唤醒，那么条件不通过，程序将不按照预定的设计走。
竞态条件的等待方法（interruptible，non-interruptible，timed）在各个平台上的实现和性能上的表现可能会不一致。而且在某些平台上很难提供按顺序唤醒的特性，更有甚者线程中断的功能也不是在所有平台上都能够让线程立即暂停的。
通常情况下不需要竞态条件的实现类实现顺序唤醒的功能，也没有必要让线程立即终止。但是要求实现类明确指出这些功能确切的表现形式，以免发生误解。中断操作通常意味着终止一个线程。但是如果中断操作发生在线程正在处理一个任务的过程中，那么线程无法被记录阻塞。实现类应该把这种行为描述清楚。


<a id="orga3c7b61"></a>

# 接口定义

    /**
       * Causes the current thread to wait until it is signalled or
       * {@linkplain Thread#interrupt interrupted}.
       *
       * 在得到通知唤醒或者线程中断之前一组阻塞等待。
       *
       * <p>The lock associated with this {@code Condition} is atomically
       * released and the current thread becomes disabled for thread scheduling
       * purposes and lies dormant until <em>one</em> of four things happens:
       * <ul>
       * <li>Some other thread invokes the {@link #signal} method for this
       * {@code Condition} and the current thread happens to be chosen as the
       * thread to be awakened; or
       * <li>Some other thread invokes the {@link #signalAll} method for this
       * {@code Condition}; or
       * <li>Some other thread {@linkplain Thread#interrupt interrupts} the
       * current thread, and interruption of thread suspension is supported; or
       * <li>A &quot;<em>spurious wakeup</em>&quot; occurs.
       * </ul>
       *
       * 如果有以下的一种情况发生：1）某个线程唤醒当前线程（Condition.signal），并且当前线程被选择唤醒；
       * 2）某个线程唤醒所有等待的线程（Condition.signalAll）；
       * 3）发生了一次虚假唤醒（spurious wakeup）；
       *
       * <p>In all cases, before this method can return the current thread must
       * re-acquire the lock associated with this condition. When the
       * thread returns it is <em>guaranteed</em> to hold this lock.
       *
       * 如果线程由于调用阻塞方法被阻塞了，希望在被唤醒之后执行，首先必须要保证线程能够获得竞态条件关联的锁。
       * 正如 BoundedBuffer 中的例子，在 put 时，先获得锁。如果队列已经达到容量上限，那么调用 notFull.await()。
       * notFull.await()方法会让当前线程阻塞并原子释放对象锁。
       *
       * <p>If the current thread:
       * <ul>
       * <li>has its interrupted status set on entry to this method; or
       * <li>is {@linkplain Thread#interrupt interrupted} while waiting
       * and interruption of thread suspension is supported,
       * </ul>
       * then {@link InterruptedException} is thrown and the current thread's
       * interrupted status is cleared. It is not specified, in the first
       * case, whether or not the test for interruption occurs before the lock
       * is released.
       *
       * 如果线程在等待(waiting)或阻塞（Blocked）状态（Bloked）时，线程被设置中断状态为被中断（interrupted），
       * 那么会抛出 InterruptedException 异常，并且被中断（interrupted）标记被清除。
       * 第一次发生中断操作之后，线程持有的锁也会被原子释放。
       *
       * <p><b>Implementation Considerations</b>
       *
       * <p>The current thread is assumed to hold the lock associated with this
       * {@code Condition} when this method is called.
       * It is up to the implementation to determine if this is
       * the case and if not, how to respond. Typically, an exception will be
       * thrown (such as {@link IllegalMonitorStateException}) and the
       * implementation must document that fact.
       *
       * <p>An implementation can favor responding to an interrupt over normal
       * method return in response to a signal. In that case the implementation
       * must ensure that the signal is redirected to another waiting thread, if
       * there is one.
       *
       * 注意事项：
       * 线程持有竞态条件相关锁的情况下才调用竞态条件的方法。
       * 如果未持有锁的情况下调用竞态条件的方法，那么可以抛出一个 IllegalMonitorStateException 异常。并且接口文档上需要说明。
       * 如果实现类通过中断来唤醒其他线程，那么请确保 signal 也能唤醒其他线程。
       *
       * @throws InterruptedException if the current thread is interrupted
       *         (and interruption of thread suspension is supported)
       */
      void await() throws InterruptedException;
    
      /**
       * Causes the current thread to wait until it is signalled.
       *
       * 线程在被唤醒之前一直等待。正如方法名称所示，该方法忽略线程中断条件校验。
       *
       */
      void awaitUninterruptibly();
    
      /**
       * Causes the current thread to wait until it is signalled or interrupted,
       * or the specified waiting time elapses.
       *
       * 在线程被中断或者被唤醒或者超过等待时间前线程一直处于等待状态
       *
       *
       * @param nanosTimeout the maximum time to wait, in nanoseconds
       * @return an estimate of the {@code nanosTimeout} value minus
       *         the time spent waiting upon return from this method.
       *         A positive value may be used as the argument to a
       *         subsequent call to this method to finish waiting out
       *         the desired time.  A value less than or equal to zero
       *         indicates that no time remains.
       * @throws InterruptedException if the current thread is interrupted
       *         (and interruption of thread suspension is supported)
       */
      long awaitNanos(long nanosTimeout) throws InterruptedException;
    
      /**
       * Causes the current thread to wait until it is signalled or interrupted,
       * or the specified waiting time elapses. This method is behaviorally
       * equivalent to:
       *
       * 同 awaitNanos 方法。
       *  <pre> {@code awaitNanos(unit.toNanos(time)) > 0}</pre>
       *
       * @param time the maximum time to wait
       * @param unit the time unit of the {@code time} argument
       * @return {@code false} if the waiting time detectably elapsed
       *         before return from the method, else {@code true}
       * @throws InterruptedException if the current thread is interrupted
       *         (and interruption of thread suspension is supported)
       */
      boolean await(long time, TimeUnit unit) throws InterruptedException;
    
      /*
       * 通 awaitNanos 方法。awaitNanos 是给定距离现在的纳秒，awaitUntil 是给定截止日期。
       */
      boolean awaitUntil(Date deadline) throws InterruptedException;
    
      /**
       * Wakes up one waiting thread.
       *
       * 唤醒其他等待线程
       *
       * <p>If any threads are waiting on this condition then one
       * is selected for waking up. That thread must then re-acquire the
       * lock before returning from {@code await}.
       *
       * 被唤醒的线程在调用 await 方法前必须持有锁。
       *
       * <p><b>Implementation Considerations</b>
       *
       * <p>An implementation may (and typically does) require that the
       * current thread hold the lock associated with this {@code
       * Condition} when this method is called. Implementations must
       * document this precondition and any actions taken if the lock is
       * not held. Typically, an exception such as {@link
       * IllegalMonitorStateException} will be thrown.
       *
       * 注意事项：
       * 实现类在调用 signal 方法前必须先持有锁。明确明确说明如果未持有锁但是却调用了 signal 方法，
       * 会抛出 IllegalMonitorStateException 异常
       */
      void signal();
    
      /**
       * Wakes up all waiting threads.
       *
       * 唤醒所有的等待线程。
       * 其他注意事项通 signal 类似。
       *
       * <p>If any threads are waiting on this condition then they are
       * all woken up. Each thread must re-acquire the lock before it can
       * return from {@code await}.
       *
       * <p><b>Implementation Considerations</b>
       *
       * <p>An implementation may (and typically does) require that the
       * current thread hold the lock associated with this {@code
       * Condition} when this method is called. Implementations must
       * document this precondition and any actions taken if the lock is
       * not held. Typically, an exception such as {@link
       * IllegalMonitorStateException} will be thrown.
       */
      void signalAll();


<a id="org0226b88"></a>

# ConditionObject 实现类


<a id="orgc709f74"></a>

## AbstractQueuedSynchronizer

一个基于先进先出的等待队列实现的在多线程环境下，提供线程阻塞和取消阻塞的同步器。


<a id="orgc8ab98a"></a>

## Node

AbstractQueuedSynchronizer.ConditionObject 依赖 AbstractQueuedSynchronizer.Node。在介绍 ConditionObject 只来，先来看一下 Node 的实现。
Node 是表示等待队列的元素，本身有上一个节点的指向和下一个节点的指向，多个节点构成一个链表特性的等待队列。
等待队列是 CLH（Craig，Landin，and Hagersten）锁队列的一种变种。CLH 锁是一队列（本身由一组节点组成的链表）锁，能够确保无饥饿和先来先得的公平性，一般用做自旋锁。
AbstractQueuedSynchronizer 中，由 Node 组成的队列锁并非用作自旋锁，而是做成了同步机制。实现同步机制的策略和自旋锁的策略类似，也是通过控制前一个节点的信息来实现。节点中（Node）有个属性“status”跟踪线程的阻塞状态，会随着线程的阻塞状态变更而变更。如果当前节点的线程已经执行完毕或则已经可以释放共享资源时，会将节点释放，即链路第一个节点出队列。链路第一个节点出队列时，会通知后续的节点。后续节点在得到其对应的节点通知前，一直在试图获取是否可以往下执行的校验（循环校验是否满足往下执行的条件，而并非是编程阻塞状态）。如此可达到让线程按照顺序自旋阻塞和按照顺序先后执行。
由于线程在获取节点时可能和其他线程一起竞争同一顺序的节点，可能会导致获取失败继续等待。
将一个元素添加到 CLH 队列中只需要对队尾元素做修改，将队尾元素的下一个元素指向到新元素中，并将队尾的引用指向新增加的元素（具体的操作会比这复杂）。这个操作需要保证原子操作。同样的出队列也只需要操作队列第一个元素，将队列的第一个元素指向到之前第一个元素的下一个元素。该操作也要保证是原子性。并且由于等待时间过长或者线程中断等因素，还需要考虑操作是否真正成功。
Node 设计的有许多种状态，比如 CANCELLED（取消状态，表示节点关联的线程被终止了），SIGNAL（通知，表示竞争到节点的线程被允许执行了），CONDITION（等待，表示线程在等待合适的时机执行），PROPAGATE（传播，在共享模式中（多个线程共享一个节点）中，下一个贡献节点可以等待转播）。如果节点被设置成 CANCELLED 状态，那么上一个节点引用指向该节点的需要重新指向一个未被职位取消置为 CANCELLED 的节点。


<a id="org86519a2"></a>

## ConditionObject

ConditionObject 是一个单向列表的条件队列。结合 Node 实现的单向链表，为 AbstractQueuedSynchronizer 提供阻塞（await，awaitNanos，awaitUntil 等）和取消阻塞（doSignal，doSignalAll）功能。

    public class ConditionObject implements Condition, java.io.Serializable {
          private static final long serialVersionUID = 1173984872572414699L;
          /**
           * 队列的的第一个 Node 元素。通知方法（signal 和 signalAll）一般都只针对队列的第一个元素的。
           * 通知方法会告通知下一个元素。
           *
           */
          private transient Node firstWaiter;
    
          /**
           * 队列的最后一个元素。入队操作（addConditionWaiter）需要知道在操作前队列最后一个元素的位置，
           * 并添加一个等待元素到队尾，并将队尾引用指向新增加的元素。
           */
          private transient Node lastWaiter;
    
          /**
           * Creates a new {@code ConditionObject} instance.
           */
          public ConditionObject() { }
    
          // Internal methods
    
          /**
           * 添加一个元素到队尾。此时新增加的元素关联的线程是当前线程。
           * @return its new wait node
           */
          private Node addConditionWaiter() {
              Node t = lastWaiter;
              // If lastWaiter is cancelled, clean out.
              if (t != null && t.waitStatus != Node.CONDITION) {
                  unlinkCancelledWaiters();
                  t = lastWaiter;
              }
              Node node = new Node(Thread.currentThread(), Node.CONDITION);
              if (t == null)
                  firstWaiter = node;
              else
                  t.nextWaiter = node;
              lastWaiter = node;
              return node;
          }
    
          /**
           * 将 Node 链表（ConditionObject 对象持有的 Node 链表）中指定的元素（一般是链表中第一个元素）
           * 依次将 Node 的状态从 CONDITION（-2）变更成 0，
           * 即 Node 从等待条件状态变成条件已经满足的状态，
           * 并且转移到同步队列中（AbstractQueuedSynchronizer 持有的 Node 链表）。
           * @param first (non-null) the first node on condition queue
           */
          private void doSignal(Node first) {
              do {
                  if ( (firstWaiter = first.nextWaiter) == null)
                      lastWaiter = null;
                  first.nextWaiter = null;
              } while (!transferForSignal(first) &&
                       (first = firstWaiter) != null);
          }
    
          /**
           * 将 ConditionObject 对象持有的 Node 链表上所有的元素都移到 AbstractQueuedSynchronizer 对象
           * 持有的 Node 元素链表上。
           * @param first (non-null) the first node on condition queue
           */
          private void doSignalAll(Node first) {
              lastWaiter = firstWaiter = null;
              do {
                  Node next = first.nextWaiter;
                  first.nextWaiter = null;
                  transferForSignal(first);
                  first = next;
              } while (first != null);
          }
    
          /**
           * 如果链表中的元素的状态不是等待条件的（Condition）的状态，将元素剔除出链表。
           */
          private void unlinkCancelledWaiters() {
              Node t = firstWaiter;
              Node trail = null;
              while (t != null) {
                  Node next = t.nextWaiter;
                  if (t.waitStatus != Node.CONDITION) {
                      t.nextWaiter = null;
                      if (trail == null)
                          firstWaiter = next;
                      else
                          trail.nextWaiter = next;
                      if (next == null)
                          lastWaiter = trail;
                  }
                  else
                      trail = t;
                  t = next;
              }
          }
    
          // public methods
    
          /**
           * 将条件队列中的第一个元素移到 将 ConditionObject 对象持有的 Node 链表的第一个元素移到
           * AbstractQueuedSynchronizer 对象持有的 Node 链表上
           *
           * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
           *         returns {@code false}
           */
          public final void signal() {
              if (!isHeldExclusively())
                  throw new IllegalMonitorStateException();
              Node first = firstWaiter;
              if (first != null)
                  doSignal(first);
          }
    
          /**
           * 和 signal 类似，只是将所有的链表元素都迁移过去。
           *
           * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
           *         returns {@code false}
           */
          public final void signalAll() {
              if (!isHeldExclusively())
                  throw new IllegalMonitorStateException();
              Node first = firstWaiter;
              if (first != null)
                  doSignalAll(first);
          }
    
          /**
           * 实现了一个可以响应中断的等待某个条件的才执行的等待方法。
           * 如果 Node 元素关联的线程被中断，那么抛出 InterruptedException。
           * 保存锁的状态。后续执行释放锁时需要比对锁的状态，用于判断是否释放成功；
           * 通过 LockSupport.park(this)的方式阻塞线程。
           * 如果线程等待过程中（LockSupport.park(this)）被中断，抛出线程中断异常（InterruptedException）
           */
          public final void await() throws InterruptedException {
              if (Thread.interrupted())
                  throw new InterruptedException();
              Node node = addConditionWaiter();
              int savedState = fullyRelease(node);
              int interruptMode = 0;
              while (!isOnSyncQueue(node)) {
                  LockSupport.park(this);
                  if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                      break;
              }
              if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                  interruptMode = REINTERRUPT;
              if (node.nextWaiter != null) // clean up if cancelled
                  unlinkCancelledWaiters();
              if (interruptMode != 0)
                  reportInterruptAfterWait(interruptMode);
          }
    
          /**
           * 同await方法，并增加了等待超时时间。
           */
          public final long awaitNanos(long nanosTimeout)
                  throws InterruptedException {
              if (Thread.interrupted())
                  throw new InterruptedException();
              Node node = addConditionWaiter();
              int savedState = fullyRelease(node);
              final long deadline = System.nanoTime() + nanosTimeout;
              int interruptMode = 0;
              while (!isOnSyncQueue(node)) {
                  if (nanosTimeout <= 0L) {
                      transferAfterCancelledWait(node);
                      break;
                  }
                  if (nanosTimeout >= spinForTimeoutThreshold)
                      LockSupport.parkNanos(this, nanosTimeout);
                  if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                      break;
                  nanosTimeout = deadline - System.nanoTime();
              }
              if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                  interruptMode = REINTERRUPT;
              if (node.nextWaiter != null)
                  unlinkCancelledWaiters();
              if (interruptMode != 0)
                  reportInterruptAfterWait(interruptMode);
              return deadline - System.nanoTime();
          }
    
          /**
           * 同await方法，增加超时的具体时间
           */
          public final boolean awaitUntil(Date deadline)
                  throws InterruptedException {
              long abstime = deadline.getTime();
              if (Thread.interrupted())
                  throw new InterruptedException();
              Node node = addConditionWaiter();
              int savedState = fullyRelease(node);
              boolean timedout = false;
              int interruptMode = 0;
              while (!isOnSyncQueue(node)) {
                  if (System.currentTimeMillis() > abstime) {
                      timedout = transferAfterCancelledWait(node);
                      break;
                  }
                  LockSupport.parkUntil(this, abstime);
                  if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                      break;
              }
              if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                  interruptMode = REINTERRUPT;
              if (node.nextWaiter != null)
                  unlinkCancelledWaiters();
              if (interruptMode != 0)
                  reportInterruptAfterWait(interruptMode);
              return !timedout;
          }
      }


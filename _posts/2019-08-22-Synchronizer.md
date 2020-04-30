---
title: Java 同步器学习笔记
tags: [Java, JUC, 源码, 多线程, 锁]
layout: post
author: wjf
---
<div id="table-of-contents">
<h2>目录</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#org2d4118d">同步器</a></li>
<li><a href="#org01ae0de">AbstractOwnableSynchronizer 类介绍</a></li>
<li><a href="#org2386d90">AbstractQueuedSynchronizer</a></li>
<li><a href="#org7ebd538">AbstractQueuedSynchronizer 子类实现的方法介绍</a>
<ul>
<li><a href="#org89fa8d1">tryAcquire</a></li>
<li><a href="#org189c148">tryRelease</a></li>
</ul>
</li>
</ul>
</div>
</div>


<a id="org2d4118d"></a>

# 同步器

同步器为多线程环境下多个线程访问修改同一个资源时，提供线程同步、互斥、等待等一系列功能。JDK 提供的主要同步器类继承如下：
![FairSync类继承关系](/assets/image/FairSync_diagram.png)

<a id="org01ae0de"></a>

# AbstractOwnableSynchronizer 类介绍

AbstractOwnableSynchronizer 类最基本的功能是够被一个线程独占（类名的意义）。AbstractOwnableSynchronizer 有个私有属性 exclusiveOwnerThread，是独占线程的引用，并提供该线程的 get，set 方法。


<a id="org2386d90"></a>

# AbstractQueuedSynchronizer

AbstractQueuedSynchronizer 是同步器的重要实现。AbstractQueuedSynchronizer 是基于先进先出（FIFO）的等待队列实现的多线程间的同步工具。AbstractQueuedSynchronizer 等待队列上的线程都会有一个对应的需要原子操作的 int 类型的数值表示线程当前的状态。AbstractQueuedSynchronizer 包含两部分内容，一部分是要求子类实现改变状态的方法，即需要子类实现获取锁和释放锁的逻辑；另一部分是实现线程进度等待队列自旋和阻塞，通知唤醒，出等待队列等功能。

由于 AbstractQueuedSynchronizer 本身提供了大量的 public 方法，并且方法实现的是线程进入等待队列等保证线程同步的功能，并不一定适合直接暴露给其他类直接使用，所以要求子类是一个 non-public 类型的内部类。

AbstractQueuedSynchronizer 提供了两种同步策略，分别是独占模式和共享模式。独占模式只允许一个线程获取锁。如果当前已经有一个线程获取锁了，那么其他线程获取锁时，都进入等待队列。共享模式允许多个线程同时获取锁，但是不保证线程获取锁时一定能够成功。AbstractQueuedSynchronizer 本身是没有获取锁和释放锁的具体实现，但是为了考虑到默写情况下，子类可能只需要提供一种策略模式，所以定义的获取锁和释放锁的方法时 protected，并且方法体只是抛出 UnsupportedOperationException 异常。但是 AbstractQueuedSynchronizer 本身是负责维护等待队列和通知唤醒，所以一旦线程在共享模式下获取锁，AbstractQueuedSynchronizer 需要判断下一个等待线程是否需要也是需要获取锁。AbstractQueuedSynchronizer 只维护一个双向链表作为等待队列，所以不同线程使用不同的同步策略时，他们都位于一个等待队列上。子类如果只需要一种同步策略，那么只需要实现一种同步策略。

锁定义了获取竞态条件（Condition）的接口，并且锁一般都依赖同步器实现许多功能，可能是为了方便锁的实现，所以在 AbstractQueuedSynchronizer 中有一个竞态条件的实现类 ConditionObject 的内部类。

AbstractQueuedSynchronizer 要求子类实现的内容有

-   tryAcquire 获取独占锁
-   tryRelease 释放独占锁锁
-   tryAcquireShared 获取共享锁
-   tryReleaseShared 释放共享锁
-   isHeldExclusively 是否是当前线程在持有同步器的独占锁

在使用上述方法时，必须保证线程安全，并且这个方法的执行时间应该要尽可能的短，并且要求不会被阻塞。

线程被AbstractQueuedSynchronizer 丢入等待队列的操作并不是一个原子操作，能够成功进入等待队列的核心逻辑如下：

    while (!tryAcquire(arg)) {
        //如果还没有进队列则进入队列
        //根据需要阻塞当前线程
    }
    
    while (!tryRelease(arg)) {
        //将队列第一个元素绑定的线程唤醒 LockSupport.unpack
    }

由于在入队前会处理许多判断校验等工作，所以如果不做特殊处理，可能会发生后续进入的线程（进入到等待队列前进来）会比先进来的线程更早的进入到等待队列中。不做特殊处理的方式是非公平锁，而保证先进来的线程会先入队到等待队列中的是公平锁。

AbstractQueuedSynchronizer 提供了一个基础的但是搞笑并具有扩展性的同步器功能，但是 AbstractQueuedSynchronizer 并不是一个完整的同步器。AbstractQueuedSynchronizer 的一部分功能比如 tryAcquire 等依赖子类实现。

以下是一个文档中完整同步器的例子：

     class Mutex implements Lock, java.io.Serializable {
    
      // Our internal helper class
      private static class Sync extends AbstractQueuedSynchronizer {
        // Report whether in locked state
        protected boolean isHeldExclusively() {
          return getState() == 1;
        }
    
        // Acquire the lock if state is zero
        public boolean tryAcquire(int acquires) {
          assert acquires == 1; // Otherwise unused
          if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
          }
          return false;
        }
    
        // Release the lock by setting state to zero
        protected boolean tryRelease(int releases) {
          assert releases == 1; // Otherwise unused
          if (getState() == 0) throw new IllegalMonitorStateException();
          setExclusiveOwnerThread(null);
          setState(0);
          return true;
        }
    
        // Provide a Condition
        Condition newCondition() { return new ConditionObject(); }
    
        // Deserialize properly
        private void readObject(ObjectInputStream s)
            throws IOException, ClassNotFoundException {
          s.defaultReadObject();
          setState(0); // reset to unlocked state
        }
      }
    
      // The sync object does all the hard work. We just forward to it.
      private final Sync sync = new Sync();
    
      public void lock()                { sync.acquire(1); }
      public boolean tryLock()          { return sync.tryAcquire(1); }
      public void unlock()              { sync.release(1); }
      public Condition newCondition()   { return sync.newCondition(); }
      public boolean isLocked()         { return sync.isHeldExclusively(); }
      public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
      public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
      }
      public boolean tryLock(long timeout, TimeUnit unit)
          throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
      }
    }

-   特别提醒：Mutex 不能直接继承 AbstractQueuedSynchronizer，因为 AbstractQueuedSynchronizer 有太多的同步器功能相关的 public 方法，不适合暴露出去。所以 Mutex 有一个内部类 Sync，继承 AbstractQueuedSynchronizer，实现 tryAcquire 等 AbstractQueuedSynchronizer 未提供的功能。

AbsctractQueuedSynchronizer 属性与方法介绍：

    public abstract class AbstractQueuedSynchronizer
      extends AbstractOwnableSynchronizer
      implements java.io.Serializable {
    
      private static final long serialVersionUID = 7373984972572414691L;
    
      /**
       * 创建一个同步状态为 0 (state=0)的同步器
       */
      protected AbstractQueuedSynchronizer() { }
    
      /**
       *
       * Node 是构成同步器等待队列的基础数据结构，详情参考：Condition 笔记关于 Node 部分的说明
       *
       */
      static final class Node {...}
    
      /**
       * Node 组成的双向链表的第一个元素
       * 初始化 AbstractQueuedSynchronizer 时，head 指向 null。如果需要初始化 AbstractQueuedSynchronizer 时，构
       * 建 head 元素，那么通过 setHead 方法设置 head 的值。并且保证 waitStatus 不能是取消(CANCELLED)的状态。
       */
      private transient volatile Node head;
    
      /**
       * Node 组成的双向链表的最后一个元素。通过进队列的方法（enq）改变 tail 的值。
       */
      private transient volatile Node tail;
    
      /**
       * 同步器的状态
       */
      private volatile int state;
    
      /**
       * 返回同步器的状态
       */
      protected final int getState() {
          return state;
      }
    
      /**
       * 改变同步器的状态
       */
      protected final void setState(int newState) {
          state = newState;
      }
    
      /**
       * 比较并修改同步器的状态。。如果和给定的值相等，那么将状态改成具体的值。
       *
       */
      protected final boolean compareAndSetState(int expect, int update) {
          // See below for intrinsics setup to support this
          return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
      }
    
      /**
       * 自旋时间的阀值。
       * 在阻塞线程前，线程会先尝试获取锁，直到超过这个阀值，使用 LockSupport.park 方法阻塞线程。
       * 线程阻塞和唤醒需要一定的资源开销，如果能够在自旋期间获得锁将节省这部分的时间开销。
       * 但是自旋代表线程利用计算机资源做目标外的事情，虽然唤醒线程需要资源开销，但是一直自旋也需要浪费当前的 CPU 资源，
       * 所以自旋的时间也不建议很长。
       */
      static final long spinForTimeoutThreshold = 1000L;
    
      /**
       * 将 Node 元素加入到列表中。这是一个先进先出队列，所以加入的元素都是添加到队尾。
       */
      private Node enq(final Node node) {
          for (;;) {
              Node t = tail;
              if (t == null) { // Must initialize
                  if (compareAndSetHead(new Node()))
                      tail = head;
              } else {
                  node.prev = t;
                  if (compareAndSetTail(t, node)) {
                      t.next = node;
                      return t;
                  }
              }
          }
      }
    
      /**
       * 根据给定的策略并关联当前线程创建一个元素，并将元素入队。
       */
      private Node addWaiter(Node mode) {
          Node node = new Node(Thread.currentThread(), mode);
          // Try the fast path of enq; backup to full enq on failure
          Node pred = tail;
          if (pred != null) {
              node.prev = pred;
              if (compareAndSetTail(pred, node)) {
                  pred.next = node;
                  return node;
              }
          }
          enq(node);
          return node;
      }
    
      /**
       * 设置队列的第一个元素，并且将元素的相关线程和前一个元素置空。置空的目的一来是提醒 GC 回收，
       * 二来是废除不必要的队列相关的操作。
       *
       * @param node the node
       */
      private void setHead(Node node) {
          head = node;
          node.thread = null;
          node.prev = null;
      }
    
      /**
       * 唤醒后续元素绑定的线程。
       *
       * @param node the node
       */
      private void unparkSuccessor(Node node) {
          /*
           * 如果 waitStatus 是负数（通常表示需要告诉链表的后续元素当前元素已经唤醒了）那么将元素的 waitStatus 改成 0。
           * 如果当前操作没有修改成功，或者被后续元素绑定的线程修改了 waitStatus 的值，对同步器会不造成功能上的影响。
           */
          int ws = node.waitStatus;
          if (ws < 0)
              compareAndSetWaitStatus(node, ws, 0);
    
          /*
           * 唤醒后续元素绑定的线程。如果当前元素指定的下一个元素不存在，那么从链表的结尾往回回溯一个 waitStatus<=0 的元素，
           * 并将它绑定的线程唤醒
           */
          Node s = node.next;
          if (s == null || s.waitStatus > 0) {
              s = null;
              for (Node t = tail; t != null && t != node; t = t.prev)
                  if (t.waitStatus <= 0)
                      s = t;
          }
          if (s != null)
              LockSupport.unpark(s.thread);
      }
    
      /**
       * 共享模式下释放锁。
       */
      private void doReleaseShared() {
          /*
           * 需要确保将 head 状态置为 PROPAGATE
           */
          for (;;) {
              Node h = head;
              if (h != null && h != tail) {
                  int ws = h.waitStatus;
                  if (ws == Node.SIGNAL) {
                      /**
                       * 将状态置为 0。如果设置失败，表示其他线程已经将状态设置为其他状态了，那么直接重新循环检查。
                       */
                      if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                          continue;            // loop to recheck cases
                      /**
                       * 如果状态设置成功，唤醒后续元素
                       */
                      unparkSuccessor(h);
                  }
                  else if (ws == 0 &&
                           !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                      /**
                       * ws == 0 意味着第一个元素的状态已经被其他线程置为 0，并且后续元素已经被通知唤醒或者没有后续元素。
                       * 如果设置失败则重新循环检查。
                       */
                      continue;                // loop on failed CAS
              }
              if (h == head)                   // loop if head changed
                  break;
          }
      }
    
      /**
       * 将原来的队列第一个元素出队列，并将指定的元素设置为队列第一个元素
       *
       * @param node the node
       * @param propagate the return value from a tryAcquireShared
       */
      private void setHeadAndPropagate(Node node, int propagate) {
          Node h = head; // Record old head for check below
          setHead(node);
          /*
           * 如果 tryAcquireShared 的结果大于 0（表示共享锁还被其他的线程持有），或者队列第一个元素已经被其他线程标记为空，
           * 或者队列第一个元素的状态是传播或者通知时，需要唤醒后续元素。
           *
           */
          if (propagate > 0 || h == null || h.waitStatus < 0 ||
              (h = head) == null || h.waitStatus < 0) {
              Node s = node.next;
              if (s == null || s.isShared())
                  doReleaseShared();
          }
      }
    
      /**
       * 取消获取锁
       *
       * @param node the node
       */
      private void cancelAcquire(Node node) {
          if (node == null)
              return;
    
          node.thread = null;
    
          //过滤掉排在当前元素前面的已经被取消的元素
          Node pred = node.prev;
          while (pred.waitStatus > 0)
              node.prev = pred = pred.prev;
    
          //后续 CAS 修改需要使用到
          Node predNext = pred.next;
    
          //将当前元素的状态置为取消，后续准备从出队列。
          node.waitStatus = Node.CANCELLED;
    
          if (node == tail && compareAndSetTail(node, pred)) {
              //如果当前元素在队尾，那么将队列从等待队列中清除，并将该元素的上一个元素置为队尾。
              compareAndSetNext(pred, predNext, null);
          } else {
              // 如果后续元素需要被唤醒，将元素出队列（重新设置上一个元素和下一个元素的上下元素指针的值）
              //并设置元上一个元素的状态为 SIGNAL 或者唤醒后续元素
              int ws;
              if (pred != head &&
                  ((ws = pred.waitStatus) == Node.SIGNAL ||
                   (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                  pred.thread != null) {
                  //只有 head 或者被 cancel 的元素绑定的线程为 null
                  Node next = node.next;
                  if (next != null && next.waitStatus <= 0)
                      //如果元素还有后续的元素，那么元素出队列。
                      compareAndSetNext(pred, predNext, next);
              } else {
                  //唤醒后续的元素
                  unparkSuccessor(node);
              }
    
              node.next = node; // help GC
          }
      }
    
      /**
       * 如果节点竞争锁失败，那么检查等待队列的状态，并在符合条件的情况下修改状态的值成 SIGNAL。
       * 如果判断线程应该被阻塞，那么返回 true；否则返回 false。
       * 只有前一个元素是队列的第一个元素才能竞争到锁。但是队列第二个元素不一定能够确定竞争到锁，
       *
       * @param pred node's predecessor holding status
       * @param node the node
       * @return {@code true} if thread should block
       */
      private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
          //当前节点的上一个节点的状态
          int ws = pred.waitStatus;
          if (ws == Node.SIGNAL)
              /*
               * 如果前一个元素的状态是 SIGNAL 表明可以安全挂起，那么直接返回 true。
               */
              return true;
          if (ws > 0) {
              /*
               * 将取消状态的元素从队列中删除。
               */
              do {
                  node.prev = pred = pred.prev;
              } while (pred.waitStatus > 0);
              //node.prev 在循环中已经更新
              pred.next = node;
          } else {
              /*
               * 等待队列的状态应该是 0（初始化的状态）或者是 PROPAGATE（传播），表示在没被阻塞钱前等待唤醒通知。
               * 调用方法的线程需要保证在阻塞前不能获取锁。
               */
              compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
          }
          return false;
      }
    
      /**
       * 阻塞当前线程并返回线程中断标志。
       *
       * @return {@code true} if interrupted
       */
      private final boolean parkAndCheckInterrupt() {
          LockSupport.park(this);
          return Thread.interrupted();
      }
    
      /**
       * 独占模式下获取锁。该方法不响应线程中断，但是会把线程中断标志返回。
       *
       * @param node the node
       * @param arg the acquire argument
       * @return {@code true} if interrupted while waiting
       */
      final boolean acquireQueued(final Node node, int arg) {
          boolean failed = true;
          try {
              boolean interrupted = false;
              //
              for (;;) {
                  final Node p = node.predecessor();
                  //只有前一元素时队列第一个元素时才去获取锁。
                  //在非公平锁的情况下，尽管是队列第二个元素，但是不一定能够获取到锁。如果在获取锁的过程中，外部有一个方法调用获取锁成功，那么当前线程还是需要继续挂起。
                  if (p == head && tryAcquire(arg)) {
                      setHead(node);
                      p.next = null; // help GC
                      failed = false;
                      return interrupted;
                  }
    
                  if (shouldParkAfterFailedAcquire(p, node) &&
                      parkAndCheckInterrupt())
                      //线程被中断，将中断标志设置为 true。但是由于立刻自旋重新走 for 循环中的流程，所以线程中断不会产生影响
                      interrupted = true;
              }
          } finally {
              //如果在获取锁时报错，那么将当前元素出队
              if (failed)
                  cancelAcquire(node);
          }
      }
    
      /**
       * 可以响应中断的获取锁
       * @param arg the acquire argument
       */
      private void doAcquireInterruptibly(int arg)
          throws InterruptedException {
          final Node node = addWaiter(Node.EXCLUSIVE);
          boolean failed = true;
          try {
              for (;;) {
                  final Node p = node.predecessor();
                  if (p == head && tryAcquire(arg)) {
                      setHead(node);
                      p.next = null; // help GC
                      failed = false;
                      return;
                  }
                  if (shouldParkAfterFailedAcquire(p, node) &&
                      parkAndCheckInterrupt())
                      //LockSuport.park 可以响应中断。一旦线程中断后，线程从阻塞中恢复，那么抛出异常。
                      throw new InterruptedException();
              }
          } finally {
              if (failed)
                  cancelAcquire(node);
          }
      }
    
      /**
       * 在响应中断获取锁的基础上增加等待超时功能
       *
       * @param arg the acquire argument
       * @param nanosTimeout max wait time
       * @return {@code true} if acquired
       */
      private boolean doAcquireNanos(int arg, long nanosTimeout)
              throws InterruptedException {
          if (nanosTimeout <= 0L)
              return false;
          final long deadline = System.nanoTime() + nanosTimeout;
          final Node node = addWaiter(Node.EXCLUSIVE);
          boolean failed = true;
          try {
              for (;;) {
                  final Node p = node.predecessor();
                  if (p == head && tryAcquire(arg)) {
                      setHead(node);
                      p.next = null; // help GC
                      failed = false;
                      return true;
                  }
                  nanosTimeout = deadline - System.nanoTime();
                  if (nanosTimeout <= 0L)
                      return false;
                  //在没有超过自旋的时间阀值前一直自旋。
                  if (shouldParkAfterFailedAcquire(p, node) &&
                      nanosTimeout > spinForTimeoutThreshold)
                      LockSupport.parkNanos(this, nanosTimeout);
                  if (Thread.interrupted())
                      throw new InterruptedException();
              }
          } finally {
              if (failed)
                  cancelAcquire(node);
          }
      }
    
      /**
       * 共享模式下，不响应中断情况下获取锁
       * 独占模式下，如果有一个线程从获取到锁开始到释放锁结束，整个等待队列只有自旋尝试获取锁的为入队状态和入队了都是等待获取锁的状态
       * 共享模式下如果获得锁，会检查后续元素是否也是共享锁，如果是直接唤醒。
       * @param arg the acquire argument
       */
      private void doAcquireShared(int arg) {
          final Node node = addWaiter(Node.SHARED);
          boolean failed = true;
          try {
              boolean interrupted = false;
              for (;;) {
                  final Node p = node.predecessor();
                  if (p == head) {
                      int r = tryAcquireShared(arg);
                      if (r >= 0) {
                          setHeadAndPropagate(node, r);
                          p.next = null; // help GC
                          if (interrupted)
                              selfInterrupt();
                          failed = false;
                          return;
                      }
                  }
                  if (shouldParkAfterFailedAcquire(p, node) &&
                      parkAndCheckInterrupt())
                      interrupted = true;
              }
          } finally {
              if (failed)
                  cancelAcquire(node);
          }
      }
    
      /**
       * 独占模式下，不响应中断获取锁。
       * 先尝试获取锁（tryAcquire -  具体的子类实现），如果失败添加元素到队列中(addWaiter)，
       * 然后判断刚创建的元素能否持有锁（acquireQueued）。
       * 如果线程被中断，那么设置当前线程为中断状态（selfInterrupt）。
       *
       * @param arg the acquire argument.  This value is conveyed to
       *        {@link #tryAcquire} but is otherwise uninterpreted and
       *        can represent anything you like.
       */
      public final void acquire(int arg) {
          if (!tryAcquire(arg) &&
              acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
              selfInterrupt();
      }
    
      /**
       * 独占模式下释放锁。
       * 释放独占锁（tryRelease - 具体的子类实现）,唤醒队列中的后续元素。
       *
       * @param arg the release argument.  This value is conveyed to
       *        {@link #tryRelease} but is otherwise uninterpreted and
       *        can represent anything you like.
       * @return the value returned from {@link #tryRelease}
       */
      public final boolean release(int arg) {
          if (tryRelease(arg)) {
              Node h = head;
              if (h != null && h.waitStatus != 0)
                  unparkSuccessor(h);
              return true;
          }
          return false;
      }
    
      /**
       * 共享模式下获取锁。
       * 获取共享模式下的锁（tryAcquireShared - 具体的子类实现），如果获取失败则在不响应中断的情况下获取
       * 共享锁(doAcquireShared)。
       *
       * @param arg the acquire argument.  This value is conveyed to
       *        {@link #tryAcquireShared} but is otherwise uninterpreted
       *        and can represent anything you like.
       */
      public final void acquireShared(int arg) {
          if (tryAcquireShared(arg) < 0)
              doAcquireShared(arg);
      }
    
      /**
       * 共享模式下释放锁
       * 尝试释放锁（tryReleaseShared），如果成功通知后续元素（doReleaseShared）
       *
       * @param arg the release argument.  This value is conveyed to
       *        {@link #tryReleaseShared} but is otherwise uninterpreted
       *        and can represent anything you like.
       * @return the value returned from {@link #tryReleaseShared}
       */
      public final boolean releaseShared(int arg) {
          if (tryReleaseShared(arg)) {
              doReleaseShared();
              return true;
          }
          return false;
      }
    }


<a id="org7ebd538"></a>

# AbstractQueuedSynchronizer 子类实现的方法介绍

AbstractQueuedSynchronizer 的子类非常多，比如 ReentrantLock.Sync，ThreadPoolExecutor.Worker 等。这里只罗列实现了 AbstractQueuedSynchronizer 本身不提供实现细节要求子类实现的 ReentrantLock.FairSync 类，关于这部分方法的实现说明。


<a id="org89fa8d1"></a>

## tryAcquire

代码如下：

    /**
     * 独占策略下，以公平的方式（先到先得）获取锁
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        //c == 0表示锁可以被获取
        if (c == 0) {
            //判断是否有前驱元素，如果有则获取锁失败。hasQueuedPredecessors是保证公平锁的关键功能。
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //重入锁的特定，可以被多个线程持有。
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }


<a id="org189c148"></a>

## tryRelease

ReentrantLock.Sync 的 tryRelease实现如下：

    /**
     * 释放锁。只有当所有线程都释放锁才会返回true，否则只是减少状态的值。
     */
    protected final boolean tryRelease(int releases) {
         int c = getState() - releases;
         if (Thread.currentThread() != getExclusiveOwnerThread())
             throw new IllegalMonitorStateException();
         boolean free = false;
         if (c == 0) {
             free = true;
             setExclusiveOwnerThread(null);
         }
         setState(c);
         return free;
     }



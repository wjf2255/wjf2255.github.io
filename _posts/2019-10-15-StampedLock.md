---
title: StampedLock源码介绍
tags: [Java, JUC, 源码, 多线程, 锁]
layout: post
author: wjf
---

<div id="table-of-contents">
<h2>目录</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#org551cbdd">介绍</a></li>
<li><a href="#org47c736e">源码分析</a></li>
</ul>
</div>
</div>


<a id="org551cbdd"></a>

# 介绍

StampedLock 是一种读写锁，但是是 JUC 锁中比较特殊的一个。因为它的实现逻辑不依赖 AbstractQueuedSynchronizer，线程等待和唤醒也不依赖 LockSupport（而是直接使用 Unsafe.park 和 unpark 方法。当然 LockSupport 本身也是使用 Unsafe.park 和 unpark 方法实现的的线程阻塞和唤醒）。StampedLock 不依赖 JUC 包中的其他类，独自实现了读写锁的功能。鉴于此没有将 StampedLock 放在锁笔记中一起分析它的源码。

StampedLock 虽然是读写锁，但是比起 JUC 下的读写锁，StampedLock 提供了一种乐观读的方法。并且获取锁的过程中，大量使用自旋和 CAS 修改的方式获取锁，理论上比 ReentrantReadWriteLock 拥有更高的并发性能。

但是也是由于 StampedLock 大量使用自旋的原因（ReentrantReadWriteLock 也使用了自旋，但是没有 StampedLock 频繁），CPU 的消耗理论上也比 ReentrantReadWriteLock 高。

StampedLock 非常适合写锁中的操作非常快的业务场景。因为读锁如果因为写锁而获取锁失败，读锁会做重试获取和有限次的自旋的方式，比较晚进入到等待队列中。如果在自旋过程中，写锁能释放，那么获取读锁的线程就能避免被操作系统阻塞和唤醒等耗资源操作，增加读锁的响应效率。

StampedLock 和 ReentrantReadWriteLock 相比使用上需要格外注意。StampedLock 严重依赖版本号，需要严格按照要求使用这个版本号，并且 StampedLock 不支持重入。

StampedLock 提供三种方式使用读写锁，分别是写、读和乐观度。StampedLock 也同步器一样也有一个 state 保存锁的信息。同步器中主要保存重入次数，而 StampedLock 保存版本信息和正在被使用的上诉三种方式的信息，即线程使用了三种方式中的一种后，信息会被记录到 state 中。StampedLock 提供的 try\*方法如果成功获取锁则返回版本号，如果获取失败则返回 0。

-   写：writeLock 方法如果当下获取锁失败，线程被被阻塞（这里的阻塞包括了线程自旋和操作系统级别的线程阻塞）直到获取锁成功。writeLock 方法会返回一个版本号，用于释放锁 unlockWrite(long stamp)用。如果被上了写锁，其他线程在获取乐观读锁时都会失败（即返回的版本号为 0）。
-   读：readLock 方法如果当前获取锁失败，线程被阻塞（同样也包括自旋和操作系统级别的线程阻塞）知道获取锁成功。
-   乐观读（Optimistic Reading）：tryOptimisticRead 如果有其他线程获取了写锁，那么返回 0，否则返回版本号。这个版本号可能会在释放乐观读之前发生改变。如果获取了乐观读锁，此时还未释放，但是其他线程获取了写锁，那么之前获取到的乐观读的版本号不是最新的版本号了。但是乐观读的性能非常好，非常适合耗时非常少代码块。一般通过乐观读上锁后读取数据保存在本地变量中，然后通过校验版本号来检查这部分的数据是否还有效（如果版本号一致，说明数据未被修改）。

如果当前已经上了写锁，或者上了读锁并且没有其他线程获取过读锁，或者乐观读并且版本号是有效的，那么通过 tryConvertToWriteLock 做锁升级都是允许的。

StampedLock 没有公平策略，所有的获取都是尽最大的努力获取到锁。

官方的使用例子如下：

```java
     class Point {
      private double x, y;
      private final StampedLock sl = new StampedLock();
    
      /**
       * 写锁的例子
       */
      void move(double deltaX, double deltaY) {
        //获取写锁。如果失败线程阻塞知道获取锁。
        //stamp 是获取到锁后的版本号，释放锁时需要使用到这个值。
        //不要改动这个值内容
        long stamp = sl.writeLock();
        try {
          x += deltaX;
          y += deltaY;
        } finally {
          //释放写锁
          sl.unlockWrite(stamp);
        }
      }
    
      /**
       * 乐观读例子
       */
      double distanceFromOrigin() {
        //使用乐观读并获取版本号
        //乐观读不需要释放锁的操作
        long stamp = sl.tryOptimisticRead();
        double currentX = x, currentY = y;
        //如果版本号不可用，说明数据已经被修改了。一般建议是使用上读锁再次获取数据。
        if (!sl.validate(stamp)) {
           //上读锁并返回最新版本号信息
           stamp = sl.readLock();
           try {
             currentX = x;
             currentY = y;
           } finally {
              //释放读锁
              sl.unlockRead(stamp);
           }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
      }
    
      //锁升级例子
      void moveIfAtOrigin(double newX, double newY) {
        //上读锁并返回版本号
        long stamp = sl.readLock();
        try {
          while (x == 0.0 && y == 0.0) {
            //升级锁成写锁。如果有其他线程正在使用读锁，那么当前线程阻塞直到其他线程释放完读锁。
            long ws = sl.tryConvertToWriteLock(stamp);
            if (ws != 0L) {
              stamp = ws;
              x = newX;
              y = newY;
              break;
            }
            else {
              //释放读锁
              sl.unlockRead(stamp);
              //获取写锁
              stamp = sl.writeLock();
            }
          }
        } finally {
          //释放锁
          sl.unlock(stamp);
        }
      }
    }
```


<a id="org47c736e"></a>

# 源码分析

StampedLock 和 AbstractQueuedSynchronizer 类似，利用内置的 WNode 构建一个双向链表用于存储等待线程。与 AbstractQueuedSynchronizer 不同的是 WNode 还有一个纵向存储同时获取读锁的线程，方便前面写锁释放后同时唤醒排在后面的获取读锁阻塞的线程。

![wnode](/assets/image/wnode.png)

```java
    public class StampedLock implements java.io.Serializable {
    
      /**
       * Java 虚拟机的可用的处理器数量，用于计算自旋次数
       * 这个值服务器的 CPU 核数量并不完全一致，它 和 CPU 核数，超线程数量相关，但是又不是简单的乘积。
       */
      private static final int NCPU = Runtime.getRuntime().availableProcessors();
    
      /**
       * 获取锁失败后准备进入等待队列前的自旋次数
       */
      private static final int SPINS = (NCPU > 1) ? 1 << 6 : 0;
    
      /**
       * 如果当前等待队列的第二个节点（快到当前获取到锁了），那么自旋等待。这个参数是这个场景下的自旋次数
       */
      private static final int HEAD_SPINS = (NCPU > 1) ? 1 << 10 : 0;
    
      /**
       * 第二次阻塞最自旋次数
       */
      private static final int MAX_HEAD_SPINS = (NCPU > 1) ? 1 << 16 : 0;
    
      /**
       * 线程是否让出 CPU 的比较参数
       */
      private static final int OVERFLOW_YIELD_RATE = 7;
    
      /**
       * state 用于存储读写锁标记的分割位 1 << 7  存储写锁标记，低 7 位存储读锁次数
       */
      private static final int LG_READERS = 7;
    
      /**
       * 0000 0001，读锁变更单位，即获取读锁时，低 6 位加 1。
       */
      private static final long RUNIT = 1L;
      /**
       * 1000 0000，写锁标记。当 state & WBIT = WBIT 时，有线程持有写锁。1000 0000。
       */
      private static final long WBIT  = 1L << LG_READERS;
      /**
       * 0111 1111，读状态标识全为 1 时的值，方便做位运算，判断是否读锁溢出
       */
      private static final long RBITS = WBIT - 1L;
      /**
       * 0111 1110 读锁上限
       */
      private static final long RFULL = RBITS - 1L;
      /**
       * 1111 1111 读写锁全部为 1，方便做位运算，判断未上锁(s & ABITS == 0)，读锁溢出等（(s & ABITS) >= RFULL）
       */
      private static final long ABITS = RBITS | WBIT;
      /**
       * 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1000 0000 RBITS 取反的结果，方便获取写锁以及更高位数的值
       */
      private static final long SBITS = ~RBITS;
    
      // 1 0000 0000 state 初始化的值，和获取锁失败返回的 0 值做区分。
      private static final long ORIGIN = WBIT << 1;
    
      // 线程中断的标识。
      private static final long INTERRUPTED = 1L;
    
      // 节点的状态
      private static final int WAITING   = -1;
      private static final int CANCELLED =  1;
    
      // 节点的类型
      private static final int RMODE = 0;
      private static final int WMODE = 1;
    
      /** 节点的类结构 */
      static final class WNode {
          volatile WNode prev;
          volatile WNode next;
          volatile WNode cowait;    // 读节点的队列
          volatile Thread thread;   // 线程
          volatile int status;      // 节点状态。0 表示是初始状态，-1 是等待状态，1 是取消状态
          final int mode;           // RMODE or WMODE
          WNode(int m, WNode p) { mode = m; prev = p; }
      }
    
      /**
       * 指向队列的第一个节点
       */
      private transient volatile WNode whead;
      //等待队列的最后一个节点
      private transient volatile WNode wtail;
    
      // 读写锁视图。
      transient ReadLockView readLockView;
      transient WriteLockView writeLockView;
      transient ReadWriteLockView readWriteLockView;
    
      /**
       * 状态。保存写锁和读锁信息
       */
      private transient volatile long state;
      // 如果读锁达到 state 中能够存储的上限后，溢出的读锁数量保存在 readerOverflow 中
      private transient int readerOverflow;
    
      /**
       * 构造函数。将 state 值初始化成 ORIGIN 1 0000 0000。区分获取锁失败后返回的 0 值。
       */
      public StampedLock() {
          state = ORIGIN;
      }
    
      /**
       * 上写锁。如果上锁失败会阻塞
       *
       * @return 版本号。释放锁和改变模式（写锁变读锁）时用到
       */
      public long writeLock() {
          long s, next;
          /**
           * ((s = state) & ABITS) == 0 未上锁的状态条件才存在
           * 如果未上过锁，那么通过 CAS 更新写锁标志位。如果成功返回新的版本号。初始情况下为 1 1000 0000
           * CAS 更新失败（多个线程通过获取锁）则调用 acquireWrite 方法获取锁并返回版本号。
           */
          return ((((s = state) & ABITS) == 0L &&
                   U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ?
                  next : acquireWrite(false, 0L));
      }
    
      /**
       * 尝试获取一个写锁。如果获取成功则返回一个版本号（大于 0 的长整形数值），失败则返回 0
       * 获取没有成功获取锁不会阻塞线程
       * @return 0 或者一个版本号
       */
      public long tryWriteLock() {
          long s, next;
          return ((((s = state) & ABITS) == 0L &&
                   U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ?
                  next : 0L);
      }
    
      /**
       * 尝试获取锁。如果在指定的时间内并且线程未被中断获取锁失败，线程会被阻塞（包括自旋）。
       *
       * @param 如果获取锁失败则阻塞，阻塞的最长时间。超过这个时间线程被唤醒并且获取锁失败（返回 0）
       * @param 等待时间的单位。
       * @return 0 或者版本号
       * @throws 如果线程阻塞抛出 InterruptedException
       */
      public long tryWriteLock(long time, TimeUnit unit)
          throws InterruptedException {
          long nanos = unit.toNanos(time);
          if (!Thread.interrupted()) {
              long next, deadline;
              if ((next = tryWriteLock()) != 0L)
                  return next;
              if (nanos <= 0L)
                  return 0L;
              if ((deadline = System.nanoTime() + nanos) == 0L)
                  deadline = 1L;
              if ((next = acquireWrite(true, deadline)) != INTERRUPTED)
                  return next;
          }
          throw new InterruptedException();
      }
    
      /**
       * 尝试获取锁。如果获取失败线程阻塞（包括自旋）。如果线程中断则抛出 InterruptedException
       *
       * @return a stamp that can be used to unlock or convert mode
       * @throws InterruptedException if the current thread is interrupted
       * before acquiring the lock
       */
      public long writeLockInterruptibly() throws InterruptedException {
          long next;
          if (!Thread.interrupted() &&
              (next = acquireWrite(true, 0L)) != INTERRUPTED)
              return next;
          throw new InterruptedException();
      }
    
      /**
       * 获取读锁。如果获取成功则返回版本号，如果获取失败则线程阻塞。
       *
       * @return 版本号
       */
      public long readLock() {
          long s = state, next;
          /**
           * whead == wtatil 表示队列中没有等待线程
           * (s & ABITS) < RFULL 没有上写锁并且锁未溢出
           * CAS 修改成功
           * 如果以上条件都成立，返回版本号 next = s + RUNIT，否则通过调用 acquireRead 获取锁
           */
          return ((whead == wtail && (s & ABITS) < RFULL &&
                   U.compareAndSwapLong(this, STATE, s, next = s + RUNIT)) ?
                  next : acquireRead(false, 0L));
      }
    
      /**
       * 获取读锁。如果当前获取到锁，则立即返回版本号，如果无法获取到锁则自旋或者线程阻塞。
       *
       * @return a stamp that can be used to unlock or convert mode,
       * or zero if the lock is not available
       */
      public long tryReadLock() {
          //自旋获取锁
          for (;;) {
              long s, m, next;
              // 如果已经上写锁，那么获取失败，返回 0
              if ((m = (s = state) & ABITS) == WBIT)
                  return 0L;
              //如果未超过读锁上限，那么 CAS 更新上读锁
              else if (m < RFULL) {
                  if (U.compareAndSwapLong(this, STATE, s, next = s + RUNIT))
                      return next;
              }
              // 超过读锁上限，调用 tryIncReaderOverflow 方法处理上限部分的读锁次数存储
              else if ((next = tryIncReaderOverflow(s)) != 0L)
                  return next;
          }
      }
    
      /**
       * 获取读锁。支持超时时间和响应线程中断。
       *
       * @param time the maximum time to wait for the lock
       * @param unit the time unit of the {@code time} argument
       * @return a stamp that can be used to unlock or convert mode,
       * or zero if the lock is not available
       * @throws InterruptedException if the current thread is interrupted
       * before acquiring the lock
       */
      public long tryReadLock(long time, TimeUnit unit)
          throws InterruptedException {
          long s, m, next, deadline;
          long nanos = unit.toNanos(time);
          if (!Thread.interrupted()) {
              if ((m = (s = state) & ABITS) != WBIT) {
                  if (m < RFULL) {
                      if (U.compareAndSwapLong(this, STATE, s, next = s + RUNIT))
                          return next;
                  }
                  else if ((next = tryIncReaderOverflow(s)) != 0L)
                      return next;
              }
              if (nanos <= 0L)
                  return 0L;
              if ((deadline = System.nanoTime() + nanos) == 0L)
                  deadline = 1L;
              if ((next = acquireRead(true, deadline)) != INTERRUPTED)
                  return next;
          }
          throw new InterruptedException();
      }
    
      /**
       * 获取读锁。该方法响应线程中断。
       *
       * @return a stamp that can be used to unlock or convert mode
       * @throws InterruptedException if the current thread is interrupted
       * before acquiring the lock
       */
      public long readLockInterruptibly() throws InterruptedException {
          long next;
          if (!Thread.interrupted() &&
              (next = acquireRead(true, 0L)) != INTERRUPTED)
              return next;
          throw new InterruptedException();
      }
    
      /**
       * 乐观读。
       * 通过判断是否上写锁快速返回一个版本号或者 0。如果上写锁了返回 0，否则返回一个版本号（读锁位数的取反值）。
       * 由于乐观读返回的版本号是固定的值，所以任何乐观读返回的版本号都是一样的
       *
       * @return a stamp, or zero if exclusively locked
       */
      public long tryOptimisticRead() {
          long s;
          return (((s = state) & WBIT) == 0L) ? (s & SBITS) : 0L;
      }
    
      /**
       * 校验版本号
       *
       * @param stamp a stamp
       * @return {@code true} if the lock has not been exclusively acquired
       * since issuance of the given stamp; else false
       */
      public boolean validate(long stamp) {
          // 加入内存屏障，确保 state 是最新的值
          U.loadFence();
          // SBITS 读位数取反，(stamp & SBITS) == (state & SBITS) 表示忽略读锁看其他位数是否发生变化，如果发生变化则返回 false，未发生变化则返回 true
          return (stamp & SBITS) == (state & SBITS);
      }
    
      /**
       * 如果版本号不正确则抛出 IllegalMonitorStateException 异常，正确则释放写锁
       *
       *
       * @param stamp a stamp returned by a write-lock operation
       * @throws IllegalMonitorStateException if the stamp does
       * not match the current state of this lock
       */
      public void unlockWrite(long stamp) {
          WNode h;
          /**
           * state != stamp 版本号发生变化
           * (stamp & WBIT) == 0L 没有上写锁
           * 由于是释放写锁，在释放前版本号不应该发生变化且应该是上过写锁的。不满足以上要求则抛出异常
           */
          if (state != stamp || (stamp & WBIT) == 0L)
              throw new IllegalMonitorStateException();
          // 写位数+1 会向上进 1，写位数变成 0，释放锁。如果向上进 1 后 state == 0 则表示溢出，那么 state 设为初始默认值 ORIGIN。否则为写位数加一后的结果
          state = (stamp += WBIT) == 0L ? ORIGIN : stamp;
          if ((h = whead) != null && h.status != 0)
              release(h);
      }
    
      /**
       * 如果版本号正确，则释放读锁并唤醒后续节点。
       *
       * @param stamp a stamp returned by a read-lock operation
       * @throws IllegalMonitorStateException if the stamp does
       * not match the current state of this lock
       */
      public void unlockRead(long stamp) {
          long s, m; WNode h;
          for (;;) {
               /**
                * (s = state) & SBITS) != (stamp & SBITS) 除读锁位数外，其他位数是否发生变化
                *  (stamp & ABITS) == 0L 之前获取的版本号没有上过读锁也没有上过写锁
                *  (m = s & ABITS) == 0L || m == WBIT 当前版本号没有读锁或者上了写锁
                * 以上情况又发生则抛出 IllegalMonitorStateException
                */
              if (((s = state) & SBITS) != (stamp & SBITS) ||
                  (stamp & ABITS) == 0L || (m = s & ABITS) == 0L || m == WBIT)
                  throw new IllegalMonitorStateException();
              // 读锁未溢出
              if (m < RFULL) {
                  // CAS 修改版本号
                  if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {
                      /**
                       * m == RUNIT 表示当前只有个读锁，并且需要将这个唯一的读锁给释放掉
                       * (h = whead) != nul 表示等待队列中节点（鉴于当前运行是读锁的线程，所以等待队列中 head 应该是写锁）
                       *  h.status != 0 说明线程被阻塞而不是在自旋
                       * 以上情况都满足，需要协助唤醒队列中的节点
                       */
                      if (m == RUNIT && (h = whead) != null && h.status != 0)
                          release(h);
                      break;
                  }
              }
              // 读锁未溢出，通过 tryDecReaderOverflow 维护版本号。如果 tryDecReaderOverflow 返回不为 0 说明释放锁成功了，跳出自旋
              else if (tryDecReaderOverflow(s) != 0L)
                  break;
          }
      }
    
      /**
       * 如果版本号正确，那么释放对应的锁。
       *
       * @param stamp a stamp returned by a lock operation
       * @throws IllegalMonitorStateException if the stamp does
       * not match the current state of this lock
       */
      public void unlock(long stamp) {
          long a = stamp & ABITS, m, s; WNode h;
          // ((s = state) & SBITS) == (stamp & SBITS) 表示写标志未发生变化
          while (((s = state) & SBITS) == (stamp & SBITS)) {
              // ((m = s & ABITS) == 0L) 表示未上任何锁。跳出循环抛出异常
              if ((m = s & ABITS) == 0L)
                  break;
              // (m == WBIT 表示只有写锁。如果只有写锁，但是版本信息和只有写锁不相符，跳出循环抛出异常
              else if (m == WBIT) {
                  if (a != m)
                      break
                  // 版本信息和只有写锁相符合，版本号加上一个写锁位数。如果 state 溢出（state == 0L），那么等于初始化值，未溢出则为加后 的结果
                  state = (s += WBIT) == 0L ? ORIGIN : s;
                  // 如果等待队列不为空并且头结点的状态不是初始化的状态，那么唤醒等待队列的后续节点，之后释放锁操作结束。
                  if ((h = whead) != null && h.status != 0)
                      release(h);
                  return;
              }
              /**
               * a == 0L 表示未上读写锁
               * a >= WBIT 即上了写锁又上了读锁
               * 满足以上任一条件则跳出循环抛出异常
               */
              else if (a == 0L || a >= WBIT)
                  break;
              // (m < RFULL 表示只上了读锁并且读锁数量未溢出
              else if (m < RFULL) {
                  // 利用 CAS 方式读锁减一
                  if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {
                      // 如果读锁只有一个并且等待队列不为空并且头结点的状态不是初始化的状态，那么唤醒等待队列后续节点。
                      if (m == RUNIT && (h = whead) != null && h.status != 0)
                          release(h);
                      return;
                  }
              }
              // 溢出部分使用 tryDecReaderOverflow 方法处理释放读锁。如果返回值不为 0，则释放成功完成释放锁操作；否则自旋
              else if (tryDecReaderOverflow(s) != 0L)
                  return;
          }
          throw new IllegalMonitorStateException();
      }
    
      /**
       * 升级为写锁
       * 首先验证版本号是否有效，无效则返回 0。
       * 如果只有一个写锁（当前线程已经获取了写锁），立即返回当前版本号（版本号未发生变化）
       * 如果只有一个读锁，去掉读锁换上写锁（state -1 + WBIT）,返回新版本号
       * 如果是乐观读，加上读锁（state + WBIT），返回新版本号
       * 其他情况返回 0，升级失败
       *
       * @param stamp a stamp
       * @return a valid write stamp, or zero on failure
       */
      public long tryConvertToWriteLock(long stamp) {
          long a = stamp & ABITS, m, s, next;
          /**
           * ((s = state) & SBITS) == (stamp & SBITS) 表示获取版本号到调用 tryConvertToWriteLock 期间读锁未发生变化
           */
          while (((s = state) & SBITS) == (stamp & SBITS)) {
              // (m = s & ABITS) == 0L 表示没有任何读写锁
              if ((m = s & ABITS) == 0L) {
                  // a != 0L 表示之前获取的版本号表明有读写锁，当时 m==0L 说明，说明之前获取的版本号已经过期，升级锁失败
                  if (a != 0L)
                      break;
                  // CAS 操作加写锁，并返回新的版本号。能做此操作说明没有读写锁，不需要考虑溢出的问题
                  if (U.compareAndSwapLong(this, STATE, s, next = s + WBIT))
                      return next;
              }
              // 说明有写锁
              else if (m == WBIT) {
                  // a != m 说明在操作期间，版本已经发生变化，stamp 不是最新的版本号，跳出循环升级失败
                  if (a != m)
                      break;
                  return stamp;
              }
              // 当前只持有一个读锁并且获取到的版本号有读锁，说明获取版本号时可能多个也可能一个线程获取了读锁，但是现在还剩一个线程持有读锁（当前线程），那么将读锁更新成写锁，并返回版本号
              else if (m == RUNIT && a != 0L) {
                  if (U.compareAndSwapLong(this, STATE, s,
                                           next = s - RUNIT + WBIT))
                      return next;
              }
              else
                  break;
          }
          return 0L;
      }
    
      /**
       * 转成读锁
       * 版本号正确并且没有任何锁，那么加一个读锁。
       * 版本号正确并且读锁溢出，那么使用 tryIncReaderOverflow 处理溢出部分
       * 版本号正确并且只有一个写锁，那么释放写锁并且读锁加 1 ,并且唤醒后续线程。
       * 版本号正确并且是乐观读（没有任何锁），那么加上一个读锁
       * 其他情况转换读锁失败
       *
       * @param stamp a stamp
       * @return a valid read stamp, or zero on failure
       */
      public long tryConvertToReadLock(long stamp) {
          //  a = stamp & ABITS a b 表示版本号中读写锁的情况
          long a = stamp & ABITS, m, s, next; WNode h;
          // ((s = state) & SBITS) == (stamp & SBITS) 说明在做换读锁操作时，读写锁位数未发生变化，版本号有效
          while (((s = state) & SBITS) == (stamp & SBITS)) {
              // m = s & ABITS) == 0L m 表示是当前读写锁的情况，m == 0L 表示当前没有任何锁。
              if ((m = s & ABITS) == 0L) {
                  // 锁已经发生变化，锁升级失败
                  if (a != 0L)
                      break;
                  // 当前读锁未溢出
                  else if (m < RFULL) {
                      // CAS 修改锁，成功则返回最新的版本号，否则自旋
                      if (U.compareAndSwapLong(this, STATE, s, next = s + RUNIT))
                          return next;
                  }
                  // 溢出则使用 tryIncReaderOverflow 处理溢出部分的读锁。成功则返回最新的版本号
                  else if ((next = tryIncReaderOverflow(s)) != 0L)
                      return next;
              }
              // m == WBIT 表示当下的版本号只有写锁
              else if (m == WBIT) {
                  // a != m 表示状态在这期间发生变化，跳出循环，转变失败。上了写锁后，释放前不允许上读锁。a !=m 表明版本已经过期
                  if (a != m)
                      break;
                  // 更新版本号。s + (WBIT + RUNIT) 写锁位数加一后向上进一，写锁标志消失，并且读锁加一
                  state = next = s + (WBIT + RUNIT);
                  // 如果等待队列头节点不为空并且状态不是初始化的状态，说明等待队列有后续的阻塞节点。由于是写锁
                  if ((h = whead) != null && h.status != 0)
                      release(h);
                  return next;
              }
              // 如果已经上锁（a != 0），并且是读锁，那么立刻返回 stamp。当前是读锁，不需要做处理
              else if (a != 0L && a < WBIT)
                  return stamp;
              else
                  break;
          }
          return 0L;
      }
    
      /**
       * 转成乐观读锁。
       * 版本号正确并且当前是乐观读，返回最新的版本号
       * 如果是当前线程获取的写锁，释放写锁，如果有等待队列有后续节点，那么唤醒。最后返回最新的版本号
       * 如果是当前线程获取的读锁，那么释放读锁并返回最新的版本号。
       *
       * @param stamp a stamp
       * @return a valid optimistic read stamp, or zero on failure
       */
      public long tryConvertToOptimisticRead(long stamp) {
          long a = stamp & ABITS, m, s, next; WNode h;
          U.loadFence();
          for (;;) {
              if (((s = state) & SBITS) != (stamp & SBITS))
                  break;
              if ((m = s & ABITS) == 0L) {
                  if (a != 0L)
                      break;
                  return s;
              }
              else if (m == WBIT) {
                  if (a != m)
                      break;
                  state = next = (s += WBIT) == 0L ? ORIGIN : s;
                  if ((h = whead) != null && h.status != 0)
                      release(h);
                  return next;
              }
              else if (a == 0L || a >= WBIT)
                  break;
              else if (m < RFULL) {
                  if (U.compareAndSwapLong(this, STATE, s, next = s - RUNIT)) {
                      if (m == RUNIT && (h = whead) != null && h.status != 0)
                          release(h);
                      return next & SBITS;
                  }
              }
              else if ((next = tryDecReaderOverflow(s)) != 0L)
                  return next & SBITS;
          }
          return 0L;
      }
    
      /**
       * 释放写锁
       * 如果有写锁，那么释放写锁。查看是否有需要唤醒后续节点。如果释放写锁成功返回 true，否则返回 false
       *
       * @return {@code true} if the lock was held, else false
       */
      public boolean tryUnlockWrite() {
          long s; WNode h;
          if (((s = state) & WBIT) != 0L) {
              state = (s += WBIT) == 0L ? ORIGIN : s;
              if ((h = whead) != null && h.status != 0)
                  release(h);
              return true;
          }
          return false;
      }
    
      /**
       * 释放读锁
       * 如果有读锁，那么释放读锁，查看是否有需要唤醒后续节点。如果释放读锁成功返回 true，否则返回 false
       *
       * @return {@code true} if the read lock was held, else false
       */
      public boolean tryUnlockRead() {
          long s, m; WNode h;
          while ((m = (s = state) & ABITS) != 0L && m < WBIT) {
              if (m < RFULL) {
                  if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {
                      if (m == RUNIT && (h = whead) != null && h.status != 0)
                          release(h);
                      return true;
                  }
              }
              else if (tryDecReaderOverflow(s) != 0L)
                  return true;
          }
          return false;
      }
    
      // status monitoring methods
    
      /**
       * 返回当前读锁的数量
       */
      private int getReadLockCount(long s) {
          long readers;
          if ((readers = s & RBITS) >= RFULL)
              readers = RFULL + readerOverflow;
          return (int) readers;
      }
    
      /**
       * 返回当前是否有线程持有写锁
       *
       * @return {@code true} if the lock is currently held exclusively
       */
      public boolean isWriteLocked() {
          return (state & WBIT) != 0L;
      }
    
      /**
       * 返回当前是否有读锁
       *
       * @return {@code true} if the lock is currently held non-exclusively
       */
      public boolean isReadLocked() {
          return (state & RBITS) != 0L;
      }
    
      /**
       * 返回当前读锁的数量
       * @return the number of read locks held
       */
      public int getReadLockCount() {
          return getReadLockCount(state);
      }
    
      /**
       * Returns a string identifying this lock, as well as its lock
       * state.  The state, in brackets, includes the String {@code
       * "Unlocked"} or the String {@code "Write-locked"} or the String
       * {@code "Read-locks:"} followed by the current number of
       * read-locks held.
       *
       * @return a string identifying this lock, as well as its lock state
       */
      public String toString() {
          long s = state;
          return super.toString() +
              ((s & ABITS) == 0L ? "[Unlocked]" :
               (s & WBIT) != 0L ? "[Write-locked]" :
               "[Read-locks:" + getReadLockCount(s) + "]");
      }
    
      // views
    
      /**
       * 返回一个读锁的视图。
       * 利用 StampedLock 的读锁来实现 Lock 接口定义的功能。由于本身并不会处理线程中断，所以无法实现 lockInterruptibly。
       *
       * @return the lock
       */
      public Lock asReadLock() {
          ReadLockView v;
          return ((v = readLockView) != null ? v :
                  (readLockView = new ReadLockView()));
      }
    
      /**
       * 返回一个写锁的试图。
       * 利用 StampedLock 的写锁来实现 Lock 接口定义的功能。由于本身并不会处理线程中断，所以无法实现 lockInterruptibly。
       *
       * @return the lock
       */
      public Lock asWriteLock() {
          WriteLockView v;
          return ((v = writeLockView) != null ? v :
                  (writeLockView = new WriteLockView()));
      }
    
      /**
       * 返回一个读写锁的试图
       * 利用读锁视图和写锁视图实现读写锁
       *
       * @return the lock
       */
      public ReadWriteLock asReadWriteLock() {
          ReadWriteLockView v;
          return ((v = readWriteLockView) != null ? v :
                  (readWriteLockView = new ReadWriteLockView()));
      }
    
    
      final class ReadLockView implements Lock {
          public void lock() { readLock(); }
          public void lockInterruptibly() throws InterruptedException {
              readLockInterruptibly();
          }
          public boolean tryLock() { return tryReadLock() != 0L; }
          public boolean tryLock(long time, TimeUnit unit)
              throws InterruptedException {
              return tryReadLock(time, unit) != 0L;
          }
          public void unlock() { unstampedUnlockRead(); }
          public Condition newCondition() {
              throw new UnsupportedOperationException();
          }
      }
    
      final class WriteLockView implements Lock {
          public void lock() { writeLock(); }
          public void lockInterruptibly() throws InterruptedException {
              writeLockInterruptibly();
          }
          public boolean tryLock() { return tryWriteLock() != 0L; }
          public boolean tryLock(long time, TimeUnit unit)
              throws InterruptedException {
              return tryWriteLock(time, unit) != 0L;
          }
          public void unlock() { unstampedUnlockWrite(); }
          public Condition newCondition() {
              throw new UnsupportedOperationException();
          }
      }
    
      final class ReadWriteLockView implements ReadWriteLock {
          public Lock readLock() { return asReadLock(); }
          public Lock writeLock() { return asWriteLock(); }
      }
    
      // Unlock methods without stamp argument checks for view classes.
      // Needed because view-class lock methods throw away stamps.
    
      final void unstampedUnlockWrite() {
          WNode h; long s;
          if (((s = state) & WBIT) == 0L)
              throw new IllegalMonitorStateException();
          state = (s += WBIT) == 0L ? ORIGIN : s;
          if ((h = whead) != null && h.status != 0)
              release(h);
      }
    
      final void unstampedUnlockRead() {
          for (;;) {
              long s, m; WNode h;
              if ((m = (s = state) & ABITS) == 0L || m >= WBIT)
                  throw new IllegalMonitorStateException();
              else if (m < RFULL) {
                  if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {
                      if (m == RUNIT && (h = whead) != null && h.status != 0)
                          release(h);
                      break;
                  }
              }
              else if (tryDecReaderOverflow(s) != 0L)
                  break;
          }
      }
    
      private void readObject(java.io.ObjectInputStream s)
          throws java.io.IOException, ClassNotFoundException {
          s.defaultReadObject();
          state = ORIGIN; // reset to unlocked state
      }
    
      // internals
    
      /**
       * 处理超过读锁上限后溢出部分的存储
       *
       * @param s a reader overflow stamp: (s & ABITS) >= RFULL
       * @return new stamp on success, else zero
       */
      private long tryIncReaderOverflow(long s) {
          /**
           * (s & ABITS) == RFULL 表明未上写锁且读锁达到上限
           */
          if ((s & ABITS) == RFULL) {
              // 更新成成功则利用 readerOverflow 存储溢出部分的读锁次数
              // 溢出部分的版本号都是 RFULL
              if (U.compareAndSwapLong(this, STATE, s, s | RBITS)) {
                  ++readerOverflow;
                  state = s;
                  return s;
              }
          }
          // 很可能是其他线程获取了写锁。此时将 CPU 的使用权让出去
          else if ((LockSupport.nextSecondarySeed() &
                    OVERFLOW_YIELD_RATE) == 0)
              Thread.yield();
          return 0L;
      }
    
      /**
       * 超过读锁上限时，减少读锁数量需要通过调用 tryDecReaderOverflow 来减少读锁数量
       *
       * @param 版本号
       * @return 新版本号或者释放失败返回 0
       */
      private long tryDecReaderOverflow(long s) {
          //(s & ABITS) == RFULL 表示读锁超过上限
          if ((s & ABITS) == RFULL) {
              // s | RBITS 确保是锁溢出的状态
              if (U.compareAndSwapLong(this, STATE, s, s | RBITS)) {
                  int r; long next;
                  // 如果 readerOverflow > 0 则从 readerOverflow 中扣除锁的次数
                  if ((r = readerOverflow) > 0) {
                      readerOverflow = r - 1;
                      next = s;
                  }
                  // 如果 readerOverflow == 0 则从 state 中扣除锁的次数
                  else
                      next = s - RUNIT;
                   state = next;
                   // 返回新版本号
                   return next;
              }
          }
          // 到这里可能是上了写锁，那么让渡出 CPU 的权限
          else if ((LockSupport.nextSecondarySeed() &
                    OVERFLOW_YIELD_RATE) == 0)
              Thread.yield();
          return 0L;
      }
    
      /**
       * 唤醒指定节点的后续非取消节点。指定的节点通常是等待队列第一个节点，而被唤醒的节点通常是 h.next。
       * 但是如果 h.next 的指向被其他线程修改导致 h.next 发生变化（通常是取消节点引起的），
       * 那么需要从队尾往前回溯非取消状态的节点。release 方法不一定能够成功唤醒后续节点，
       * 但是取消节点操作会有额外的释放后续节点的操作确保节点能够成功被唤醒
       */
      private void release(WNode h) {
          if (h != null) {
              WNode q; Thread w;
              // 将指定节点的状态更新成初始状态
              U.compareAndSwapInt(h, WSTATUS, WAITING, 0);
              // 如果 h.next 是空或者是取消状态，那么从队尾回溯一个非取消状态的节点
              if ((q = h.next) == null || q.status == CANCELLED) {
                  for (WNode t = wtail; t != null && t != h; t = t.prev)
                      if (t.status <= 0)
                          q = t;
              }
              // 如果 q 非空且绑定了线程，唤醒线程
              if (q != null && (w = q.thread) != null)
                  U.unpark(w);
          }
      }
    
      /**
       * 尝试获取锁并返回最新版本号
       *
       * @param interruptible 是否需要响应中断
       * @param deadline if nonzero, the System.nanoTime value to timeout
       * at (and return zero)
       * @return 最新版本号或者中断标志
       */
      private long acquireWrite(boolean interruptible, long deadline) {
          WNode node = null, p;
    
          /**
           * 检查队列是否为空，如果为空则初始化队列
           * 由于是 if else，所以每次 for 循环都只处理一小段代码。越往后需要循环越多的次数才能到达
           */
          for (int spins = -1;;) { // spin while enqueuing
              long m, s, ns;
              //如果是初始化状态并且 CAS 更新锁成功，那么返回最新的版本号。初始状态是 1 1000 0000
              if ((m = (s = state) & ABITS) == 0L) {
                  if (U.compareAndSwapLong(this, STATE, s, ns = s + WBIT))
                      return ns;
              }
              // 如果是第一次进来（spins 一开始就赋值为-1），那么计算自旋次数。如果当前状态只有写锁上有值，
              // 那么自选次数为入队前的最大自旋次数，否则自选次数为 0
              else if (spins < 0)
                  spins = (m == WBIT && wtail == whead) ? SPINS : 0;
                  // 如果计算得到的自旋次数大于 0，那么使用随机判断减少自旋
              else if (spins > 0) {
                  if (LockSupport.nextSecondarySeed() >= 0)
                      --spins;
              }
              /**
               * 如果等待队列是空队列队列，初始化等待队列。在队列头部添加写节点
               */
              else if ((p = wtail) == null) { // initialize queue
                  WNode hd = new WNode(WMODE, null);
                  if (U.compareAndSwapObject(this, WHEAD, null, hd))
                      wtail = hd;
              }
              //初始化 node 节点
              else if (node == null)
                  node = new WNode(WMODE, p);
              //  将 node 节点的上一个节点更新成队尾节点
              else if (node.prev != p)
                  node.prev = p;
                  //将 node 节点添加到队列尾部。
              else if (U.compareAndSwapObject(this, WTAIL, p, node)) {
                  p.next = node;
                  break;
              }
          }
    
          /**
           * 循环等待获取锁或者阻塞
           */
          for (int spins = -1;;) {
              // h 是第一个节点，np 是队尾节点 pp 是倒数第二个节点，ps 表示是队尾节点的状态
              WNode h, np, pp; int ps;
              //如果是队列第一个节点，那么自旋等待所释放。
              if ((h = whead) == p) {
                  // 设置自旋次数
                  if (spins < 0)
                      spins = HEAD_SPINS;
                  //如果上一次自旋（第一次是 HEAD_SPINS 次）还未获取到锁，那么只要未超过 MAX_HEAD_SPINS 次，在当前自旋数量上增加 2 倍数量
                  else if (spins < MAX_HEAD_SPINS)
                      spins <<= 1;
                  // 自旋修改 state，做上锁操作
                  for (int k = spins;;) { // spin at head
                      long s, ns;
                      //如果未上锁才做上写锁的操作
                      if (((s = state) & ABITS) == 0L) {
                          if (U.compareAndSwapLong(this, STATE, s,
                                                   ns = s + WBIT)) {
                              whead = node;
                              node.prev = null;
                              return ns;
                          }
                      }
                      // 递减自旋次数
                      else if (LockSupport.nextSecondarySeed() >= 0 &&
                               --k <= 0)
                          break;
                  }
              }
              // h != null 表示队列中已经有节点
              else if (h != null) { // help release stale waiters
                  WNode c; Thread w;
                  // cowait 不为 null 表示是第一个节点是读锁，并且有多个线程获取了读锁。遍历 cowait 队列并唤醒所有的线程
                  while ((c = h.cowait) != null) {
                      if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                          (w = c.thread) != null)
                          U.unpark(w);
                  }
              }
              //头结点未发生改变
              if (whead == h) {
                  //如果队尾节点发生变化，维护队尾和新增节点的双向关系
                  if ((np = node.prev) != p) {
                      if (np != null)
                          (p = np).next = node;   // stale
                  }
                  // CAS 将队尾的状态变更成-1。如此竞争队尾时，无法同时竞争成功。
                  else if ((ps = p.status) == 0)
                      U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
                  // 如果队尾节点是取消的状态，那么将节点提出队列
                  else if (ps == CANCELLED) {
                      if ((pp = p.prev) != null) {
                          node.prev = pp;
                          pp.next = node;
                      }
                  }
                  // 阻塞线程
                  else {
                      long time; // 0 argument to park means no timeout
                      if (deadline == 0L)
                          time = 0L;
                          //如果等待超时则取消节点并返回获取锁失败
                      else if ((time = deadline - System.nanoTime()) <= 0L)
                          //取消等待节点。等待获取锁的线程获取锁失败，并且线程会被唤醒。
                          return cancelWaiter(node, node, false);
                      Thread wt = Thread.currentThread();
                      U.putObject(wt, PARKBLOCKER, this);
                      node.thread = wt;
                      /**
                       * 1. 队尾节点处于等待转改
                       * 2. 队尾节点不是队列第一个节点结点 或者 队列已经上锁
                       * 3. 队列第一个节点和队尾节点在自旋期间未发生变化
                       * 同时满足以上三个条件，当前线程被挂起
                       */
                      if (p.status < 0 && (p != h || (state & ABITS) != 0L) &&
                          whead == h && node.prev == p)
                          U.park(false, time);  // emulate LockSupport.park
                      // 线程被唤醒后将线程的 parkBlocker 属性置空
                      node.thread = null;
                      U.putObject(wt, PARKBLOCKER, null);
                      // 如果是线程中断导致的唤醒，那么取消节点。释放锁方式唤醒线程会将节点剔出队列。
                      if (interruptible && Thread.interrupted())
                          return cancelWaiter(node, node, true);
                  }
              }
          }
      }
    
      /**
       * 获取读锁
       * 1. 如果队列中就一个节点，那么通过自旋+CAS 修改的方式获取锁
       * 1. CAS 是否修改成功，如果修改成功则返回版本号
       * 2. 如果有必要则初始化队列
       * 3.
       *
       * @param interruptible 是否响应中断 true 表示响应中断
       * @param deadline 超时时间
       * @return 版本号或者中断标志（-1 表示线程被中断）
       */
      private long acquireRead(boolean interruptible, long deadline) {
          WNode node = null, p;
          for (int spins = -1;;) {
              WNode h;
              // 如果队列中就一个节点，那么通过自旋+CAS 修改的方式获取锁
              if ((h = whead) == (p = wtail)) {
                  for (long m, s, ns;;) {
                      // 如果未上写锁，则直接 CAS 修改，否则使用 tryIncReaderOverflow 修改
                      // 如果修改成功则返回版本号
                      if ((m = (s = state) & ABITS) < RFULL ?
                          U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                          (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L))
                          return ns;
                      // 如果上写锁了，那么自旋直到 CAS 修改成功
                      else if (m >= WBIT) {
                          if (spins > 0) {
                              if (LockSupport.nextSecondarySeed() >= 0)
                                  --spins;
                          }
                          else {
                              if (spins == 0) {
                                  WNode nh = whead, np = wtail;
                                  if ((nh == h && np == p) || (h = nh) != (p = np))
                                      break;
                              }
                              spins = SPINS;
                          }
                      }
                  }
              }
              // 初始化队列
              if (p == null) { // initialize queue
                  WNode hd = new WNode(WMODE, null);
                  if (U.compareAndSwapObject(this, WHEAD, null, hd))
                      wtail = hd;
              }
              else if (node == null)
                  node = new WNode(RMODE, p);
              // 如果头节点和前驱节点相同，或者前驱节点是写节点，跳出循环执行下一个大循环
              else if (h == p || p.mode != RMODE) {
                  if (node.prev != p)
                      node.prev = p;
                  else if (U.compareAndSwapObject(this, WTAIL, p, node)) {
                      p.next = node;
                      break;
                  }
              }
              // 表示已经存在读节点，新增的读节点添加到已经存在读节点的 cowait 队列上
              else if (!U.compareAndSwapObject(p, WCOWAIT,
                                               node.cowait = p.cowait, node))
                  node.cowait = null;
              else {
                  for (;;) {
                      WNode pp, c; Thread w;
                      // 如果头节点不为空且头节点 cowait 队列不为空，那么唤整个 cowait 队列（头节点的线程是当前持有锁的线程，应该都在运行中。）
                      if ((h = whead) != null && (c = h.cowait) != null &&
                          U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                          (w = c.thread) != null) // help release
                          U.unpark(w);
                      /**
                       * h == (pp = p.prev) 头结点是当前节点的前一个节点的前一个节点（快到自己了）
                       * h == p  头结点是当前节点的前一个节点（快到自己了）
                       * pp == null 当前节点的前一个节点的前一个节点是空（快到自己了）
                       * 只要不上写锁，当前线程就不断自旋修改版本号以便快速获取锁
                       */
                      if (h == (pp = p.prev) || h == p || pp == null) {
                          long m, s, ns;
                          do {
                              if ((m = (s = state) & ABITS) < RFULL ?
                                  U.compareAndSwapLong(this, STATE, s,
                                                       ns = s + RUNIT) :
                                  (m < WBIT &&
                                   (ns = tryIncReaderOverflow(s)) != 0L))
                                  return ns;
                          } while (m < WBIT);
                      }
                      /**
                       * 头结点未发生变化
                       * 并且指定节点的前一个节点的指向和 pp 属性指向一致
                       */
                      if (whead == h && p.prev == pp) {
                          long time;
                          /**
                           * pp = null（前面第二个元素时 null 表示快到自己了）
                           * h == p 前一个节点是头结点表示快到自己了
                           * p.status > 0 表示前一个节点已经取消等待，可能快到自己了
                           * 如果快到自己了，跳出当前循环，重新进行不断尝试获取锁的自旋
                           */
                          if (pp == null || h == p || p.status > 0) {
                              node = null;
                              break;
                          }
                          // 如果等待的时间已经到了，那么取消等待(调用 cancelWaiter)
                          if (deadline == 0L)
                              time = 0L;
                          else if ((time = deadline - System.nanoTime()) <= 0L)
                              return cancelWaiter(node, p, false);
                          Thread wt = Thread.currentThread();
                          U.putObject(wt, PARKBLOCKER, this);
                          node.thread = wt;
                          /**
                           * h != pp 头结点不是指定节点前的第二个节点
                           *  (state & ABITS) == WBIT 有个其他线程获取到了写锁
                           * whead == h 头节点未发生变化
                           * p.prev == pp 前一个节点和前前节点的指向未发生变化
                           * 如果没有快到当前线程(队列上距离当前线程还有 2 个以上的节点)，并且有其他线程获取了写锁那么阻塞队列
                           */
                          if ((h != pp || (state & ABITS) == WBIT) &&
                              whead == h && p.prev == pp)
                              U.park(false, time);
                          // 以下代码会在线程唤醒之后执行
                          // park 方式唤醒的线程可能是等待时间到了，或者线程被中断了，或者 unpark 方法执行了
                          node.thread = null;
                          U.putObject(wt, PARKBLOCKER, null);
                          // 如果是线程被中断，那么取消等待节点
                          if (interruptible && Thread.interrupted())
                              return cancelWaiter(node, p, true);
                      }
                  }
              }
          }
          // 如果头节点和前驱节点相同，或者前驱节点是写节点那么从第一个大循环中跳出执行本次自旋
          for (int spins = -1;;) {
              WNode h, np, pp; int ps;
              /**
               * 如果头节点是指定节点的前一个节点，那么自旋等待获取锁
               */
              if ((h = whead) == p) {
                  if (spins < 0)
                      spins = HEAD_SPINS;
                  else if (spins < MAX_HEAD_SPINS)
                      spins <<= 1;
                  for (int k = spins;;) { // spin at head
                      long m, s, ns;
                      /**
                       * (m = (s = state) & ABITS) < RFULL 表示未上写锁，并且读锁未溢出
                       * CAS 加上一个读锁 如果成功则返回版本号。
                       * 成功后协助唤醒头结点的 cowait 队列
                       * 失败则自旋
                       */
                      if ((m = (s = state) & ABITS) < RFULL ?
                          U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                          (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L)) {
                          WNode c; Thread w;
                          whead = node;
                          node.prev = null;
                          while ((c = node.cowait) != null) {
                              if (U.compareAndSwapObject(node, WCOWAIT,
                                                         c, c.cowait) &&
                                  (w = c.thread) != null)
                                  U.unpark(w);
                          }
    
                          return ns;
                      }
                      else if (m >= WBIT &&
                               LockSupport.nextSecondarySeed() >= 0 && --k <= 0)
                          break;
                  }
              }
              // 头结点不为空，那么协助唤醒头结点的 cowait 队列
              else if (h != null) {
                  WNode c; Thread w;
                  while ((c = h.cowait) != null) {
                      if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                          (w = c.thread) != null)
                          U.unpark(w);
                  }
              }
              // 头结点未发生变化
              if (whead == h) {
                  //如果前节点发生变化，那么更新指向关系
                  if ((np = node.prev) != p) {
                      if (np != null)
                          (p = np).next = node;   // stale
                  }
                  // 协助维护前节点的等待状态
                  else if ((ps = p.status) == 0)
                      U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
                  // 前节点如果是取消状态，将前节点从队列中删除
                  else if (ps == CANCELLED) {
                      if ((pp = p.prev) != null) {
                          node.prev = pp;
                          pp.next = node;
                      }
                  }
                  // 阻塞线程。通上个循环最后的线程阻塞。
                  else {
                      long time;
                      if (deadline == 0L)
                          time = 0L;
                      else if ((time = deadline - System.nanoTime()) <= 0L)
                          return cancelWaiter(node, node, false);
                      Thread wt = Thread.currentThread();
                      U.putObject(wt, PARKBLOCKER, this);
                      node.thread = wt;
                      if (p.status < 0 &&
                          (p != h || (state & ABITS) == WBIT) &&
                          whead == h && node.prev == p)
                          U.park(false, time);
                      node.thread = null;
                      U.putObject(wt, PARKBLOCKER, null);
                      if (interruptible && Thread.interrupted())
                          return cancelWaiter(node, node, true);
                  }
              }
          }
      }
    
      /**
       * 取消指定的节点，并将其从等待队列中删除。删除 cowait 队列中取消了的节点，并唤醒剩余 cowait 队列上的节点。检查是否有必要唤醒等待队列中的第一个等待节点（非第一个节点，非取消状态的节点）
       *
       * @param node if nonnull, the waiter 等待节点
       * @param group either node or the group node is cowaiting with 纵向队列上的等待节点
       * @param interrupted if already interrupted 是否响应中断
       * @return INTERRUPTED if interrupted or Thread.interrupted, else zero
       */
      private long cancelWaiter(WNode node, WNode group, boolean interrupted) {
          // 节点都不能为空
          if (node != null && group != null) {
              Thread w;
              node.status = CANCELLED;
              //将取消的节点从纵向(cowait)队列中剔除
              for (WNode p = group, q; (q = p.cowait) != null;) {
                  if (q.status == CANCELLED) {
                      U.compareAndSwapObject(p, WCOWAIT, q, q.cowait);
                      p = group; // restart
                  }
                  else
                      p = q;
              }
              // group 和 node 是一个节点。
              if (group == node) {
                  // 唤醒纵向队列（cowait）未取消的节点
                  for (WNode r = group.cowait; r != null; r = r.cowait) {
                      if ((w = r.thread) != null)
                          U.unpark(w);       // wake up uncancelled co-waiters
                  }
                  // 由于当前节点需要剔出队列，所以需要将当前节点的前一个未取消节点和后一个未取消节点重新绑定。绑定操作类似:node.prev.next = node.next;node.next.prev = node.pre;
                  for (WNode pred = node.prev; pred != null; ) { // unsplice
                      WNode succ, pp;        // find valid successor
                      while ((succ = node.next) == null ||
                             succ.status == CANCELLED) {
                          WNode q = null;    // find successor the slow way
                          for (WNode t = wtail; t != null && t != node; t = t.prev)
                              if (t.status != CANCELLED)
                                  q = t;     // don't link if succ cancelled
                          if (succ == q ||   // ensure accurate successor
                              U.compareAndSwapObject(node, WNEXT,
                                                     succ, succ = q)) {
                              if (succ == null && node == wtail)
                                  U.compareAndSwapObject(this, WTAIL, node, pred);
                              break;
                          }
                      }
                      if (pred.next == node) // unsplice pred link
                          U.compareAndSwapObject(pred, WNEXT, node, succ);
                      //唤醒当前节点下一个未取消节点(cowait 队列上的节点)
                      if (succ != null && (w = succ.thread) != null) {
                          succ.thread = null;
                          U.unpark(w);       // wake up succ to observe new pred
                      }
                      if (pred.status != CANCELLED || (pp = pred.prev) == null)
                          break;
                      node.prev = pp;        // repeat if new pred wrong/cancelled
                      U.compareAndSwapObject(pp, WNEXT, pred, succ);
                      pred = pp;
                  }
              }
          }
    
          // 检查是否需要唤醒后续节点
          WNode h;
          while ((h = whead) != null) {
              // q 是队列中非第一个节点并且处于等待状态或者初始状态的节点
              long s; WNode q; // similar to release() but check eligibility
              if ((q = h.next) == null || q.status == CANCELLED) {
                  for (WNode t = wtail; t != null && t != h; t = t.prev)
                      if (t.status <= 0)
                          q = t;
              }
              // 第一个节点未变化
              if (h == whead) {
                  // 如果 q 节点存在并且 h 的状态为初始化状态，并且当前不是写锁，并且 q 是读线程
                  // 满足以上条件则唤醒第一个等待节点(头节点的后续未取消的节点)
                  if (q != null && h.status == 0 &&
                      ((s = state) & ABITS) != WBIT && // waiter is eligible
                      (s == 0L || q.mode == RMODE))
                      release(h);
                  break;
              }
          }
          return (interrupted || Thread.interrupted()) ? INTERRUPTED : 0L;
      }
    
      // Unsafe mechanics
      private static final sun.misc.Unsafe U;
      private static final long STATE;
      private static final long WHEAD;
      private static final long WTAIL;
      private static final long WNEXT;
      private static final long WSTATUS;
      private static final long WCOWAIT;
      private static final long PARKBLOCKER;
    
      static {
          try {
              U = sun.misc.Unsafe.getUnsafe();
              Class<?> k = StampedLock.class;
              Class<?> wk = WNode.class;
              STATE = U.objectFieldOffset
                  (k.getDeclaredField("state"));
              WHEAD = U.objectFieldOffset
                  (k.getDeclaredField("whead"));
              WTAIL = U.objectFieldOffset
                  (k.getDeclaredField("wtail"));
              WSTATUS = U.objectFieldOffset
                  (wk.getDeclaredField("status"));
              WNEXT = U.objectFieldOffset
                  (wk.getDeclaredField("next"));
              WCOWAIT = U.objectFieldOffset
                  (wk.getDeclaredField("cowait"));
              Class<?> tk = Thread.class;
              PARKBLOCKER = U.objectFieldOffset
                  (tk.getDeclaredField("parkBlocker"));
    
          } catch (Exception e) {
              throw new Error(e);
          }
      }
```


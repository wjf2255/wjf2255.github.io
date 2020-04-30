---
title: JUC队列源码介绍-SynchronousBlockingQueue
tags: [Java, JUC, 源码, 多线程, 锁, 队列, SynchronousBlockingQueue]
layout: post
author: wjf
---

<div id="table-of-contents">
<h2>目录</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#org4b15594">介绍</a></li>
<li><a href="#orgb3a8842">原理</a>
<ul>
<li><a href="#org07a0449">栈 transfer 操作逻辑</a></li>
<li><a href="#org44aba7b">队列 transfer 操作逻辑</a></li>
<li><a href="#orgf1d9156">awaitFulfill</a></li>
</ul>
</li>
<li><a href="#org6c6d564">出队方法</a></li>
<li><a href="#org47df757">入队方法</a></li>
</ul>
</div>
</div>


<a id="org4b15594"></a>

# 介绍

SynchronounsBlockingQueue 是一种同步阻塞队列，每当有线程操作添加元素时，添加线程会一直阻塞直到有其他线程获取这个元素。SynchronounsBlockingQueue 不想其他的阻塞队列有个数组或者链表的数据结构存储元素，即阻塞队列的长度一直是 0。元素直接从添加线程转移到获取线程，所以 SynchronounsBlockingQueue 尽管也是继承自 Collection，但是迭代器遍历不到任何元素。也是由于这个原因，定义的用于返回但是不删除队列元素的 peek()方法永远都返回 Null。

SynchronounsBlockingQueue 实现了[dual stack and dual queue](http://www.cs.rochester.edu/u/scott/synchronization/pseudocode/duals.html)算法，以比较好的性能实现了元素从一个线程移到另外一个线程上。SynchronounsBlockingQueue 基于该算法提供了公平和非公平策略供选择使用。


<a id="orgb3a8842"></a>

# 原理

一般处理共享资源都是用锁来保护以保证线程安全，但是是用锁会增加额外的开销。在高性能场景下，一般会尽量避免是用锁。比如 StampedLock 会采用大量自旋和 CAS 修改的技术，最后才是用锁。

SynchronousQueue 也是采用类似的方式。SynchronousQueue 实现并扩展了[dual stack and dual queue](http://www.cs.rochester.edu/u/scott/synchronization/pseudocode/duals.html)算法，构建了两个底层数据结构，分别是用于实现非公平策略的栈和实现公平策略的队列。这两种策略模式性能表现上差距不大，通常情况下，公平策略下的线程的响应速度会高一些，而非公平策略能够支持更多的线程数。

虽然说 SynchronousQueue 的队列长度永远是 0，但是实际上底层的栈或者队列依然会存储线程获取元素或者添加元素的操作以及操作对应的元素。

SynchronousQueue 不管在什么时候都处于一下三种状态：

1.  date：线程操作入队
2.  request：线程操作出队或者底层队列（或者栈）是空队列
3.  fulfill：线程请求获取元素并且此时队列（或者栈）持有元素；线程操作入队并且此时有线程等待获取元素

SynchronousQueue 下的栈和队列在概念上有非常多相似的地方，但是为了能够独立向前栈和队列是独立实现的。SynchronousQueue 实现的算法细节和“dual stack and dual queue”论文中描述的算法有许多不一样的地方，尤其是关于取消操作。

1.  论文中的算法是基于 bit-marked pointers，SynchronousQueue 是基于单向列表节点的 bits 表示，目的是方便后续改进。
2.  SynchronousQueues 中入队和出队线程需要阻塞等待 fulfilled 状态。
3.  支持通过超时、线程中断探测取消等方式清除节点。

SynchronousQueue 的线程阻塞通过自旋或者 LockSuport 的 park/unpark 方法实现。如果节点位于 fulfilled 节点的下一个位置，那么通过自旋等待成为 fulfilled，否则使用 LockSupport 的 park 方法，是线程阻塞。

SynchronousQueue 对于添加元素和获取元素的操作，对栈和队列抽象出一个公共方法：E transfer(E e, boolean timed, long nanos)，底层公平策略基于队列实现 transfer，非公平策略基于栈实现 transfer。

```java
    /**
      * Transferer 是 dual stack 和 dual queue 算法抽象出来的父类
      * dual stacks 和 dual queues 有非常多相似的代码，但是为了能够让两者分别先前演进，实现上完全分开。
      * 但是由于实现的目的或者提供的功能是类似的，所以抽象出一个公共父类。或许将来可以共同演进的代码可以放到父类中。
      */
     abstract static class Transferer<E> {
         /**
          * 抽象出来的 put 和 take 操作。如果 e==null，那么是 take 操作；否则是 put 操作
          *
          * @param e 元素。如果元素不存在，那么是 request，否则是 date。
          * @param timed 操作是否需要超时
          * @param nanos 超时时间
          * @return 如果是 date 操作并且匹配到补足节点，那么返回补足元素。如果超时返回 null；如果是 request 操作，返回匹配到的元素。
          */
         abstract E transfer(E e, boolean timed, long nanos);
     }
```

<a id="org07a0449"></a>

## 栈 transfer 操作逻辑

循环处理一下三个操作：

1.  如果栈是空栈或者和本次操作的模式一致（date、request、fulfill），构建节点入栈。
2.  如果栈正好能够满足操作（比如 date 请求，结果能够匹配到一个 request），往栈中添加一个 fulfilled 节点，匹配到一个补足的节点，将这两个节点 pop 出栈。由于其他线程正在做第三部操作，该匹配操作不一定能够成功。
3.  如果栈顶节点已经匹配到 fulfill 状态的节点，那么线程优先处理栈顶节点的匹配操作，将栈顶元素和栈顶元素和它的 fulfill 状态的元素出栈，之后再处理自己的业务。

主要操作的 transfer 操作流程：
空队列下的添加和获取元素操作：
![take 流程程示意图](/assets/image/date_on_a_empty_stack.png)

空队列下的获取元素的操作：
![poll(long timeount, TimeUnit unit)流程示意图](/assets/image/poll流程示意图.png)

```java
    /**
     * dual stack 实现
     */
    static final class TransferStack<E> extends Transferer<E> {
    
         // 节点的模式。
         static final int REQUEST    = 0;
         static final int DATA       = 1;
         static final int FULFILLING = 2;
    
         // 判断节点的状态是否是 FULFILLING 模式
         static boolean isFulfilling(int m) { return (m & FULFILLING) != 0; }
    
         /**
          * 实现栈的底层数据结构
          */
         static final class SNode {
             // 下一个节点的引用
             volatile SNode next;
             // 补足节点
             volatile SNode match;
             // 操作请求的线程，方便其他线程唤醒
             volatile Thread waiter;
             // 元素。如果是 request 操作，元素时 null
             Object item;
             // 节点的模式
             int mode;
    
             SNode(Object item) {
                 this.item = item;
             }
    
             /*
              * CAS 更新节点的下一个节点引用，并返回是否更新成功
              */
             boolean casNext(SNode cmp, SNode val) {
                 return cmp == next &&
                     UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
             }
    
             /**
              * 匹配节点
              * CAS 将节点的 match 属性引用从 null 改成 s。查看自身节点 waiter 是否有指向具体线程，如果有将线程唤醒。
              *
              * @param s 匹配到的节点
              * @return true 如果匹配到节点，那么返回 true。
              */
             boolean tryMatch(SNode s) {
                 if (match == null &&
                     UNSAFE.compareAndSwapObject(this, matchOffset, null, s)) {
                     Thread w = waiter;
                     // waiter = null 说明添加节点的线程已经使用 LockSupport.park 挂起了
                     if (w != null) {
                         waiter = null;
                         LockSupport.unpark(w);
                     }
                     return true;
                 }
                 return match == s;
             }
    
             /**
              * 取消节点
              * 将 match CAS 从 null 改成自身。
              */
             void tryCancel() {
                 UNSAFE.compareAndSwapObject(this, matchOffset, null, this);
             }
    
             /**
              * 判断是否是取消节点。比较 match 和自身是否是同一个节点。
              */
             boolean isCancelled() {
                 return match == this;
             }
         }
    
         // 堆顶元素
         volatile SNode head;
    
         // CAS 修改堆顶元素。从 h 改成 nh
         boolean casHead(SNode h, SNode nh) {
             return h == head &&
                 UNSAFE.compareAndSwapObject(this, headOffset, h, nh);
         }
    
         /**
          * 构建或者重建 SNode 节点。由于竞争失败可能导致不在需要 SNode，所以延迟创建节点，直到需要用到的时候才创建，避免无需要的内存开销。
          */
         static SNode snode(SNode s, Object e, SNode next, int mode) {
             if (s == null) s = new SNode(e);
             s.mode = mode;
             s.next = next;
             return s;
         }
    
         /**
          * 添加或者获取元素。如果 e!=null 则是添加元素，否则是获取元素
          */
         @SuppressWarnings("unchecked")
         E transfer(E e, boolean timed, long nanos) {
             // 延迟初始化
             SNode s = null;
             // 模式
             int mode = (e == null) ? REQUEST : DATA;
    
             for (;;) {
                 SNode h = head;
                 // 如果是空栈或者模式相同 那么可能需要入队
                 if (h == null || h.mode == mode) {
                     // 如果 timed == true 并且 nanos <= 0 说明操作不需要等待或者已经超时，可能需要废弃节点
                     if (timed && nanos <= 0) {
                         // 如果栈顶非空并且是取消的节点，那么栈顶出栈
                         if (h != null && h.isCancelled())
                             casHead(h, h.next);
                         // 空栈时 put(E e)会走到这，直接退出
                         else
                             return null;
                     // 空栈并且操作需要等待，那么将栈顶从 null 改成新创建的节点
                     } else if (casHead(h, s = snode(s, e, h, mode))) {
                         // 等待其他元素匹配
                         SNode m = awaitFulfill(s, timed, nanos);
                         // 如果匹配到的元素时说明等待超时或者不需要等待直接退出
                         if (m == s) {
                             clean(s);
                             return null;
                         }
                         // 额外操作，帮助补足节点出栈。这个条件极少触发
                         if ((h = head) != null && h.next == s)
                             casHead(h, s.next);
                         return (E) ((mode == REQUEST) ? m.item : s.item);
                     }
                 // 如果栈顶元素不是 fulfill，那么从栈中匹配节点。如果匹配不到添加节点
                 } else if (!isFulfilling(h.mode)) {
                     // 如果栈顶节点已经取消，那么弹出栈顶元素
                     if (h.isCancelled())
                         casHead(h, h.next);
                     // 如果栈顶元素有效，那么构建新节点并入栈成为栈顶节点。新节点是 FULFILLING 状态
                     else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                         // 匹配补足节点
                         for (;;) {
                             // 如果有补足元素，那么栈顶节点的后一个节点就是补足节点
                             // 因为栈中的节点或者是所有的添加元素的节点（和还未来得及出栈的已经匹配到节点的节点），或者全部是获取元素的节点。所以第一个另外一种模式的节点入栈，那么后续的节点必然是非此模式的节点，否则就是模式一样的节点。
                             SNode m = s.next;       // m is s's match
                             // m == null 之前是有能匹配到的节点的，现在丢失了说明有其他现在操作，改变栈数据。
                             // 如果可匹配的节点丢失，那么退出本次循环重新走最外层循环逻辑
                             if (m == null) {
                                 // 将本次操作添加的栈顶元素弹出
                                 casHead(s, null);
                                 // 重新走外层循环时重新构建节点。
                                 s = null;
                                 break;
                             }
                             // 匹配到元素后，需要将栈顶和栈第二位置的节点一起弹出，需要将第三位置上的节点职位栈顶元素
                             SNode mn = m.next;
                             // 匹配校验。如果校验成功则并唤醒匹配到的节点的线程
                             if (m.tryMatch(s)) {
                                 // 将第三位置的元素设置为栈顶元素
                                 casHead(s, mn);
                                 // 如果是 request 操作，返回匹配到的元素，如果是 date 操作，返回当前节点的元素
                                 return (E) ((mode == REQUEST) ? m.item : s.item);
                             // 匹配校验失败则将栈第二位节点弹出栈
                             } else
                                 s.casNext(m, mn);
                         }
                     }
                 // 如果栈顶已经是 FULFILLING，那么帮助栈顶匹配补足节点。
                 // 额外操作，完成后继续走循环代码
                 } else {
                     SNode m = h.next;
                     if (m == null)
                         casHead(h, null);
                     else {
                         SNode mn = m.next;
                         if (m.tryMatch(h))
                             casHead(h, mn);
                         else
                             h.casNext(m, mn);
                     }
                 }
             }
         }
    
         /**
          * 等待匹配到补足元素
          *
          * @param s the waiting node
          * @param timed true if timed wait
          * @param nanos timeout value
          * @return matched node, or s if cancelled
          */
         SNode awaitFulfill(SNode s, boolean timed, long nanos) {
             // 等待超时时间
             final long deadline = timed ? System.nanoTime() + nanos : 0L;
             // 当前线程。如果线程需要被挂起，那么会将线程指给节点上的 waiter，方便后续匹配到补足元素后唤醒该线程
             Thread w = Thread.currentThread();
             //自旋次数
             int spins = (shouldSpin(s) ?
                          (timed ? maxTimedSpins : maxUntimedSpins) : 0);
             for (;;) {
                 // 如果线程被中断，那么取消节点（将匹配节点置为自身）。
                 if (w.isInterrupted())
                     s.tryCancel();
                 // 如果补足节点已经存在，那么直接返回补足节点（超时、线程中断等情况下会返回自身）
                 SNode m = s.match;
                 if (m != null)
                     return m;
                 // 如果设置的有超时时间，
                 if (timed) {
                     nanos = deadline - System.nanoTime();
                     if (nanos <= 0L) {
                         s.tryCancel();
                         continue;
                     }
                 }
                 // 判断是否需要继续自旋
                 if (spins > 0)
                     spins = shouldSpin(s) ? (spins-1) : 0;
                 // 自旋已经结束，此时还未找到补足节点，那么将线程指给节点
                 else if (s.waiter == null)
                     s.waiter = w;
                 // 如果没有设置超时时间，那么挂起线程
                 else if (!timed)
                     LockSupport.park(this);
                 // 设置了超时时间，那么计算挂起线程的等待时间并在这段时间内挂起线程
                 else if (nanos > spinForTimeoutThreshold)
                     LockSupport.parkNanos(this, nanos);
             }
         }
    
         /**
          * Returns true if node s is at head or there is an active
          * fulfiller.
          */
         boolean shouldSpin(SNode s) {
             SNode h = head;
             return (h == s || h == null || isFulfilling(h.mode));
         }
    
         /**
          * 将节点从栈中删除
          */
         void clean(SNode s) {
             s.item = null;
             s.waiter = null;
    
             // 后继节点
             SNode past = s.next;
             // 寻找有效的后继节点
             if (past != null && past.isCancelled())
                 past = past.next;
    
             // 设置有效的栈顶元素
             SNode p;
             while ((p = head) != null && p != past && p.isCancelled())
                 casHead(p, p.next);
    
             // 检查有效的栈顶元素和有效后继节点是否存在取消节点，如果存在则将其剔出栈
             while (p != null && p != past) {
                 SNode n = p.next;
                 // 如果节点取消，那么将节点剔出栈
                 if (n != null && n.isCancelled())
                     p.casNext(n, n.next);
                 // 遍历后继节点
                 else
                     p = n;
             }
         }
     }
```

<a id="org44aba7b"></a>

## 队列 transfer 操作逻辑

循环处理一下两个操作：

1.  如果栈是空栈或者和本次操作的模式一致（date、request、fulfill），构建节点入栈
2.  如果栈正好能够满足操作（比如 date 请求，结果能够匹配到一个 request），利用 CAS 方式将队列中匹配到的节点更新成 fulfill 状态，并将节点出队。


<a id="orgf1d9156"></a>

## awaitFulfill

transfer 方法根据是否有元素的操作判断该操作是 date 还是 request，并依赖 awaitFulfill 处理等待匹配到 fulfill 操作。awaitFulfill 循环处理一下操作：

1.  判断是否中断，如果中断则取消对应节点
2.  如果已经匹配到不足的节点，返回补足的节点
3.  如果等待超时，那么取消线程（取消线程会把节点的不足节点置为自身）
4.  在自旋次数未达到一定程度前，自旋等待补足节点
5.  超过自旋次数，将当前线程的引用指给节点的 waiter，方便其他线程匹配唤醒当前线程（该线程将会被 LockSupport 挂起）
6.  LockSupport 挂起当前线程

```java
    /**
     * dual queue 的实现类
     */
    static final class TransferQueue<E> extends Transferer<E> {
        /**
         * queue 是通过单向链表实现。QNode 是实现单向链表的节点
         */
        static final class QNode {
            // 下一个节点
            volatile QNode next;
            // 元素
            volatile Object item;
            // 添加节点的线程
            volatile Thread waiter;
            // 节点的模式。request、data、fullled
            final boolean isData;
    
            QNode(Object item, boolean isData) {
                this.item = item;
                this.isData = isData;
            }
    
            boolean casNext(QNode cmp, QNode val) {
                return next == cmp &&
                    UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
            }
    
            boolean casItem(Object cmp, Object val) {
                return item == cmp &&
                    UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
            }
    
            /**
             * 设置为取消节点。将元素从 cmp 改成节点自身。
             */
            void tryCancel(Object cmp) {
                UNSAFE.compareAndSwapObject(this, itemOffset, cmp, this);
            }
    
            /**
             * 判断是否是取消节点。判断的依据是如果元素时节点自身那么就是取消节点。
             */
            boolean isCancelled() {
                return item == this;
            }
    
            /**
             * 判断节点是否离开队列。
             * 判断的依据是下一个节点的指向指向自身
             */
            boolean isOffList() {
                return next == this;
            }
        }
    
        // 队列头节点
        transient volatile QNode head;
        // 队列尾节点
        transient volatile QNode tail;
        /**
         * 取消但是还未从队列中剔除的节点引用
         */
        transient volatile QNode cleanMe;
    
        /**
         * 构造方法，初始化一个虚拟节点或者头结点。当队列是空队列的时候，队尾指针也指向该节点。
         */
        TransferQueue() {
            QNode h = new QNode(null, false);
            head = h;
            tail = h;
        }
    
        /**
         * CAS 修改队首节点，并将老节点的下一个节点的引用置空。
         */
        void advanceHead(QNode h, QNode nh) {
            if (h == head &&
                UNSAFE.compareAndSwapObject(this, headOffset, h, nh))
                h.next = h; // forget old next
        }
    
        /**
         * CAS 修改队尾节点
         */
        void advanceTail(QNode t, QNode nt) {
            if (tail == t)
                UNSAFE.compareAndSwapObject(this, tailOffset, t, nt);
        }
    
        /**
         * CAS 修改取消但是没有提出队列的节点
         */
        boolean casCleanMe(QNode cmp, QNode val) {
            return cleanMe == cmp &&
                UNSAFE.compareAndSwapObject(this, cleanMeOffset, cmp, val);
        }
    
        /**
         * 添加和获取元素的实现
         */
        @SuppressWarnings("unchecked")
        E transfer(E e, boolean timed, long nanos) {
    
            // 延迟初始化
            QNode s = null;
            // 本次添加节点的模式 queue 中只有
            boolean isData = (e != null);
    
            for (;;) {
                // 尾节点
                QNode t = tail;
                // 头节点
                QNode h = head;
                // 队列初始化时会构建一个虚拟节点，t == null || h == null 说明还未初始化完成，那么自旋等待初始化完成
                // 在 SynchronousQueue 中，这种情况不会发生
                if (t == null || h == null)
                    continue;
                // 如果是空队列或者是和尾结点拥有相同的模式，那么做入队操作，添加到队尾。
                if (h == t || t.isData == isData) {
                    // 队尾的下一个节点指向
                    QNode tn = t.next;
                    // t != tail 说明队列发生变化，那么重新走 for 循环，获取最新的队尾节点
                    if (t != tail)
                        continue;
                    // 如果队尾的下一个节点指向的不是 null，那么将下一个节点更新成最新的队尾，然后重新走 for 循环，获取最新的队尾节点
                    if (tn != null) {
                        advanceTail(t, tn);
                        continue;
                    }
                    // 如果设置为有超时时间，并且超时时间<=0，那么直接返回 null 结束操作。
                    // 这段代码一般是由于是调用 transfer 设置为不需要等待导致(offer(E e)但是是空队列)或者已经超过了等待时间。
                    if (timed && nanos <= 0)        // can't wait
                        return null;
                    // 如果节点未初始化，那么初始化节点
                    if (s == null)
                        s = new QNode(e, isData);
                    // CAS 修改队尾的下一个节点的属性，改成。如果修改失败表示队尾节点已经发生变化，那么从新走 for 循环，获取最新的队尾节点
                    if (!t.casNext(null, s))
                        continue;
                    // 修改队列的队尾节点，从 t 改成 s。
                    advanceTail(t, s);
                    // 等待匹配补足节点
                    Object x = awaitFulfill(s, e, timed, nanos);
                    // x == s 说明超时或者线程中断，将节点提出队列
                    if (x == s) {
                        clean(t, s);
                        return null;
                    }
                    // 此时已经配到到正确的结果，节点已经被取消。
                    // 如果此时节点还未出队（s.next != s），那么将 s 出队
                    if (!s.isOffList()) {
                        // 如果尾结点是头结点的话（空队列），那么将 s 置为头结点（头结点是虚节点，相当于 s 出队）
                        advanceHead(t, s);
                        // x != null 已经匹配过补足元素。如果没有匹配到则 x == s 否则 s != s。到这一般是匹配到元素了。那么将 s 取消（s.item = s）
                        if (x != null)
                            s.item = s;
                        s.waiter = null;
                    }
                    // 返回匹配到的节点的数据。如果是 date 操作，返回 null，如果是 request 返回补足点的 item 数据。
                    return (x != null) ? (E)x : e;
                // 队列中存在补足节点，那么寻找补足节点
                } else {
                    // 队列中第一个有效的节点，也是计划中的补足节点
                    QNode m = h.next;
                    // 如果队列发生变化，那么从新中循环获取最新的队列信息。
                    if (t != tail || m == null || h != head)
                        continue;
                    // 补足节点的数据
                    Object x = m.item;
                    /**
                     * 满足以下任一条件则重新走循环代码
                     * 1. m 节点已经是 fulfilled (被其他线程匹配)
                     * 2. m 节点被取消了（超时取消）
                     * 3. CAS 将 m 的 item 改成 e 失败（(被其他线程匹配或者取消超时）
                     */
                    if (isData == (x != null) ||    // m already fulfilled
                        x == m ||                   // m cancelled
                        !m.casItem(x, e)) {         // lost CAS
                        // 以上条件满足说明 m 节点需要出队。这里利用 CAS 将队首节点改成 m 节点，帮助其出队。
                        advanceHead(h, m);          // dequeue and retry
                        continue;
                    }
                    // 到这说明 m 是补足节点，将 m 出队
                    advanceHead(h, m);
                    // 帮助唤醒补足节点的线程
                    LockSupport.unpark(m.waiter);
                    // 返回匹配到的节点的数据。如果是 date 操作，返回 null，如果是 request 返回补足点的 item 数据。
                    return (x != null) ? (E)x : e;
                }
            }
        }
    
        /**
         * 等待匹配补足节点
         * 和 stack 的 awaitFulfill 的思路一致
         *
         * @param s 等待匹配补足节点的节点
         * @param e 节点的元素
         * @param timed 是否有超时设置。如果是那么需要在超时时间<=0 时，不再继续等待
         * @param nanos 超时时间
         * @return 如果匹配到补足节点，那么返回补足节点的元素，否则返回自身
         */
        Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Thread w = Thread.currentThread();
            int spins = ((head.next == s) ?
                         (timed ? maxTimedSpins : maxUntimedSpins) : 0);
            for (;;) {
                if (w.isInterrupted())
                    s.tryCancel(e);
                Object x = s.item;
                if (x != e)
                    return x;
                if (timed) {
                    nanos = deadline - System.nanoTime();
                    if (nanos <= 0L) {
                        s.tryCancel(e);
                        continue;
                    }
                }
                if (spins > 0)
                    --spins;
                else if (s.waiter == null)
                    s.waiter = w;
                else if (!timed)
                    LockSupport.park(this);
                else if (nanos > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanos);
            }
        }
    
        /**
         * 将节点剔出队列
         * @param pred 将被踢出队列的节点的上一个节点
         * @param s 将被踢出队列的节点
         */
        void clean(QNode pred, QNode s) {
            // 置空 waiter 属性
            s.waiter = null;
            // 如果是 s 不是队尾节点，那么将 s 从队列中删除，否则将 pred 指给 cleanMe，并将上一个 cleanMe 指向的取消节点剔出队列
            while (pred.next == s) {
                QNode h = head;
                // 第一个有效节点。队列初始化时会初始化一个虚拟节点。
                QNode hn = h.next;
                // 如果第一个有效节点不为空并且是取消的节点，那么将其更新成为头结点，重新走循环
                if (hn != null && hn.isCancelled()) {
                    advanceHead(h, hn);
                    continue;
                }
                QNode t = tail;
                // 如果头尾节点是同一个节点，说明队列是空队列结束循环
                if (t == h)
                    return;
                // 尾结点的下一个节点
                QNode tn = t.next;
                // 如果尾结点发生变化，那么重新走循环获取最新的尾结点
                if (t != tail)
                    continue;
                // 如果尾结点的下一个节点不是 null，那么将其更新成最新的尾结点
                if (tn != null) {
                    advanceTail(t, tn);
                    continue;
                }
                // 如果取消的节点不是尾结点，那么将 s 的上一个节点的下一个节点指向改成 s 的下一个节点（节点从单向链表中删除）并结束循环
                if (s != t) {
                    QNode sn = s.next;
                    if (sn == s || pred.casNext(s, sn))
                        return;
                }
                // 获取到上一次做 clean 操作时的 pred 节点。
                QNode dp = cleanMe;
                // dp != null 那么需要将上一次做取消操作但是未从队列中删除的节点本次需要从队列中删除
                if (dp != null) {    // Try unlinking previous cancelled node
                    // 上一次被取消的节点
                    QNode d = dp.next;
                    QNode dn;
                    /*
                     * 判断条件如果满足，那么将 cleanMe CAS 修改成 null
                     * 1. d == null 取消节点已经消失
                     * 2. d == dp 取消节点已经消失，并且 dp 也是取消的节点
                     * 3. d.isCancelled() == false，d 现在不是取消节点
                     * 4. d != t d 节点不是尾结点（尾结点不做从队列剔出的操作）
                     * 5. (dn = d.next) != null d 节点有后续节点（d 节点非尾结点--尾结点不从队列删除）
                     * 6. dp.casNext(d, dn) == true CAS 将 d 节点剔出队列操作成功
                     */
                    if (d == null ||               // d is gone or
                        d == dp ||                 // d is off list or
                        !d.isCancelled() ||        // d not cancelled or
                        (d != t &&                 // d not tail and
                         (dn = d.next) != null &&  //   has successor
                         dn != d &&                //   that is on list
                         dp.casNext(d, dn)))       // d unspliced
                        casCleanMe(dp, null);
                    // dp == pred 说明 cleanMe 已经存储过了直接退出循环
                    if (dp == pred)
                        return;
                // cleanMe 不存在，将 cleanMe 从 null 改成 pred。修改成功则退出否则重新走循环
                } else if (casCleanMe(null, pred))
                    return;
            }
        }
    }
```

<a id="org6c6d564"></a>

# 出队方法

SynchronousQueue 提供了三个出队方法：

1.  public E take() throws InterruptedException：该方法会一直阻塞线程直到其他线程添加元素或者线程中断。
2.  public E poll(long timeout, TimeUnit unit) throws InterruptedException：该方法支持设置超时时间。如果超过超时时间还未获取到数据或者等待过程中贤臣刚被中断，那么线程不会继续挂起。如果超时时间设置为0，队列如果有补足节点那么返回补足节点的元素，否则直接结束，返回null。
3.  public E poll()：通2中超时时间设置为0的情况。

```java
    /**
     * 等待获取元素。
     * 该方法会一直阻塞知道获取到元素或者等待线程被中断。
     *
     * @return the head of this queue
     * @throws InterruptedException {@inheritDoc}
     */
    public E take() throws InterruptedException {
        // 基于阻塞并且无超设置的transfer方法获取元素。
        E e = transferer.transfer(null, false, 0);
        // e != null 说明获取到元素，那么直接返回，否则就是等待线程被中断了。
        if (e != null)
            return e;
        // LockSupport的park方法当遇到线程中断时会将线程唤醒，并将线程置为未取消中断的状态。所以这几继续将线程置为中断并抛出中断异常。
        Thread.interrupted();
        throw new InterruptedException();
    }
    
    /**
     * 获取元素
     * 该方法会在设置的超时时间等一直等待获取元素。如果超时还未获取到元素那么返回null。
     *
     * @return the head of this queue, or {@code null} if the
     *         specified waiting time elapses before an element is present
     * @throws InterruptedException {@inheritDoc}
     */
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        // 基于可超时的transfer方法等待获取元素
        E e = transferer.transfer(null, true, unit.toNanos(timeout));
        // 如果线程为中断获取获取到元素那么返回元素，否则抛出中断遗产。
        if (e != null || !Thread.interrupted())
            return e;
        throw new InterruptedException();
    }
    
    /**
     * 获取元素
     * 该方法只有在同步队列中有其他线程添加元素并被阻塞时才会返回元素，否则返回null。
     * 该方法不会阻塞线程。
     *
     * @return the head of this queue, or {@code null} if no
     *         element is available
     */
    public E poll() {
        return transferer.transfer(null, true, 0);
    }
```

<a id="org47df757"></a>

# 入队方法

SynchronousQueue 提供了三个入队方法：

1.  public void put(E e) throws InterruptedException：线程会一直阻塞直到有其他线程获取对应的元素或者被中断。
2.  public boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException：线程在设置的超时时间内会一直阻塞直到超时获取有其他线程获取到对应的元素或者被中断。
3.  public boolean offer(E e)：该方法不阻塞线程。如果已经有其他线程等待获取元素，那么该方法添加元素成功，否则直接失败。

```java
    /**
     * 添加元素
     * 线程会一直阻塞直到有其他线程获取到该元素或者线程被中断
     * 该方法不支持添加null元素。如果元素等于null，抛出NullPointerException异常
     *
     * @throws InterruptedException {@inheritDoc}
     * @throws NullPointerException {@inheritDoc}
     */
    public void put(E e) throws InterruptedException {
        //检查元素是否合法
        if (e == null) throw new NullPointerException();
        // 基于阻塞并且无超时时间限制的transfer添加元素。如果返回null说明线程被中断。
        if (transferer.transfer(e, false, 0) == null) {
            // LockSupport.park方法阻塞的线程如果被中断，那么线程会被唤醒并且职位未中断。此时将线程置为中断，并抛出中断异常
            Thread.interrupted();
            throw new InterruptedException();
        }
    }
    
    /**
     * 添加元素
     * 线程会在超时时间内一直等待添加元素直到获取到元素或者超时或者线程被中断。
     * 不支持null元素
     *
     * @return {@code true} if successful, or {@code false} if the
     *         specified waiting time elapses before a consumer appears
     * @throws InterruptedException {@inheritDoc}
     * @throws NullPointerException {@inheritDoc}
     */
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {
        // 校验元素有效性
        if (e == null) throw new NullPointerException();
        // 基于超时时间的transfer方法添加元素。如果获取的元素不等于null，说明添加成功，返回true。
        if (transferer.transfer(e, true, unit.toNanos(timeout)) != null)
            return true;
        // 如果线程未被中断说明是超时，那么返回false
        if (!Thread.interrupted())
            return false;
        // 说明线程被中断。
        throw new InterruptedException();
    }
    
    /**
     * 添加元素
     * 只有同步队列中存在等待获取元素的线程，该方法添加元素才能成功，否则直接添加失败返回false
     *
     * @param e the element to add
     * @return {@code true} if the element was added to this queue, else
     *         {@code false}
     * @throws NullPointerException if the specified element is null
     */
    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        return transferer.transfer(e, true, 0) != null;
    }
```

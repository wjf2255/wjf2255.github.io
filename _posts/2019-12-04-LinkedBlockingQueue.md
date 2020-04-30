---
title: JUC队列源码介绍-LinkedBlockingQueue
tags: [Java, JUC, 源码, 多线程, 锁, 队列, LinkedBlockingQueue]
layout: post
author: wjf
---

<div id="table-of-contents">
<h2>目录</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#org44b71d7">介绍</a></li>
<li><a href="#orgdd1d626">数据存储设计</a></li>
<li><a href="#orgfc80c74">构造方法介绍</a></li>
<li><a href="#orgf4c542c">入队方法介绍</a></li>
<li><a href="#org123af8d">出队方法介绍</a></li>
<li><a href="#org34cc398">迭代器介绍</a></li>
<li><a href="#org429ae36">其他方法介绍</a></li>
</ul>
</div>
</div>


<a id="org44b71d7"></a>

# 介绍

LinkedBlockingQueue 是一个基于单向链表和双锁队列算法实现的先进先出的同步队列。头部元素是最早进入队列的元素，尾部元素时最晚进入队列的元素。在添加元素时，将会在单向链表的末尾添加元素。

ArrayBlockingQueue 由于是循环数组，每次出队操作需要将后续的元素前移，导致入队的位置会发生变化，从而出队时，入队操作无法并发执行。ArrayBlockingQueue 中的入队和出队操作由一个重入锁控制，不能并发执行。LinkedBlockingQueue 由于采用的链表，出队只更新队首元素和队首指向，入队只更新队尾元素和队尾指向，两个操作互相独立，所以采用双锁策略。即入队和出队可以同时执行。由于此种策略，LinkedBlockingQueue 的性能一般优于 ArrayBlockingQueue。

通 ArrayBlockingQueue 一样 LinkedBlockingQueue 初始化不指定容量时，默认是 Integer.MAX<sub>VALUE</sub>。同时由于继承关系也属于集合（Collection)的子类，拥有迭代器的功能。继承关系如下：
![img](/assets/image/inkedBlockingQueue.png)


<a id="orgdd1d626"></a>

# 数据存储设计

LinkedBlockingQueue 通过单向链表存储数据。代码如下：

```java
    /**
     * 构成单向链表的节点
     */
    static class Node<E> {
    
        // 数据对象
        E item;
    
        /**
         * 下一节点的指向
         */
        Node<E> next;
    
        // 节点的构造函数
        Node(E x) { item = x; }
    }
```

LinkedBlockingQueue 初始化时会构建一个 item 为 null 的节点，队首（head）和队尾（tail）都指向这个节点。如果节点添加元素时，将会创建一个新的节点，队首节点的 next 指向新创建的节点，并且队尾节点指向新创建的节点。如果出队时，如果队首的下下个节点存在，则队首节点的 next 指向下下个节点，否则指向空。队尾如果指向的是这个被删除的节点，那么指向队首节点。

![img](/assets/image/linked_node.png)

由于是单向链表，所以任何遍历或者查找元素都需要从队首开始查找。

LinkedBlockingQueue 有个属性 private final int capacity 表示阻塞队列容量上限。这个属性在阻塞队列初始化时就确定，使用中不会发生变化。往队列中添加元素时，每次都需要比较是否达到上限。只有未达到上限时才能将元素添加到队列中。

LinkedBlockingQueue 还一个重要属性表示队列的长度：private final AtomicInteger count。每次往队列添加元素后该值加一；出队后该值减一。由于入队和出队可以并发执行，所以为了保证线程安全使用 AtomicInteger。


<a id="orgfc80c74"></a>

# 构造方法介绍

构造方法定义了 LinkedBlockingQueue 如何被初始化。这个是使用 LinkedBlockingQueue 的第一步。

```java
    /**
     * 构造一个 LinkedBlockingQueue。
     * 无参构造函数会利用指定容量的构造器完成初始化，给定的容量时 Integer.MAX_VALUE。
     */
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }
    
    /**
     * 构造一个 LinkedBlockingQueue
     * 根据给定的容量初始化一个 LinkedBlockingQueue：
     * 1. 校验 capacity 的有效性
     * 2. 初始化一个 Node 节点，tail 和 head 都指向这个节点。
     *
     * @param capacity the capacity of this queue
     * @throws IllegalArgumentException if {@code capacity} is not greater
     *         than zero
     */
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }
    
    /**
     * 根据给定一个集合初始化一个 LinkedBlockingQueue
     * 1. 利用指定容量的构造方法初始化一个容量为 Integer.MAX_VALUE 的 LinkedBlockingQueue
     * 2. 遍历集合，将集合中的元素入队到新创建的 LinkedBlockingQueue 中。
     * 3. 2 的操作前后需要使用入队锁上锁和释放锁以确保阻塞队列线程安全
     *
     * @param c the collection of elements to initially contain
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
    public LinkedBlockingQueue(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // Never contended, but necessary for visibility
        try {
            int n = 0;
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
                if (n == capacity)
                    throw new IllegalStateException("Queue full");
                enqueue(new Node<E>(e));
                ++n;
            }
            count.set(n);
        } finally {
            putLock.unlock();
        }
    }
```

由于使用未指定容量的构造方法，阻塞队列的长度限制都是 Integer.MAX<sub>VALUE</sub>，容易导致内存异常。所以没有特殊情况都需要规划一个合理的容量，利用指定容量的构造方法初始化 LinkedBlockingQueue。

由于阻塞队列本身也是一个集合，所以第三个构造函数可以很方便的根据另外一个阻塞队列已有的数据生成一个新的阻塞队列。但是如果此时另外一个阻塞队列还在处理业务，尤其是 ArrayBlockingQueue，可能会导致迭代器过期从而无法完全复制阻塞队列的元素。


<a id="orgf4c542c"></a>

# 入队方法介绍

LinkedBlockingQueue 有三个入队的方法，分别提供了

1.  入队如果队列已经满了那么阻塞等待
2.  入队如果队列已经满了那么等待，但是可以设置超时时间
3.  入队如果已经满了，那么不等待直接返回入队失败

具体的代码如下：
```java
    /**
     * 将给定元素入队到队列中
     * 如果给定的元素是 null，那么抛出 NullPointerException。
     * 如果入队锁在等待过程中线程被中断，那么抛出 InterruptedException
     *
     * @throws InterruptedException {@inheritDoc}
     * @throws NullPointerException {@inheritDoc}
     */
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        // 队列长度
        int c = -1;
        // 为指定元素构造一个节点
        Node<E> node = new Node<E>(e);
        // 入队锁
        final ReentrantLock putLock = this.putLock;
        // 队列长度
        final AtomicInteger count = this.count;
        // 操作前先上锁
        putLock.lockInterruptibly();
        try {
            /**
             * 如果当前队列长度达到容量上限，那么线程需要进入等待队列，等待队列未满通知。
             * 由于在此之前已经执行过入队锁上锁，所以此处的 count 只能减少（其他线程出队操作）。
             * 由于 notFull.await()会唤醒阻塞在 putLock.lockInterruptibly 上的线程，
             * 所以可能有多个线程阻塞在 notFull.await()，所以使用 while 方法判断是否需要继续阻塞
             */
            while (count.get() == capacity) {
                notFull.await();
            }
            // 节点添加到单向链表尾部
            enqueue(node);
            // 队列长度加一
            c = count.getAndIncrement();
            // 由于 c 返回的是加之前的队列长度，所以用 c+1 表示最新的队列长度。如果队列长度小于容量上限，那么队列未满信号。
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        // c == 0 表示当前队列在做本次入队前是空队列，那么此时需要发布一个队列不是空队列信号
        // 该操作能够及时通知到在空队列上获取元素受阻的线程。
        // 只有队列从没有元素编程有元素时才需要发送该消息。
        if (c == 0)
            signalNotEmpty();
    }
    
    /**
     * 将给定元素入队到队列中
     * 如果由于队列慢了无法添加到元素中而进度等待队列，允许设置等待的超时时间
     * 如果入队成功则返回 true，入队失败则返回 false
     * 其他特性和上一个添加方法类似
     *
     * @return {@code true} if successful, or {@code false} if
     *         the specified waiting time elapses before space is available
     * @throws InterruptedException {@inheritDoc}
     * @throws NullPointerException {@inheritDoc}
     */
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {
    
        if (e == null) throw new NullPointerException();
        // 转换时间。该时间是现在等待队列上的等待超时时间
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                // 如果超时时间小于等于 0 则直接返回入队失败。
                if (nanos <= 0)
                    return false;
                // 设置在等待队列中的等待超时时间。超时时，nanos 变量更新成 0，while 下一次循环时方法返回 false
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(new Node<E>(e));
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return true;
    }
    
    /**
     * 元素入队。
     * 该入队方法不会由于队列慢无法入队而导致线程阻塞。
     * 如果入队成功则返回 true，否则返回 false
     *
     * @throws NullPointerException if the specified element is null
     */
    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        // 如果队列已满，那么直接入队失败返回 false。
        if (count.get() == capacity)
            return false;
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            // 通过队列长度未满做再次校验，校验是否允许入队。上一个校验 count.get() == capacity 和上锁之间很可能有其他的入队操作导致上一次校验过时。
            if (count.get() < capacity) {
                enqueue(node);
                c = count.getAndIncrement();
                if (c + 1 < capacity)
                    notFull.signal();
            }
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        // 由于初始值是-1，如果 c >= 0 那么说明已经执行过 c = count.getAndIncrement()，即已经执行入队操作
        return c >= 0;
    }
```

<a id="org123af8d"></a>

# 出队方法介绍

LinkedBlockingQueue 提供了 3 个出队方法和一个获取队首元素的方法。3 个方法的大致功能时

1.  将队首元素出队。如果队列是空队列，那么线程阻塞直到获得队列未空通知（notEmpty.signal())
2.  将队首元素出队。如果队列是空队列，那么在设置的超时时间内等待，超过超时时间则返回 null
3.  将队首元素出队。如果出队失败则返回 null。如果是空队列则返回 null。

获取队首元素的方法不会改变队列，只是将具体值的引用返回。如果队列是空队列则返回 null。

具体代码如下：
```java
    /**
     * 队首元素出队
     * 如果队列是控队列，那么当前线程会进入等待队列，直到收到 notEmpty.signal()通知
     */
    public E take() throws InterruptedException {
        // 出队的元素
        E x;
        // 队列长度的方法私有属性。主要用于标记是否发生改变。
        int c = -1;
        // 队列长度
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        // 出队锁上锁
        takeLock.lockInterruptibly();
        try {
            // 如果队列是空的那么线程进入等待队列。while 的用户和出队类似。
            while (count.get() == 0) {
                notEmpty.await();
            }
            // 元素出队，并将元素赋值给 x。如果出队成功则返回具体的元素否则返回初始值 null。
            x = dequeue();
            // 队列元素减一并将更新前的值赋值给 c，如此则 c >= 0 则表示出队成功
            c = count.getAndDecrement();
            // 如果 c > 1 那么表示队列在出队后还有元素在队列中，发送 notEmpty.signal()信号
            if (c > 1)
                notEmpty.signal();
        } finally {
            // 释放出队锁
            takeLock.unlock();
        }
        // 如果出队前的队列长度和容量上限一致，那么出队后做队列未满通知。
        // 队列未满通知在出队方法中只有此种情况才发送信号，以便及时通知到由于队列已满导致入队阻塞的线程。
        if (c == capacity)
            signalNotFull();
        return x;
    }
    
    /**
     * 队首元素出队
     * 如果队列是控队列，那么当前线程会进入等待队列，直到收到 notEmpty.signal()通知或者等待时间超过了设置的超时时间
     */
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E x = null;
        int c = -1;
        long nanos = unit.toNanos(timeout);
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                // 如果等待时间超过了超时时间，那么会更新 nanos，更新成<=0，那么直接返回 null。
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
    
    /**
     * 队首元素出队
     * 如果队列是控队则直接返回 null，否则将队首元素直接出队。该方法不会犹豫队列是空队列而阻塞线程。
     */
    public E poll() {
        final AtomicInteger count = this.count;
        // 空队列直接返回 null
        if (count.get() == 0)
            return null;
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            // 再次确认是否是空队列。这是由于上一次的直接校验和出队锁上锁期间可能由于其他线程出队操作导致队列已经发生变化
            if (count.get() > 0) {
                x = dequeue();
                c = count.getAndDecrement();
                // 队列如果出队后还有元素则发送 notEmpty.signal()信号
                if (c > 1)
                    notEmpty.signal();
            }
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
    
    /**
     * 获取队首元素
     * 该方法不改变队列，只是返回队首元素的对象引用地址。
     */
    public E peek() {
        if (count.get() == 0)
            return null;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            Node<E> first = head.next;
            // 如果队列是空队列，那么返回 null，否则返回队首元素的引用地址。
            if (first == null)
                return null;
            else
                return first.item;
        } finally {
            takeLock.unlock();
        }
    }
```

<a id="org34cc398"></a>

# 迭代器介绍

由于 LinkedBlockingQueue 也是继承自 Collection，所以可以使用迭代器 Interator 遍历队列和使用迭代器删除元素。

LinkedBlockingQueue 利用内部类 Itr 实现迭代器功能。iterator()方法每次初始化一个 Itr 实例。具体的 Itr 代码如下：

```java
     /**
      * LinkedBlockingQueue 迭代器
      * LinkedBlockingQueue 通过内部类 Itr 实现迭代器功能，方法 iterator()返回 Itr 的实例化对象。
      * Itr 持有 currentElement 当前元素的引用，current 当前节点的引用，lastRet 最后一次遍历到的元素的引用。
      * 在初始化时 Itr 会初始化 current，队列有元素时还会初始化 currentElement。
      * 如果在创建迭代器后队列被清空，由于属性已经生成好，所以 hasNext 和 next()不能够反映最新的变化。
      */
     private class Itr implements Iterator<E> {
    
        // 初始化队列第一个有效节点，遍历时指向下一个节点
        private Node<E> current;
        // 迭代器遍历时最后遍历到的节点
        private Node<E> lastRet;
        // current 节点持有的元素
        private E currentElement;
    
        /**
         * 迭代器构造函数
         * 初始化 current，指向 head.next。如果队列是空队列，那么 currentElement = null
         * 如果不是空队列，初始化 currentElement = current.item。
         */
        Itr() {
            fullyLock();
            try {
                current = head.next;
                if (current != null)
                    currentElement = current.item;
            } finally {
                fullyUnlock();
            }
        }
    
        /**
         * 判断是否还有下一个元素，true 表示还有下一个元素，false 表示没有更多元素
         * current 指向的是下一个节点，所以直接根据 current!= null 来判断是否有下一个节点。
         */
        public boolean hasNext() {
            return current != null;
        }
    
        /**
         * 返回指定节点的下一个有效节点
         *
         */
        private Node<E> nextNode(Node<E> p) {
            // 遍历后续节点知道找到下一个有效节点
            for (;;) {
                Node<E> s = p.next;
                // s == p 表示此处的链表错误，构成了循环，返回第一个有效元素退出循环。
                if (s == p)
                    return head.next;
                // s == null 说明链表已经到达最后一个节点；s.item != null 说明是有效的下一个节点
                if (s == null || s.item != null)
                    return s;
                // 说明链表正常并且未找到有效的节点，那么继续遍历下一个节点。
                p = s;
            }
        }
    
        /**
         * 迭代器后移一个位置并返回下一个节点
         */
        public E next() {
            fullyLock();
            try {
                if (current == null)
                    throw new NoSuchElementException();
                E x = currentElement;
                lastRet = current;
                current = nextNode(current);
                currentElement = (current == null) ? null : current.item;
                return x;
            } finally {
                fullyUnlock();
            }
        }
    
        /**
         * 将迭代器指向的元素剔出队列
         * 遍历队列找到迭代器私有属性 lastRet 指向的节点。如果存在利用 unlink 方法，将节点剔出队列。
         */
        public void remove() {
            if (lastRet == null)
                throw new IllegalStateException();
            fullyLock();
            try {
                Node<E> node = lastRet;
                lastRet = null;
                // 遍历链表寻找 lastRet 节点的位置
                for (Node<E> trail = head, p = trail.next;
                     p != null;
                     trail = p, p = p.next) {
                    if (p == node) {
                        unlink(p, trail);
                        break;
                    }
                }
            } finally {
                fullyUnlock();
            }
        }
    }

```
<a id="org429ae36"></a>

# 其他方法介绍

LinkedBlockingQueue 还有一些不常用的方法：

1.  public boolean remove(Object o) ：从队列中删除给定的元素。如果成功返回 true，如果失败返回 false
2.  public boolean contains(Object o)：判断队列中是否包含指定的元素。如果包含则返回 true，否则返回 false
3.  public Object[] toArray()：将队列中的元素转成数组返回
4.  public void clear()：清空队列
5.  public int drainTo(Collection<? super E> c)：将队列中的元素添加到指定的集合中，并清空当前阻塞队列


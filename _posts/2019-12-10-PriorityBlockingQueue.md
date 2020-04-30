---
title: JUC队列源码介绍-PriorityBlockingQueue
tags: [Java, JUC, 源码, 多线程, 锁, 队列, PriorityBlockingQueue]
layout: post
author: wjf
---

<div id="table-of-contents">
<h2>目录</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#org66d906b">介绍</a></li>
<li><a href="#org4bbd91c">实现原理</a></li>
<li><a href="#orga4016ad">初始化方法</a></li>
<li><a href="#org6552a89">出队方法</a></li>
<li><a href="#org5bbbf96">入队方法</a></li>
<li><a href="#org345c4fc">二叉堆相关方法</a></li>
<li><a href="#org189b2fd">迭代器</a></li>
</ul>
</div>
</div>


<a id="org66d906b"></a>

# 介绍

PriorityBlockingQueue 是基于二叉堆（binary heap）实现的有序阻塞队列。队列容量上线是 Integer.MAX<sub>VALUE</sub> - 8，如果一直往队列中添加元素，并且来不及消费，可能会产生 OOM。PriorityBlockingQueue 的排序策略和 PriorityQueue 的排序策略一直，都是基于比较器（Comparator），如果没有比较机器则基于对象的 compareTo 方法。所以如果没比较器，那么必须要求队列元素实现 Comparator 方法。

PriorityBlockingQueue 继承自 Collection，也有迭代器相关功能。但是由于本身是基于二叉堆实现，遍历队列元素时无法按照排序的顺序遍历。如果需要按照顺序的顺序遍历，那么可以将 PriorityBlockingQueue 先转成数组（toArray()），在利用 Arrays.sort 将数组排序。

PriorityBlockingQueue 的继承关系如下：
![img](/assets/image/PriorityBlockingQueue.png)


<a id="org4bbd91c"></a>

# 实现原理

PriorityBlockingQueue 利用二叉堆的数据结构存储元素，能够保证队首或者最大（最大堆）或者最小（最小堆）来实现 PriorityBlockingQueue 每次出队元素最大或最小保证出队能够按照优先级。PriorityBlockingQueue 提供按照排序获取元素，但是并不要求队列内部的元素都按照这个顺序存放，所以二叉堆是一个非常好的选择。二叉堆既能够保证堆顶最大或者最小，又没有为了保证整体的顺序而带来的自旋等额外操作。

PriorityBlockingQueue 是基于数组实现的二叉堆，堆顶元素是数组下标 0。如果某元素的下标是 n，那么左子元素的下标是 2n+1，右子元素下标是 2（n+1)。如下是元素对应的二叉堆和数组的示意图：
![img](/assets/image/二叉堆.png)

二叉堆新元素入队：

1.  将新元素添加到数组上有效元素的后一个位置上。
2.  将新元素和父元素比较大小，如果有必要则调整父元素和新元素的位置。
3.  如果 2 做调整，那么继续比较直到和 0 下标的位置比较。

二叉堆出队：

1.  将数组最后一个位置上的元素替换掉 0 位置上的元素，并且最后一个位置置为 null
2.  数组有效元素减一
3.  和左右子节点比较大小，如果需要替换则替换元素位置
4.  如果 3 做了替换那么继续向下比较直到最后一层节点。

为了保证线程安全，PriorityBlockingQueue 改变队列数据（数组数据）的操作都是用可重入锁 ReentrantLock 保护。每次操作前 ReentrantLock 上锁，操作后释放锁。

和其他的阻塞队列一样，PriorityBlockingQueue 提供如果队列满了，入队线程阻塞；如果队列空了，出队线程阻塞的功能。PriorityBlockingQueue 利用 AbstractQueuedSynchronizer 提供的 ConnectionObject 的等待队列实现线程不满足队列要求而进入等待队列直到阻塞队列满足要求后通知线程继续执行。


<a id="orga4016ad"></a>

# 初始化方法

PriorityBlockingQueue 提供三个构造方法：

1.  无参构造函数：利用默认值构建数组大小并且不指定比较策略。需要元素本省能够支持比较（继承 Comparable）。
2.  指定初始构建数组大小和指定比较策略构方法
3.  根据指定集合构建阻塞队列的构造方法

具体代码如下：
```java
    /**
     * PriorityBlockingQueue 无参构造函数
     * 初始化一个数组长度为 11 的数组用于存储底层元素
     * 由于没有指定元素的比较策略，所以要求元素本身定义了比较方式。
     *
     */
    public PriorityBlockingQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }
    
    /**
     * 构造方法
     * 根据指定的容量大小构造一个该大小长度的数组作为存储元素的容器（二叉堆的容器）
     * 由于没有指定元素的比较策略，所以要求元素本身定义了比较方式。
     * @throws IllegalArgumentException if {@code initialCapacity} is less
     *         than 1
     */
    public PriorityBlockingQueue(int initialCapacity) {
        this(initialCapacity, null);
    }
    
    /**
     * 构造方法
     * 根据指定的容量大小构造一个该大小长度的数组作为存储元素的容器（二叉堆的容器）
     * 根据指定的排序策略构建二叉堆
     * @param initialCapacity 初始数组大小
     * @param comparator 排序策略
     * @throws IllegalArgumentException 如果 initialCapacity<1 则抛出 IllegalArgumentException 异常
     */
    public PriorityBlockingQueue(int initialCapacity,
                                 Comparator<? super E> comparator) {
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        this.comparator = comparator;
        this.queue = new Object[initialCapacity];
    }
    
    /**
     * 构造方法
     * 根据给定的集合构造一个 PriorityBlockingQueue
     *
     * @param  c 集合中的元素将会被添加到新创建到的 PriorityBlockingQueue 中
     * @throws ClassCastException 如果元素没有提供和其他元素的比较策略那么抛出 ClassCastException 异常
     * @throws NullPointerException 如果被添加的元素指向 null，那么抛出该异常
     */
    public PriorityBlockingQueue(Collection<? extends E> c) {
        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        // 是否需要根据二叉堆的排序规则重新排序
        boolean heapify = true;
        boolean screen = true;
        /*
         * 如果是排序的数组，那么将当前的 PriorityBlockingQueue 的策略置为排序数组的策略。
         * 由于已经是有序，不需要堆化。
         */
        if (c instanceof SortedSet<?>) {
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            heapify = false;
        }
        /*
         * 如果集合是 PriorityBlockingQueue，那么直接使用该 PriorityBlockingQueue 的比较策略
         * 由于是 PriorityBlockingQueue，不需要堆化
         */
        else if (c instanceof PriorityBlockingQueue<?>) {
            PriorityBlockingQueue<? extends E> pq =
                (PriorityBlockingQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            screen = false;
            if (pq.getClass() == PriorityBlockingQueue.class) // exact match
                heapify = false;
        }
        Object[] a = c.toArray();
        int n = a.length;
        // 利用数组拷贝，生成底层数组
        if (a.getClass() != Object[].class)
            a = Arrays.copyOf(a, n, Object[].class);
        //PriorityBlockingQueue 不支持入队 null，所以对生成后的数组做 Null 检查。
        if (screen && (n == 1 || this.comparator != null)) {
            for (int i = 0; i < n; ++i)
                if (a[i] == null)
                    throw new NullPointerException();
        }
        this.queue = a;
        this.size = n;
        // 如果需要堆化，则执行堆化。堆化是指按照二叉堆的排序规则重新对数组排序
        if (heapify)
            heapify();
    }
```

<a id="org6552a89"></a>

# 出队方法

PriorityBlockingQueue 提供了三种元素出队方法和一中获取队列第一个元素的方法。
poll(): 如果队列是空队列，则返回 null
take(): 如果队列是空队列，则线程阻塞直到非空信号通知到该阻塞线程
poll(long timeout, TimeUnit unit): 如果队列是控队列，那么线程阻塞。但是可以设置等待超时时间
peek(): 只返回队首元素，不改变队列。

```java
    public E poll() {
          final ReentrantLock lock = this.lock;
          lock.lock();
          try {
              // 利用 dequeue 方法将队首元素出队，并将队首元素返回给调用者
              return dequeue();
          } finally {
              lock.unlock();
          }
      }
    
      public E take() throws InterruptedException {
          final ReentrantLock lock = this.lock;
          lock.lockInterruptibly();
          E result;
          try {
              // 利用 dequeue 方法将队首元素出队，但是如果返回的是 null，说明本次出队时队列是空队列，线程进入等待队列。
              while ( (result = dequeue()) == null)
                  notEmpty.await();
          } finally {
              lock.unlock();
          }
          return result;
      }
    
      public E poll(long timeout, TimeUnit unit) throws InterruptedException {
          long nanos = unit.toNanos(timeout);
          final ReentrantLock lock = this.lock;
          lock.lockInterruptibly();
          E result;
          try {
              // 利用 dequeue 方法将队首元素出队，但是如果返回的是 null，说明本次出队时队列是空队列，线程进入等待队列，直到非空队列信号通知到该线程或者超过等待时间
              while ( (result = dequeue()) == null && nanos > 0)
                  nanos = notEmpty.awaitNanos(nanos);
          } finally {
              lock.unlock();
          }
          return result;
      }
    
      public E peek() {
          final ReentrantLock lock = this.lock;
          lock.lock();
          try {
              // 如果队列是空队列，则返回 Null，非空队列则返回队首元素
              return (size == 0) ? null : (E) queue[0];
          } finally {
              lock.unlock();
          }
      }
    
      /**
       * 私有的出队，为公有的出队方法提供出队操作
       */
      private E dequeue() {
          // 数组最后一个元素的下标位置
          int n = size - 1;
          // 空队列，返回 null
          if (n < 0)
              return null;
          else {
              Object[] array = queue;
              // 数组的一个元素时队首元素，该元素出队
              E result = (E) array[0];
              // 二叉堆出队：将数组最后一个元素移到下标 0 的位置，然后做下浮操作
              E x = (E) array[n];
              array[n] = null;
              Comparator<? super E> cmp = comparator;
              // 二叉堆下浮
              if (cmp == null)
                  siftDownComparable(0, x, array, n);
              else
                  siftDownUsingComparator(0, x, array, n, cmp);
              // 队列大小减一
              size = n;
              return result;
          }
      }
```

<a id="org5bbbf96"></a>

# 入队方法

PriorityBlockingQueue 提供了 4 个入队方法，分别是：

1.  add(E e)：如果添加成功返回 true，否则返回 false。方法内部直接调用 offer(E e)实现。
2.  offer(E e)：入队方法。如果入队成功则返回 true，否则返回 false。主要包括三部分内容：
    1.  判断是否超过数组容量限制，组底层数组扩展
    2.  元素入二叉堆
    3.  发送队列非空信号
3.  put(E e)：元素入队。直接调用 offer(E e)，只是不返回添加操作的是否成功
4.  offer(E e, long timeout, TimeUnit unit)：直接调用 offer(E e)实现。timeout 参数和 unit 参数直接过滤

```java
    /**
     * 元素入队。
     * 直接调用 offer(e)实现
     *
     * @param e 入队元素
     * @return 如果入队成功返回 true，否则返回 false
     * @throws ClassCastException 如果阻塞队列没有指定排序策略，那么需要依赖元素本身能够支持比较。如果不支持则抛出 ClassCastException
     * @throws NullPointerException 如果添加的元素时 Null 那么抛出 NullPointerException
     */
    public boolean add(E e) {
        return offer(e);
    }
    
    /**
     * 元素入队
     * 判断是否超过数组容量限制，组底层数组扩展
     * 元素入二叉堆
     * 发送队列非空信号
     *
     * @param e 入队元素
     * @return 如果入队成功返回 true，否则返回 false
     * @throws ClassCastException  如果阻塞队列没有指定排序策略，那么需要依赖元素本身能够支持比较。如果不支持则抛出 ClassCastException
     * @throws NullPointerException 如果添加的元素时 Null 那么抛出 NullPointerException
     */
    public boolean offer(E e) {
        // 如果队列元素时空对象那么抛出 NullPointerException 异常
        if (e == null)
            throw new NullPointerException();
        final ReentrantLock lock = this.lock;
        lock.lock();
        int n, cap;
        Object[] array;
        // 如果超过了容量上限，那么先将底层数组扩容
        while ((n = size) >= (cap = (array = queue).length))
            tryGrow(array, cap);
        try {
            Comparator<? super E> cmp = comparator;
            // 如果阻塞队列有比较策略，那么使用比较策略比较元素大小（确定在二叉堆中的位置），如果没有则使用元素本身的比较策略。
            // 将元素添加到 n 位置（数组后效元素位置+1），然后执行二叉堆上浮操作。
            if (cmp == null)
                siftUpComparable(n, e, array);
            else
                siftUpUsingComparator(n, e, array, cmp);
            // 阻塞队列容量加一
            size = n + 1;
            // 发送队列非空信号
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
        return true;
    }
    
    /**
     *  元素入队
     *
     * @param e 入队元素
     * @return 如果入队成功返回 true，否则返回 false
     * @throws ClassCastException  如果阻塞队列没有指定排序策略，那么需要依赖元素本身能够支持比较。如果不支持则抛出 ClassCastException
     * @throws NullPointerException 如果添加的元素时 Null 那么抛出 NullPointerException
     */
    public void put(E e) {
        offer(e); // never need to block
    }
    
    /**
     * 元素入队。
     * timeout 和 unit 参数不起任何作用。
     */
    public boolean offer(E e, long timeout, TimeUnit unit) {
        return offer(e); // never need to block
    }
```

<a id="org345c4fc"></a>

# 二叉堆相关方法

PriorityBlockingQueue 是通过基于数组的二叉堆数据结构实现的优先队列功能，主要的方法包括：

1.  扩大数组大小
2.  堆下浮：在堆结构中向下比较（子节点）找到合适的位置（如果存在的话）进行位置转换
3.  堆上浮：在堆结构中向上比较（父节点）找到合适的位置（如果存在的话）进行位置转换
4.  堆化：根据堆排序的逻辑对数组上的元素重新排序

```java
    /**
     * 扩大数组容量
     * 添加元素时，如果当前数组已经无法添加新的元素时，那么增加数组的大小。通常情况下是增加两倍，但是如果达到最大值，最大值是 Integer.MAX_VALUE - 8，那么抛出 OutOfMemoryError 异常。
     *
     * @param array 阻塞队列的底层数组
     * @param oldCap 阻塞队列的底层数组的长度
     */
    private void tryGrow(Object[] array, int oldCap) {
        // 在阻塞队列扩容期间，先释放锁，以便其他线程能出队和入队操作。性能上的考虑。
        lock.unlock();
        Object[] newArray = null;
        // 由于已经释放锁，为了保证扩容只有一个线程操作，所以使用 cas 修改 allocationSpinLock 控制
        if (allocationSpinLock == 0 &&
            UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                     0, 1)) {
            try {
                // 计算扩容后的大小
                int newCap = oldCap + ((oldCap < 64) ?
                                       (oldCap + 2) : // grow faster if small
                                       (oldCap >> 1));
                if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                    int minCap = oldCap + 1;
                    // minCap<0 说明已经超过 Interger.MAX_VALUE，所以变成负值
                    // 如果超过上限则抛出 OutOfMemoryError 异常
                    if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                        throw new OutOfMemoryError();
                    newCap = MAX_ARRAY_SIZE;
                }
                // 创建一个新长度的数组作为二叉堆的容器
                if (newCap > oldCap && queue == array)
                    newArray = new Object[newCap];
            } finally {
                allocationSpinLock = 0;
            }
        }
        // 如果 newArray == null 说明已经有其他的线程做扩容了，那么将 CPU 的权限让渡出去。
        if (newArray == null)
            Thread.yield();
        // 重新上锁
        lock.lock();
        // 如果 newArray != null && queue == array 说明扩容成功，那么将原来数组的数据拷贝到新创建的数组中，并将新数组作为阻塞队列的二叉树容器
        // 如果是其他线程扩容，那么将在调用该方法的 offer 方法中 while 判断未扩容成功（n == cap），继续执行 tryGrow 方法做扩容。
        if (newArray != null && queue == array) {
            queue = newArray;
            System.arraycopy(array, 0, newArray, 0, oldCap);
        }
    }
    
    /**
     * 二叉堆上浮
     *
     * @param k 添加元素的数组下标位置
     * @param x 入队元素
     * @param array 存储元素的数组
     */
    private static <T> void siftUpComparable(int k, T x, Object[] array) {
        // 获取元素的比较策略用于比较大小
        Comparable<? super T> key = (Comparable<? super T>) x;
        // 递归做上浮操作，寻找二叉堆添加元素位置的过程。
        while (k > 0) {
            // 获取父节点在数组上的位置
            int parent = (k - 1) >>> 1;
            // 获取父节点的元素
            Object e = array[parent];
            // 比较大小。如果当前节点已经 大于或者等于父节点，那么不需要继续上浮操作，跳出循环。
            if (key.compareTo((T) e) >= 0)
                break;
            // 小于父节点，将父节点的元素赋值给比较的位置（每次递归操作比较位置会发生变化）
            array[k] = e;
            // 更改比较位置，调整成父节点。
            k = parent;
        }
        // 将元素赋值到定位到的二叉树位置中
        array[k] = key;
    }
    
    /**
     * 二叉堆上浮。
     * 和上一个上浮操作逻辑唯一不同的地方是利用指定的比较策略比较。
     */
    private static <T> void siftUpUsingComparator(int k, T x, Object[] array,
                                       Comparator<? super T> cmp) {
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = array[parent];
            if (cmp.compare(x, (T) e) >= 0)
                break;
            array[k] = e;
            k = parent;
        }
        array[k] = x;
    }
    
    /**
     * 二叉堆下沉操作
     *
     * @param k 下沉开始的位置，一般是 0（二叉堆堆顶位置）
     * @param x 比较元素，一般是 0 指向的元素（二叉堆堆顶元素）
     * @param array 阻塞队列数组
     * @param n 队列长度
     */
    private static <T> void siftDownComparable(int k, T x, Object[] array,
                                               int n) {
        // 队列中有元素才需要做下层操作，负责直接添加到 0 位置就可
        if (n > 0) {
            Comparable<? super T> key = (Comparable<? super T>)x;
            // 递归退出的临界值。
            int half = n >>> 1;
            // 递归查找下沉位置
            while (k < half) {
                // 左子节点的位置
                int child = (k << 1) + 1;
                Object c = array[child];
                // 右子节点的位置
                int right = child + 1;
                // 比较左右节点的大小，如果左节点元素大于右节点元素，那么将需要比较位置调成右子节点的位置，并将待比较的元素调整成右子节点指向的元素
                if (right < n &&
                    ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                    c = array[child = right];
                // 如果待添加的元素已经小于或者等于左子节点，那么跳出循环
                if (key.compareTo((T) c) <= 0)
                    break;
                // 将待比较的元素赋值给比较位置的下标，并将比较位置调整成叶子节点的位置，继续递归查找合适的位置。
                array[k] = c;
                k = child;
            }
            // 已经确认合适的位置，将待添加的元素复制到该位置上
            array[k] = key;
        }
    }
    
    /**
     * 二叉堆下沉操作
     * 和上一个下沉操作唯一不同的地方在于使用指定的比较策略。
     */
    private static <T> void siftDownUsingComparator(int k, T x, Object[] array,
                                                    int n,
                                                    Comparator<? super T> cmp) {
        if (n > 0) {
            int half = n >>> 1;
            while (k < half) {
                int child = (k << 1) + 1;
                Object c = array[child];
                int right = child + 1;
                if (right < n && cmp.compare((T) c, (T) array[right]) > 0)
                    c = array[child = right];
                if (cmp.compare(x, (T) c) <= 0)
                    break;
                array[k] = c;
                k = child;
            }
            array[k] = x;
        }
    }
    
    /**
     * 堆化
     * 将阻塞队列数组上的元素按照二叉堆排序规则重新排序
     */
    private void heapify() {
        Object[] array = queue;
        // 阻塞队列长度
        int n = size;
        // 需要比较的次数。
        // 由于下沉操作本身比较会替换位置，所以不需要所有的元素都进行排序，只需要后半部分下沉或者前半部分上浮即可
        int half = (n >>> 1) - 1;
        Comparator<? super E> cmp = comparator;
        // 利用二叉堆下沉操作比较。
        // 下沉操作会比较左右子节点的大小并重新排序，比上浮操作会更加接近左叶子节点比右叶子节点元素小。
        if (cmp == null) {
            for (int i = half; i >= 0; i--)
                siftDownComparable(i, (E) array[i], array, n);
        }
        else {
            for (int i = half; i >= 0; i--)
                siftDownUsingComparator(i, (E) array[i], array, n, cmp);
        }
    }
```

<a id="org189b2fd"></a>

# 迭代器

PriorityBlockingQueue 继承自 Collection，所以也支持迭代器方法。但是由于二叉堆无法保证除了堆顶元素最小之外的其他元素的排序，所以遍历元素时，除堆顶外的其他元素无法保证按照比较策略的顺序出来。

```java
    /**
     * 返回迭代器
     * PriorityBlockingQueue 利用内部类 Itr 实现迭代器相关功能。该方法实际上是返回 Itr 的一个实例。
     * 由于初始化迭代器时，利用阻塞队列 toArray()创建了一个新数组，所以如果是在遍历过程中，阻塞队列发生变化，并不能正确的反映到迭代器上
     * @return an iterator over the elements in this queue
     */
    public Iterator<E> iterator() {
        return new Itr(toArray());
    }
    
    /**
     * 迭代器的实现类
     */
    final class Itr implements Iterator<E> {
        // 迭代器的数组。
        final Object[] array; // Array of all elements
        // 当前遍历到的数组下边
        int cursor;
        // 上一次遍历到的数组下标
        int lastRet;
    
        Itr(Object[] array) {
            lastRet = -1;
            this.array = array;
        }
    
        /**
         * 如果 cursor 没有到数组数组最后一个位置，返回 true，否则返回 false
         */
        public boolean hasNext() {
            return cursor < array.length;
        }
        /**
         * 返回下一个元素，并将游标后移
         */
        public E next() {
            // 如果已经到达最后一个了元素了，该方法抛出 NoSuchElementException
            if (cursor >= array.length)
                throw new NoSuchElementException();
            lastRet = cursor;
            // 返回 cursor 游标上的元素，并将 cursor 自增一
            return (E)array[cursor++];
        }
    
        /**
         * 删除迭代器当前遍历到的元素
         * 利用阻塞队列 removeEQ 方法删除阻塞队列中的元素
         * 将 lastRet 置为-1，表示当前遍历的元素已经被删除。
         */
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            removeEQ(array[lastRet]);
            lastRet = -1;
        }
    }

```
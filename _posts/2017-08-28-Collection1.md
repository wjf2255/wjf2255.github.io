---
title: Java集合简单介绍
tags: Java, Collection
layout: post
author: wjf
---

<div id="table-of-contents">

<h2>Java集合简单介绍</h2>

<div id="text-table-of-contents">

<ul>

<li>1. 集合(第一部分)

<ul>

<li>1.1. 集合（Collection）介绍</li>

<li>1.2. List

<ul>

<li>1.2.1. ArrayList</li>

<li>1.2.2. LinkedList</li>

<li>1.2.3. Vector</li>

<li>1.2.4. Stack</li>

<li>1.2.5. ArrayList 和 LinkedList 的应用场景</li>

</ul>

</li>

<li>1.3. Set

<ul>

<li>1.3.1. HashSet</li>

<li>1.3.2. TreeSet</li>

</ul>

</li>

<li>1.4. Map

<ul>

<li>1.4.1. HashMap</li>

<li>1.4.2. TreeMap</li>

<li>1.4.3. HashTable</li>

</ul>

</li>

</ul>

</li>

</ul>

</div>

</div>

<a id="org076f2b7"></a>

集合(第一部分)

<a id="org3fd26a1"></a>

集合（Collection）介绍

从 JDK1.2 开始提供集合工具。集合是用来存储一组相同类型的对象。这些对象在集合中成为元素（element）。有些集合支持相同的元素，有些集合则不支持。Collection 接口是所有集合的最上层接口，它定义了集合基础的一些方法:

    int size(); //返回集合元素的个数。如果集合中元素个数超过 Integer.MAX_VALUE（2e31-1），则返回该值
    boolean isEmpty() //是否是空集合
    boolean contains(Object o) //是否包含某个元素 比较是否一致，调用的是 equals()
    Iterator<E> iterator(); //返回一个迭代器，用于遍历当前集合。
        JDK1.8 中 Iterable 接口增加默认方法
        default void forEach(Consumer<? super T> action) {
            Objects.requireNonNull(action);
            for (T t : this) {
                action.accept(t);
            }
        }
    Object[] toArray(); //转成数组
    <T> T[] toArray(T[] a); //利用泛型转成数组
    boolean add(E e); //添加元素
    boolean remove(Object o); //删除元素，利用 equals 比较
    boolean containsAll(Collection<?> c); //是否包含给定集合的所有元素。利用 equals 比较
    boolean addAll(Collection<? extends E> c); //添加给定集合中的所有元素
    boolean removeAll(Collection<?> c);    //删除给定集合中的同时存在于当前集合的元素。利用 equals 比较
    default boolean removeIf(Predicate<? super E> filter) { //从 JDK1.8 开始 添加的一个默认方法（Default Mehtod）。用于根据给定的过滤条件，删除集合中的元素
            Objects.requireNonNull(filter);
            boolean removed = false;
            final Iterator<E> each = iterator();
            while (each.hasNext()) {
                if (filter.test(each.next())) {
                    each.remove();
                    removed = true;
                }
            }
            return removed;
        }
    boolean retainAll(Collection<?> c); //保留只存在于给定集合中的元素。利用 equals 比较
    void clear(); //清空集合
    boolean equals(Object o); //比较集合和对象是否相同
    int hashCode(); //返回当前集合的 hash 编码
    default Spliterator<E> spliterator() { //返回一个分隔器，用户分隔当前集合 JDK1.8
        return Spliterators.spliterator(this, 0);
    }
    default Stream<E> stream() { //提供一些方便的集合操作方法 JDK1.8
        return StreamSupport.stream(spliterator(), false);
    }
    default Stream<E> parallelStream() { 提供一个可以并行执行的 stream 操作
        return StreamSupport.stream(spliterator(), true);
    }

Map 也被认为是集合。他的定义的接口和 Collection 类似，只是 Collection 存储的是元素，Map 存储的是 k,value。Map 内部包含一个 Set 接口。

集合

Collection

├List

│├LinkedList

│├ArrayList

│└Vector

│　└Stack

└Set

Map

├Hashtable

├HashMap

└WeakHashMap

<a id="org2b688e6"></a>

List

List 是一个有顺序的集合。我们可以控制将元素添加到集合中的具体的位置，也可以通过一个索引获取位于该位置的元素。List 支持存储相同的元素。

    default void replaceAll(UnaryOperator<E> operator) { //用于替换集合中的符合规定的元素 JDK1.8
        Objects.requireNonNull(operator);
        final ListIterator<E> li = this.listIterator();
        while (li.hasNext()) {
            li.set(operator.apply(li.next()));
        }
    }
    
    default void sort(Comparator<? super E> c) { //支持排序
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }
    default Spliterator<E> spliterator() { //有顺序的分隔器
        return Spliterators.spliterator(this, Spliterator.ORDERED);
    }

<a id="org3396064"></a>

ArrayList

ArrayList 使用一个 transient Object[] elementData 存储数据。

ArrayList 的提供了两个构造函数。无参数的构造函数。默认初始化一个长度为 10 的 Object[]数组。一个带参数的构造函数，用于确认数据大小。

    /**
     * Constructs an empty list with an initial capacity of ten.
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
    
    /**
     * Constructs an empty list with the specified initial capacity.
     *
     * @param  initialCapacity  the initial capacity of the list
     * @throws IllegalArgumentException if the specified initial capacity
     *         is negative
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

ArrayList 的添加。添加方法会比较 size(集合的长度)+1 和数组的长度的大小。如果 size+1>数组长度，需要重新开辟新的数组用来存储数据。

    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    
    /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity); //如果超过 Integer.MAX_VALUE 则职位 Integer.MAX_VALUE
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

ArrayList 的删除。

    public E remove(int index) {
        rangeCheck(index);
    
        modCount++;
        E oldValue = elementData(index);
    
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    
        return oldValue;
    }

拷贝两段数组在拼凑到一起。

<a id="org3e68ca2"></a>

LinkedList

LinkedList 是一个双向链表。LinkedList 不是线程安全的，如果在多线程环境下使用，需要在调用 ListkedList 的方法上注明是 synchronized。如果无法保证上诉情况，可以使用 List list = Collections.synchronizedList(new LinkedList(&#x2026;))这种方式确保线程安全。

    /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient int size = 0;
    transient Node<E> first;
    
    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;
    
    // LinkedList 节点
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
    
        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }

<a id="org7bda87f"></a>

Vector

Vector 和 ArrayList 的功能非常相似，只是在众多的方法上，使用 synchronized 关键词。在多线程环境下，调用这些方法时，可以确保是线程安全。

<a id="orgac72786"></a>

Stack

Stack 继承 Vector，并提供栈的特性。

    public E push(E item) {
        addElement(item);
    
        return item;
    }
    
    public synchronized E pop() {
        E       obj;
        int     len = size();
    
        obj = peek();
        removeElementAt(len - 1);
    
        return obj;
    }
    
    public synchronized E peek() {
        int     len = size();
    
        if (len == 0)
            throw new EmptyStackException();
        return elementAt(len - 1);
    }

<a id="org587bd58"></a>

ArrayList 和 LinkedList 的应用场景

数组读取数据比较快，添加和删除比较慢

双向链表读取数据比较慢，添加和删除比较快

<a id="org058ebd3"></a>

Set

Set 存储不重复的一组元素，即不会存在两个元素 e1、e2，且 e1.equals(e2)，并且 Set 只允许存在一个指向 null 的元素。Set 通过在添加元素时，比较是否存在相等的元素来保证存储不一致元素的特性。如果存储的是可变的元素，一旦元素发生变更，Set 是不知道的。

<a id="org55244ff"></a>

HashSet

HashSet 通过 HashMap 或者 LinkedHashMap 实现了 Set 接口的所有方法。HashSet 无法保证存储在 Set 中的元素的顺序。

<a id="org2f687ea"></a>

TreeSet

TreeSet 通过维护一个 TreeMap 实现了 Set 接口的所有方法。TreeSet 元素是按照元素的的升序存储，访问和遍历比较快。往 TreeSet 添加的元素需要重写 eaquals 和实现 Comparable 接口，以便 TreeSet 能够判断元素之间排序先后的问题。

<a id="orga412bc0"></a>

Map

一个存储键值对的对象。Map 不允许重复的 key，并且每个 key 对应一个 value。Map 接口定义了返回 Set<key>，Set<Entry>，Collection<value>。

<a id="org2c50125"></a>

HashMap

HashMap 内部使用桶+链表/红黑树来存储。

链表的节点:

    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
    
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    
        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }
    
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }
    
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
    
        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }

红黑树节点

    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
    }

HashMap 添加节点

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //如果 table 为空或者长度为 0，则 resize()
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //计算 hashcode 并根据 hashcode 比较查找桶。
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            //第一个 node 的 hash 值即为要加入元素的 hash
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                //tree 的添加元素操作
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //不是 TreeNode,即为链表,遍历链表
    
                for (int binCount = 0; ; ++binCount) {
                    //到达链表的尾端也没有找到 key 值相同的节点，则生成一个新的 Node,并且判断链表的节点个数是不是到达转换成红黑树的上界达到，则转换成红黑树
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // 存在 key 值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }



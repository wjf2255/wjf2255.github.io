---
title: Java集合简单介绍2
tags: Java, Collection
layout: post
author: wjf
---

<div id="table-of-contents">

<h2>Java集合 介绍二</h2>

<div id="text-table-of-contents">

<ul>

<li>1. 集合 第二部分

</li>

<li>1.2. 实际问题

<ul>

<li>1.2.1. 问题描述</li>

<li>1.2.2. 解决方法</li>

</ul>

</li>

<li>1.3. Java 中的HashTable

<ul>

<li>1.3.1. HashTable 的属性</li>

<li>1.3.2. HashTable 的构造函数</li>

<li>1.3.3. HashTable 主要方法</li>

</ul>

</li>

<li>1.4. HashMap

<ul>

<li>1.4.1. 重要属性</li>

<li>1.4.2. 构造函数</li>

<li>1.4.3. 主要方法</li>

<li>1.4.4. 红黑树</li>

</ul>

</li>

</ul>

</div>

</div>

<a id="orgd3953cf"></a>

集合 第二部分

<a id="org082ea44"></a>

上次分享回顾

<a id="orgfd5e7ef"></a>

List

<a id="org16d60ff"></a>

简单介绍 hashcode

<a id="orgf89e492"></a>

数组和链表

<a id="orgbcb72f7"></a>

实际问题

<a id="org7194b97"></a>

问题描述

假设我们有一百万个在招职位和一千万个 C 端账号。我们系统会记录 C 端账号浏览职位的信息，并且相同职位只记录一次。我们使用 varchar(255)记录这条信息。我们如何在使用内存不超过 1G 的情况下，统计最热门（拥有最多 C 端账号查看的职位）的十个职位。

<a id="org5613011"></a>

解决方法

最直接的办法：对每条用户访问职位的数据进行遍历并排序。对已一次性内存无法装载下的，可以使用外排序的方法进行排序。并对每个排序结果统计出现次数。并将结果放入到最大堆中。

<a id="orgbca6d1f"></a>

Java 中的HashTable

<a id="orge0c3ca6"></a>

HashTable 的属性

    /**
     * The hash table data.
     */
    private transient Entry<?,?>[] table;
    
    /**
     * The total number of entries in the hash table.
     */
    private transient int count;
    
    /**
     * The table is rehashed when its size exceeds this threshold.  (The
     * value of this field is (int)(capacity * loadFactor).)
     *
     * @serial
     */
    private int threshold;
    
    /**
     * The load factor for the hashtable.
     *
     * @serial
     */
    private float loadFactor;

<a id="orge1ae570"></a>

HashTable 的构造函数

    /**
     * Constructs a new, empty hashtable with the specified initial
     * capacity and the specified load factor.
     *
     * @param      initialCapacity   the initial capacity of the hashtable.
     * @param      loadFactor        the load factor of the hashtable.
     * @exception  IllegalArgumentException  if the initial capacity is less
     *             than zero, or if the load factor is nonpositive.
     */
    public Hashtable(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load: "+loadFactor);
    
        if (initialCapacity==0)
            initialCapacity = 1;
        this.loadFactor = loadFactor;
        table = new Entry<?,?>[initialCapacity];
        threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    }
    
    /**
     * Constructs a new, empty hashtable with the specified initial capacity
     * and default load factor (0.75).
     *
     * @param     initialCapacity   the initial capacity of the hashtable.
     * @exception IllegalArgumentException if the initial capacity is less
     *              than zero.
     */
    public Hashtable(int initialCapacity) {
        this(initialCapacity, 0.75f);
    }
    
    /**
     * Constructs a new, empty hashtable with a default initial capacity (11)
     * and load factor (0.75).
     */
    public Hashtable() {
        this(11, 0.75f);
    }
    
    /**
     * Constructs a new hashtable with the same mappings as the given
     * Map.  The hashtable is created with an initial capacity sufficient to
     * hold the mappings in the given Map and a default load factor (0.75).
     *
     * @param t the map whose mappings are to be placed in this map.
     * @throws NullPointerException if the specified map is null.
     * @since   1.2
     */
    public Hashtable(Map<? extends K, ? extends V> t) {
        this(Math.max(2*t.size(), 11), 0.75f);
        putAll(t);
    }

<a id="orga6b6fa8"></a>

HashTable 主要方法

        /**
         * Maps the specified <code>key</code> to the specified
         * <code>value</code> in this hashtable. Neither the key nor the
         * value can be <code>null</code>. <p>
         *
         * The value can be retrieved by calling the <code>get</code> method
         * with a key that is equal to the original key.
         *
         * @param      key     the hashtable key
         * @param      value   the value
         * @return     the previous value of the specified key in this hashtable,
         *             or <code>null</code> if it did not have one
         * @exception  NullPointerException  if the key or value is
         *               <code>null</code>
         * @see     Object#equals(Object)
         * @see     #get(Object)
         */
        public synchronized V put(K key, V value) {
            // Make sure the value is not null
            if (value == null) {
                throw new NullPointerException();
            }
    
            // Makes sure the key is not already in the hashtable.
            Entry<?,?> tab[] = table;
            int hash = key.hashCode();
            int index = (hash & 0x7FFFFFFF) % tab.length;
            @SuppressWarnings("unchecked")
            Entry<K,V> entry = (Entry<K,V>)tab[index];
            for(; entry != null ; entry = entry.next) {
                if ((entry.hash == hash) && entry.key.equals(key)) {
                    V old = entry.value;
                    entry.value = value;
                    return old;
                }
            }
    
            addEntry(hash, key, value, index);
            return null;
        }

        private void addEntry(int hash, K key, V value, int index) {
            modCount++;
    
            Entry<?,?> tab[] = table;
            if (count >= threshold) {
                // Rehash the table if the threshold is exceeded
                rehash();
    
                tab = table;
                hash = key.hashCode();
                index = (hash & 0x7FFFFFFF) % tab.length;
            }
    
            // Creates the new entry.
            @SuppressWarnings("unchecked")
            Entry<K,V> e = (Entry<K,V>) tab[index];
            tab[index] = new Entry<>(hash, key, value, e);
            count++;
        }
    
        /**
         * Removes the key (and its corresponding value) from this
         * hashtable. This method does nothing if the key is not in the hashtable.
         *
         * @param   key   the key that needs to be removed
         * @return  the value to which the key had been mapped in this hashtable,
         *          or <code>null</code> if the key did not have a mapping
         * @throws  NullPointerException  if the key is <code>null</code>
         */
        public synchronized V remove(Object key) {
            Entry<?,?> tab[] = table;
            int hash = key.hashCode();
            int index = (hash & 0x7FFFFFFF) % tab.length;
            @SuppressWarnings("unchecked")
            Entry<K,V> e = (Entry<K,V>)tab[index];
            for(Entry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
                if ((e.hash == hash) && e.key.equals(key)) {
                    modCount++;
                    if (prev != null) {
                        prev.next = e.next;
                    } else {
                        tab[index] = e.next;
                    }
                    count--;
    `                V oldValue = e.value;
                    e.value = null;
                    return oldValue;
                }
            }
            return null;
        }

<a id="orga988492"></a>

HashMap

HashMap 允许空的 key(key == null)，空的值(value==null)。HashMap 不是线程安全的，在多线程环境下使用需要做一下包装 Map m = Collections.synchronizedMap(new HashMap(&#x2026;));

<a id="orgf223996"></a>

重要属性

    //默认初始容量
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    //最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;
    //默认装载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    //链表转树的临界值
    static final int TREEIFY_THRESHOLD = 8;
    //树转链表的临界值
    static final int UNTREEIFY_THRESHOLD = 6;
    //哈希表
    transient Entry<K,V>[] table;
    //键值对的数量
    transient int size;
    //扩容的阈值
    int threshold;
    //哈希数组的装载因子
    final float loadFactor;

<a id="org36fbcae"></a>

构造函数

    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and load factor.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

<a id="orgb8a9012"></a>

主要方法

    /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
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

<a id="orge483720"></a>

红黑树 （转载 http://www.cnblogs.com/skywang12345/p/3624343.html）

红黑树是特殊的二叉查找树，意味着它满足二叉查找树的特征：任意一个节点所包含的键值，大于等于左孩子的键值，小于等于右孩子的键值。

除了具备该特性之外，红黑树还包括许多额外的信息。

红黑树的每个节点上都有存储位表示节点的颜色，颜色是红(Red)或黑(Black)。

红黑树的特性:

(1) 每个节点或者是黑色，或者是红色。

(2) 根节点是黑色。

(3) 每个叶子节点是黑色。 [注意：这里叶子节点，是指为空的叶子节点！]

(4) 如果一个节点是红色的，则它的子节点必须是黑色的。

(5) 从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。

关于它的特性，需要注意的是：

第一，特性(3)中的叶子节点，是只为空(NIL 或 null)的节点。

第二，特性(5)，确保没有一条路径会比其他路径长出俩倍。因而，红黑树是相对是接近平衡的二叉树。

1. 左旋
       /*
        * 对红黑树的节点(x)进行左旋转
        *
        * 左旋示意图(对节点 x 进行左旋)：
        *      px                              px
        *     /                               /
        *    x                               y
        *   /  \      --(左旋)-.           / \                #
        *  lx   y                          x  ry
        *     /   \                       /  \
        *    ly   ry                     lx  ly
        *
        *
        */
       private void leftRotate(RBTNode<T> x) {
           // 设置 x 的右孩子为 y
           RBTNode<T> y = x.right;
       
           // 将 “y 的左孩子” 设为 “x 的右孩子”；
           // 如果 y 的左孩子非空，将 “x” 设为 “y 的左孩子的父亲”
           x.right = y.left;
           if (y.left != null)
               y.left.parent = x;
       
           // 将 “x 的父亲” 设为 “y 的父亲”
           y.parent = x.parent;
       
           if (x.parent == null) {
               this.mRoot = y;            // 如果 “x 的父亲” 是空节点，则将 y 设为根节点
           } else {
               if (x.parent.left == x)
                   x.parent.left = y;    // 如果 x 是它父节点的左孩子，则将 y 设为“x 的父节点的左孩子”
               else
                   x.parent.right = y;    // 如果 x 是它父节点的左孩子，则将 y 设为“x 的父节点的左孩子”
           }
       
           // 将 “x” 设为 “y 的左孩子”
           y.left = x;
           // 将 “x 的父节点” 设为 “y”
           x.parent = y;
       }
2. 右旋
       /*
        * 对红黑树的节点(y)进行右旋转
        *
        * 右旋示意图(对节点 y 进行左旋)：
        *            py                               py
        *           /                                /
        *          y                                x
        *         /  \      --(右旋)-.            /  \                     #
        *        x   ry                           lx   y
        *       / \                                   / \                   #
        *      lx  rx                                rx  ry
        *
        */
       private void rightRotate(RBTNode<T> y) {
           // 设置 x 是当前节点的左孩子。
           RBTNode<T> x = y.left;
       
           // 将 “x 的右孩子” 设为 “y 的左孩子”；
           // 如果"x 的右孩子"不为空的话，将 “y” 设为 “x 的右孩子的父亲”
           y.left = x.right;
           if (x.right != null)
               x.right.parent = y;
       
           // 将 “y 的父亲” 设为 “x 的父亲”
           x.parent = y.parent;
       
           if (y.parent == null) {
               this.mRoot = x;            // 如果 “y 的父亲” 是空节点，则将 x 设为根节点
           } else {
               if (y == y.parent.right)
                   y.parent.right = x;    // 如果 y 是它父节点的右孩子，则将 x 设为“y 的父节点的右孩子”
               else
                   y.parent.left = x;    // (y 是它父节点的左孩子) 将 x 设为“x 的父节点的左孩子”
           }
       
           // 将 “y” 设为 “x 的右孩子”
           x.right = y;
       
           // 将 “y 的父节点” 设为 “x”
           y.parent = x;
       }
3. 添加节点
   1. 将红黑树当做一棵二叉查找树，将节点加入其中。首选保证添加节点之后依然是一颗二叉查找树。
   2. 插入的节点为红色节点
   3. 旋转和着色，使得重新变成一棵红黑树
      /*
      - 红黑树插入修正函数
        *
      - 在向红黑树中插入节点之后(失去平衡)，再调用该函数；
      - 目的是将它重新塑造成一颗红黑树。
        *
      - 参数说明：
      - node 插入的结点        // 对应《算法导论》中的 z
        */
        private void insertFixUp(RBTNode<T> node) {
        RBTNode<T> parent, gparent;
        // 若“父节点存在，并且父节点的颜色是红色”
        while (((parent = parentOf(node))!=null) && isRed(parent)) {
            gparent = parentOf(parent);
            //若“父节点”是“祖父节点的左孩子”
            if (parent == gparent.left) {
                // Case 1 条件：叔叔节点是红色
                RBTNode<T> uncle = gparent.right;
                if ((uncle!=null) && isRed(uncle)) {
                    setBlack(uncle);
                    setBlack(parent);
                    setRed(gparent);
                    node = gparent;
                    continue;
                }
            
                // Case 2 条件：叔叔是黑色，且当前节点是右孩子
                if (parent.right == node) {
                    RBTNode<T> tmp;
                    leftRotate(parent);
                    tmp = parent;
                    parent = node;
                    node = tmp;
                }
            
                // Case 3 条件：叔叔是黑色，且当前节点是左孩子。
                setBlack(parent);
                setRed(gparent);
                rightRotate(gparent);
            } else {    //若“z 的父节点”是“z 的祖父节点的右孩子”
                // Case 1 条件：叔叔节点是红色
                RBTNode<T> uncle = gparent.left;
                if ((uncle!=null) && isRed(uncle)) {
                    setBlack(uncle);
                    setBlack(parent);
                    setRed(gparent);
                    node = gparent;
                    continue;
                }
            
                // Case 2 条件：叔叔是黑色，且当前节点是左孩子
                if (parent.left == node) {
                    RBTNode<T> tmp;
                    rightRotate(parent);
                    tmp = parent;
                    parent = node;
                    node = tmp;
                }
            
                // Case 3 条件：叔叔是黑色，且当前节点是右孩子。
                setBlack(parent);
                setRed(gparent);
                leftRotate(gparent);
            }
        }
        // 将根节点设为黑色
        setBlack(this.mRoot);
        }
4. 删除操作
   1. 删除节点
   2. 通过旋转和重新着色使得它成为一个红黑树
      /*
      - 红黑树删除修正函数
        *
      - 在从红黑树中删除插入节点之后(红黑树失去平衡)，再调用该函数；
      - 目的是将它重新塑造成一颗红黑树。
        *
      - 参数说明：
      - node 待修正的节点
        */
        private void removeFixUp(RBTNode<T> node, RBTNode<T> parent) {
        RBTNode<T> other;
        while ((node==null || isBlack(node)) && (node != this.mRoot)) {
            if (parent.left == node) {
                other = parent.right;
                if (isRed(other)) {
                    // Case 1: x 的兄弟 w 是红色的
                    setBlack(other);
                    setRed(parent);
                    leftRotate(parent);
                    other = parent.right;
                }
            
                if ((other.left==null || isBlack(other.left)) &&
                    (other.right==null || isBlack(other.right))) {
                    // Case 2: x 的兄弟 w 是黑色，且 w 的俩个孩子也都是黑色的
                    setRed(other);
                    node = parent;
                    parent = parentOf(node);
                } else {
            
                    if (other.right==null || isBlack(other.right)) {
                        // Case 3: x 的兄弟 w 是黑色的，并且 w 的左孩子是红色，右孩子为黑色。
                        setBlack(other.left);
                        setRed(other);
                        rightRotate(other);
                        other = parent.right;
                    }
                    // Case 4: x 的兄弟 w 是黑色的；并且 w 的右孩子是红色的，左孩子任意颜色。
                    setColor(other, colorOf(parent));
                    setBlack(parent);
                    setBlack(other.right);
                    leftRotate(parent);
                    node = this.mRoot;
                    break;
                }
            } else {
            
                other = parent.left;
                if (isRed(other)) {
                    // Case 1: x 的兄弟 w 是红色的
                    setBlack(other);
                    setRed(parent);
                    rightRotate(parent);
                    other = parent.left;
                }
            
                if ((other.left==null || isBlack(other.left)) &&
                    (other.right==null || isBlack(other.right))) {
                    // Case 2: x 的兄弟 w 是黑色，且 w 的俩个孩子也都是黑色的
                    setRed(other);
                    node = parent;
                    parent = parentOf(node);
                } else {
            
                    if (other.left==null || isBlack(other.left)) {
                        // Case 3: x 的兄弟 w 是黑色的，并且 w 的左孩子是红色，右孩子为黑色。
                        setBlack(other.right);
                        setRed(other);
                        leftRotate(other);
                        other = parent.left;
                    }
            
                    // Case 4: x 的兄弟 w 是黑色的；并且 w 的右孩子是红色的，左孩子任意颜色。
                    setColor(other, colorOf(parent));
                    setBlack(parent);
                    setBlack(other.left);
                    rightRotate(parent);
                    node = this.mRoot;
                    break;
                }
            }
        }
        if (node!=null)
            setBlack(node);
        }
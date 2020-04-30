---
title: AtomicInteger源码介绍
tags: [Java, JUC, 源码, 多线程, AtomicInteger]
layout: post
author: wjf
---

<div id="table-of-contents">
<h2>目录</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#org20cf3ed">介绍</a></li>
<li><a href="#org0347ec7">原理</a></li>
<li><a href="#orgd116cd8">方法列表</a>
<ul>
<li><a href="#orgc57333a">Number 方法实现</a></li>
<li><a href="#org7cb01e6">AtomicInteger 操作整形数据的方法列表</a></li>
</ul>
</li>
</ul>
</div>
</div>


<a id="org20cf3ed"></a>

# 介绍

AtomicInteger 是 JDK1.5 之后引入的原子更新整形数据的工具。同期引入的还有另外 11 个原子操作工具，这个工具都能提供简单高效并且多线程安全的原子操作。


<a id="org0347ec7"></a>

# 原理

AtomicInteger 能够保证线程安全的原因是 AtomicInteger 的整形属性 private volatile int value 使用 volatile 关键词修饰，能够保证该属性对各个线程都是最新的值；其次 value 的修改是利用 unsafe 的方法修改 value 的值，而 unsafe 的方式修改能够保证原子性。

原子性操作是指不会被线程调度机制打断的操作（来自百度百科），即线程切换还么在原子操作之前，要么在原子操作之后，如果操作失败，那么一起回滚。在多核环境下，原子操作需要操作系统控制一个 CPU 在操作某一个内存时，其他 CPU 不会操作这个内存空间。

volatile 修饰的属性具备：

-   可见性：对于任何线程读取 volatile 变量，该线程总能看到其他线程对该变量的最后的写入值。任何线程的写入都直接推到主内存中，任何线程读取都会拉取主内存的值（为了非 volatile 修饰的变量，更新和读取都针对的是线程内存中的拷贝，不会立即推到主内存中，此时其他线程无法知晓该工作线程对该共享数据的修改）。
-   原子性：volatile 修饰的变量在做赋值操作时具备原子性（i++这种不具备原子性）。比如 volatile 修改的 long 和 double 类型不会被拆分成高低 32 位交由多个 CPU 处理。

CAS 操作的原子性:
以 compareAndSwapInt 为例，代码实现如下：

```c
     inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
      // alternative for InterlockedCompareExchange
      int mp = os::is_MP();
      __asm {
        mov edx, dest
        mov ecx, exchange_value
        mov eax, compare_value
        LOCK_IF_MP(mp)
        cmpxchg dword ptr [edx], ecx
      }
    }
```

来自《Java 并发编程的艺术》
处理器会保证基本的内存操作的原子性，在多核下处理器使用基于缓存加锁或者总线加锁的方式来实现原子操作。总线枷锁是指处理器提供一个 LOCK#信号，当一个处理器在总线上输出此信号时，其他处理器的请求会被阻塞，那么该处理器就可以独占内存。缓存枷锁是指内存区域如果被缓存到处理器的缓存行中，并且在 Lock 操作期间被锁定，那么当它执行锁操作回写到内存时，处理器不在总线上声明 LOCK#信号，而是修改内部的内存地址，并允许它的缓存一致性机制来保证操作的原子性。缓存一致性机制会阻止同时修改两处及以上的处理器缓存的内存数据，当其他处理器回写已被锁定的缓存行的数据时，会使得其他处理器的缓存航无效。这种机制使得其他处理器在读写该缓存航数据时只能从内存中获取到最新的数据后再进行操作。
cmpxchg 指令会声明 LOCK#信号对内存区域加锁保证原子性。


<a id="orgd116cd8"></a>

# 方法列表


<a id="orgc57333a"></a>

## Number 方法实现

```java
    /**
      * 返回 value 属性值
      */
     public int intValue() {
         return get();
     }
    
     /**
      * 将 value 属性值转成 long 类型返回
      */
     public long longValue() {
         return (long)get();
     }
    
     /**
      * 将 value 属性值转成 float 类型返回
      */
     public float floatValue() {
         return (float)get();
     }
    
     /**
      * 将 value 属性值转成 double 类型返回
      */
     public double doubleValue() {
         return (double)get();
     }
```

<a id="org7cb01e6"></a>

## AtomicInteger 操作整形数据的方法列表

```java
    // AtomicInteger 类包装的整型数据
    private volatile int value;
    
    /**
     * 构造方法
     *
     * @param initialValue 整型数据初始值
     */
    public AtomicInteger(int initialValue) {
        value = initialValue;
    }
    
    /**
     * 构造方法
     * 整型数据的初始值为 0
     */
    public AtomicInteger() {
    }
    
    /**
     * 返回当前的整型数据
     *
     * @return 整型数据
     */
    public final int get() {
        return value;
    }
    
    /**
     * 修改整型数据
     *
     * @param newValue 修改后的整型数据数值
     */
    public final void set(int newValue) {
        value = newValue;
    }
    
    /**
     * 修改整型数据
     * 该方法不会立即推送到主内存。该方法性能高但是不能保证线程安全，需要在其他保证线程安全的技术手段配合保证线程安全下使用该方法。
     *
     * @param newValue 修改后的整型数值
     * @since 1.6
     */
    public final void lazySet(int newValue) {
        unsafe.putOrderedInt(this, valueOffset, newValue);
    }
    
    /**
     * 将数值改成指定的值，并返回修改前的数值。
     *
     * @param newValue 修改后的数值
     * @return 修改前的数值
     */
    public final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }
    
    /**
     * 比较修改
     * 如果和预期的值相同则修改成指定的值。修改成功返回 true，否则返回 false
     *
     * @param expect 预期的现在数值
     * @param update 修改后的数值
     * @return 如果修改成功返回 true，修改失败返回 false
     */
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
    
    /**
     * 比较修改
     * 该方法在修改变量前后不会加入内存屏障，即该方法修改的变量不会立刻推到主内存中，也不会让其他线程的这个变量内存失效。
     * 该方法比 compareAndSet 的性能要好。
     *
     * @param expect 预期值
     * @param update 修改后的值
     * @return 修改成功则返回 true，否则返回 false
     */
    public final boolean weakCompareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
    
    /**
     * 原有值基础上加一
     *
     * @return 返回加一前的数值
     */
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
    
    /**
     * 原有值基础上减一
     *
     * @return 减一操作前的数值
     */
    public final int getAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }
    
    /**
     * 原有值基础上加上指定的数值
     *
     * @param delta 增加的数值
     * @return 加之前的数值
     */
    public final int getAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }
    
    /**
     * 原有值基础上加一并返回加一后的数值
     *
     * @return 加一后的结果
     */
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
    
    /**
     * 原有值减一并返回减一后的结果
     *
     * @return 减一后的结果
     */
    public final int decrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
    }
    
    /**
     * 原有值增加指定的数值并返回增加后的结果
     * @param delta 指定需要增加的数值
     * @return 增加后的结果
     */
    public final int addAndGet(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
    }
```

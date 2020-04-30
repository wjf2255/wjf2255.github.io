---
title: LockSupport 学习笔记
tags: [Java, JUC, 源码, 多线程, 锁]
layout: post
author: wjf
---

<div id="table-of-contents">
<h2>目录</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#orgf9d229b">概念</a></li>
<li><a href="#org5f7ad08">作用</a></li>
<li><a href="#org05ce561">Park 方法的实现</a></li>
<li><a href="#orgeca3eb3">UnPark 方法的实现</a></li>
</ul>
</div>
</div>


<a id="orgf9d229b"></a>

# 概念

-   许可：一种类似信号量（（java.util.concurrent.Semaphore）的技术，用来标记当前线程是否允许阻塞。但是不同于信号量可以设置多个信号标识，许可只有一个是否有效的标识。如果许可是有效的，可以理解为当前线程已经被阻塞；如果许可无效，那么可以理解线程未被阻塞。


<a id="org5f7ad08"></a>

# 作用

LockSupport 是一个线程同步工具，提供线程的阻塞和取消线程阻塞的功能。LockSupport 的阻塞和取消阻塞的功能是通过类似信号量（java.util.concurrent.Semaphore）技术的许可（permit）来实现的。如果调用阻塞方法 LockSupport.park()时，许可有效，那么该方法会立刻返回，否则该方法可能会阻塞。
Thread 本身也有类似的方法实现线程阻塞（Thread.suspend)和唤醒（Thread.resume)，但是这两个方法提供的线程阻塞和取消阻塞的方式有死锁的风险，所以已经被弃用了。LockSupport 提供 park 和 unpark 方法是替换 suspend 和 resume 方法的很好的方式。
除了提供无参数的 park 方法，LockSupport 还提供了带时间参数的 park 方法，用于设置等待时间。
另外 unpark 可以先于 park 方法调用，增加了使用了灵活性。


<a id="org05ce561"></a>

# Park 方法的实现

    public static void park(Object blocker) {
      Thread t = Thread.currentThread();
      setBlocker(t, blocker);
      UNSAFE.park(false, 0L);
      setBlocker(t, null);
    }
    
    private static void setBlocker(Thread t, Object arg) {
      // Even though volatile, hotspot doesn't need a write barrier here.
      UNSAFE.putObject(t, parkBlockerOffset, arg);
    }

java.lang.Thread 类有一个属性：volatile Object parkBlocker，当我们调用 park 方法时，会使用 UNSAFE.putObject(t, parkBlockerOffset, arg)对线程的 parkBlocker 属性设值。
我们可以通过调用 java.util.concurrent.locks.LockSupport.getBlocker 的方式获取线程阻塞的原因。
LockSupport.park 方法是通过调用底层的 UNSAFE.park(false, 0L)实现线程阻塞的。我们继续分析 UNSAFE.park 是如何实现的。UNSAFE.park 源码如下：

      void Parker::park(bool isAbsolute, jlong time) {
      // Ideally we'd do something useful while spinning, such
      // as calling unpackTime().
    
      // Optional fast-path check:
      // Return immediately if a permit is available.
      // We depend on Atomic::xchg() having full barrier semantics
      // since we are doing a lock-free update to _counter.
      //Parker 内置了一个_counter 字段，用来记录许可。如果当_counter>0 时，表示线程可以获得许可；否则不可以获得许可。即当调用 park 方法时，如果_counter>0，那么将_counter 设置为 0 并返回；
      if (Atomic::xchg(0, &_counter) > 0) return;
    
      //获取当前线程，并确认是 Java 线程
      Thread* thread = Thread::current();
      assert(thread->is_Java_thread(), "Must be JavaThread");
      JavaThread *jt = (JavaThread *)thread;
    
      // Optional optimization -- avoid state transitions if there's an interrupt pending.
      // Check interrupt before trying to wait
      //如果当前线程已经被中断，那么 park 方法也直接返回。
      if (Thread::is_interrupted(thread, false)) {
        return;
      }
    
      // Next, demultiplex/decode time arguments
      //如果等待的时间已经到了，那么直接返回
      timespec absTime;
      if (time < 0 || (isAbsolute && time == 0) ) { // don't wait at all
        return;
      }
      if (time > 0) {
        //将时间转成秒和纳秒存储起来
        unpackTime(&absTime, isAbsolute, time);
      }

      // Enter safepoint region
      // Beware of deadlocks such as 6317397.
      // The per-thread Parker:: mutex is a classic leaf-lock.
      // In particular a thread must never block on the Threads_lock while
      // holding the Parker:: mutex.  If safepoints are pending both the
      // the ThreadBlockInVM() CTOR and DTOR may grab Threads_lock.
      //将当前线程更新程阻塞状态
      ThreadBlockInVM tbivm(jt);
    
      // Don't wait if cannot get lock since interference arises from
      // unblocking.  Also. check interrupt before trying wait
      //如果线程被中断或者获取锁失败则直接返回
      if (Thread::is_interrupted(thread, false) || pthread_mutex_trylock(_mutex) != 0) {
        return;
      }
    
      int status ;
      if (_counter > 0)  { // no wait needed
        //将 park 内最的_counter 设置为 0，表示许可已经被使用。
        _counter = 0;
        //释放锁
        status = pthread_mutex_unlock(_mutex);
        assert (status == 0, "invariant") ;
        // Paranoia to ensure our locked and lock-free paths interact
        // correctly with each other and Java-level accesses.
        //增加一个内存屏障，确保线程安全
        OrderAccess::fence();
        return;
      }
    
    #ifdef ASSERT
      // Don't catch signals while blocked; let the running threads have the signals.
      // (This allows a debugger to break into the running thread.)
      sigset_t oldsigs;
      sigset_t* allowdebug_blocked = os::Linux::allowdebug_blocked_signals();
      pthread_sigmask(SIG_BLOCK, allowdebug_blocked, &oldsigs);
    #endif
      //将 java 线程所拥有的操作系统线程设置成 CONDVAR_WAIT 状态
      OSThreadWaitState osts(thread->osthread(), false /* not Object.wait() */);
      jt->set_suspend_equivalent();
      // cleared by handle_special_suspend_equivalent_condition() or java_suspend_self()
    
      if (time == 0) {
        //线程设置为等待状态。等待的信息是_cond（unpark 方法将会释放这个信号）
        status = pthread_cond_wait (_cond, _mutex) ;
      } else {
        status = os::Linux::safe_cond_timedwait (_cond, _mutex, &absTime) ;
        if (status != 0 && WorkAroundNPTLTimedWaitHang) {
          pthread_cond_destroy (_cond) ;
          pthread_cond_init    (_cond, NULL);
        }
      }
      assert_status(status == 0 || status == EINTR ||
                    status == ETIME || status == ETIMEDOUT,
                    status, "cond_timedwait");
    
    #ifdef ASSERT
      pthread_sigmask(SIG_SETMASK, &oldsigs, NULL);
    #endif
    
      _counter = 0 ;
      status = pthread_mutex_unlock(_mutex) ;
      assert_status(status == 0, status, "invariant") ;
      // Paranoia to ensure our locked and lock-free paths interact
      // correctly with each other and Java-level accesses.
      OrderAccess::fence();
    
      // If externally suspended while waiting, re-suspend
      if (jt->handle_special_suspend_equivalent_condition()) {
        jt->java_suspend_self();
      }
    }

如代码中显示，Parker 内置了一个_counter 字段，用来记录许可。如果当_counter >0 时，表示线程可以获得许可；否则不可以获得许可。即当调用 park 方法时，如果_counter >0，那么将_counter  设置为 0 并返回。
park 线程等待和唤醒是通过调用 POSIX 的线程 API，如 pthread_mutex_trylock，pthread_mutex_unlock，pthread_cond_wait，pthread_cond_destroy，pthread_cond_init 等，确保_counter  变量的线程安全和调用操作系统的线程等待和唤醒。unpark 也是一样。

<a id="orgeca3eb3"></a>

# UnPark 方法的实现

源码如下：

    public static void unpark(Thread thread) {
          if (thread != null)
              UNSAFE.unpark(thread);
      }

hotspot 源码如下：

    void Parker::unpark() {
      int s, status ;
      //添加互斥锁
      status = pthread_mutex_lock(_mutex);
      assert (status == 0, "invariant") ;
      s = _counter;
      //将_counter 的值设置为 1。表示有许可可用
      _counter = 1;
      if (s < 1) {
         if (WorkAroundNPTLTimedWaitHang) {
            //发送_cond 信号。park 方法中线程会阻塞等待这个信号才会唤醒。
            status = pthread_cond_signal (_cond) ;
            assert (status == 0, "invariant") ;
            status = pthread_mutex_unlock(_mutex);
            assert (status == 0, "invariant") ;
         } else {
            status = pthread_mutex_unlock(_mutex);
            assert (status == 0, "invariant") ;
            status = pthread_cond_signal (_cond) ;
            assert (status == 0, "invariant") ;
         }
      } else {
        pthread_mutex_unlock(_mutex);
        assert (status == 0, "invariant") ;
      }
    }



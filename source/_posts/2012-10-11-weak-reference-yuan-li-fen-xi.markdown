---
layout: post
title: "weak reference 原理分析"
date: 2012-10-11 12:33
comments: true
categories: [java,weak reference,技术分析]
---

## 前言

若干年前看了Java的四种引用类型，只是简单知道了不同类型的作用，但对其实现原理一直未能想明白，本文尝试结合jdk，openjdk6的部分源码分析弱引用实现的原理，供大家参考，部分技术细节没有仔细研究，如有疑问欢迎留言讨论

## 实例分析

我们以WeakHashMap的处理过程为例介绍一个weak reference的生命周期，首先我们调用WeakHashMap的put方法放入对象到Map中，WeakHashMap的Entry继承了WeakReference
 
{% codeblock  lang:java %}
private static class Entry<K,V> extends WeakReference<K> implements Map.Entry<K,V> { 
	 private V value; 
	 private final int hash; 
	 private Entry<K,V> next; 
{% endcodeblock %}

下面是put的部分代码

{% codeblock  lang:java %}
Entry<K,V> e = tab[i]; 
        tab[i] = new Entry<K,V>(k, value, queue, h, e); 
        if (++size >= threshold) 
            resize(tab.length * 2); 
        return null; 
    } 
{% endcodeblock %}

注意new Entry传递了一个reference queue到构造函数中，此构造函数最终会调用Reference的构造函数

{% codeblock  lang:java %}  
Reference(T referent, ReferenceQueue<? super T> queue) { 
    this.referent = referent; 
    this.queue = (queue == null) ? ReferenceQueue.NULL : queue; 
    } 
{% endcodeblock %}

referent是我们之前传入的hashmap的key对象，queue的作用是用来读取referent被回收的weak reference，生产者是谁后续介绍，此时WeakHashMap中已经存在了一个对象，先将key对象的strong ref制空并尝试触发gc，比如使用System.gc()来显式的触发gc，然后调用WeakHashMap的size方法返回集合的个数，绝大多数情况下会是0，这个过程中发生了什么呢？

第一步，key没有可达的strong ref，仅仅存在一个weak reference的referent变量仍然指向了key，触发GC时，以openjdk6的parNew为例，jvm在young generation gc时会尝试获取Reference对象里的静态全局锁
 
{% codeblock  lang:java %}
/* Object used to synchronize with the garbage collector.  The collector 
     * must acquire this lock at the beginning of each collection cycle.  It is 
     * therefore critical that any code holding this lock complete as quickly 
     * as possible, allocate no new objects, and avoid calling user code. 
     */ 
    static private class Lock { }; 
    private static Lock lock = new Lock(); 
{% endcodeblock %} 	
	
在openjdk6里的部分源代码,完整代码请参考instanceRefKlass.cpp文件

{% codeblock  lang:c %}
void instanceRefKlass::acquire_pending_list_lock(BasicLock *pending_list_basic_lock) { 
  // we may enter this with pending exception set 
  PRESERVE_EXCEPTION_MARK;  // exceptions are never thrown, needed for TRAPS argument 
  Handle h_lock(THREAD, java_lang_ref_Reference::pending_list_lock()); 
  ObjectSynchronizer::fast_enter(h_lock, pending_list_basic_lock, false, THREAD); 
  assert(ObjectSynchronizer::current_thread_holds_lock( 
           JavaThread::current(), h_lock), 
         "Locking should have succeeded"); 
  if (HAS_PENDING_EXCEPTION) CLEAR_PENDING_EXCEPTION; 
} 
{% endcodeblock %}

 此处代码在parNew gc时执行，目的就是尝试获取全局锁，在gc完成后，jvm会将key被回收的weak reference组成一个queue并赋值到Reference的pending属性然后释放锁，参考方法：

{% codeblock  lang:c %}
void instanceRefKlass::release_and_notify_pending_list_lock( 
  BasicLock *pending_list_basic_lock) { 
  // we may enter this with pending exception set 
  PRESERVE_EXCEPTION_MARK;  // exceptions are never thrown, needed for TRAPS argument 
  // 
  Handle h_lock(THREAD, java_lang_ref_Reference::pending_list_lock()); 
  assert(ObjectSynchronizer::current_thread_holds_lock( 
           JavaThread::current(), h_lock), 
         "Lock should be held"); 
  // Notify waiters on pending lists lock if there is any reference. 
  if (java_lang_ref_Reference::pending_list() != NULL) { 
    ObjectSynchronizer::notifyall(h_lock, THREAD); 
  } 
  ObjectSynchronizer::fast_exit(h_lock(), pending_list_basic_lock, THREAD); 
  if (HAS_PENDING_EXCEPTION) CLEAR_PENDING_EXCEPTION; 
}
{% endcodeblock %}

在一次gc后，Reference对象的pending属性不再为空，让我们看看Reference的部分代码
首先是pending属性的说明：

{% codeblock  lang:java %}
/* List of References waiting to be enqueued.  The collector adds 
 * References to this list, while the Reference-handler thread removes 
 * them.  This list is protected by the above lock object. 
 */ 
private static Reference pending = null; 
{% endcodeblock %}

接下来是Reference中的内部类ReferenceHandler，它继承了Thread，看看run方法的代码

{% codeblock  lang:java %}
public void run() { 
        for (;;) { 
 
        Reference r; 
        synchronized (lock) { 
            if (pending != null) { 
            r = pending; 
            Reference rn = r.next; 
            pending = (rn == r) ? null : rn; 
            r.next = r; 
            } else { 
            try { 
                lock.wait(); 
            } catch (InterruptedException x) { } 
            continue; 
            } 
        } 
 
        // Fast path for cleaners 
        if (r instanceof Cleaner) { 
            ((Cleaner)r).clean(); 
            continue; 
        } 
 
        ReferenceQueue q = r.queue; 
        if (q != ReferenceQueue.NULL) q.enqueue(r); 
        } 
    } 
    } 
{% endcodeblock %}

一旦jvm notify了前面提到的锁，这个线程就被激活并开始执行，作用是将之前jvm赋值过来的pending对象中的WeakReference对象enqueue到指定的队列中，比如WeakHashMap内部定义的ReferenceQueue属性
此时map的queue中保存了referent已经被回收的WeakReference队列，也就是map的Entry对象，当调用size方法时，内部首先调用expungStaleEntries方法清除被回收掉的Entry，代码如下

{% codeblock  lang:java %}
private void expungeStaleEntries() { 
    Entry<K,V> e; 
        while ( (e = (Entry<K,V>) queue.poll()) != null) { 
            int h = e.hash; 
            int i = indexFor(h, table.length); 
 
            Entry<K,V> prev = table[i]; 
            Entry<K,V> p = prev; 
            while (p != null) { 
                Entry<K,V> next = p.next; 
                if (p == e) { 
                    if (prev == e) 
                        table[i] = next; 
                    else 
                        prev.next = next; 
                    e.next = null;  // Help GC 
                    e.value = null; //  "   " 
                    size--; 
                    break; 
                } 
                prev = p; 
                p = next; 
            } 
        } 
    } 
{% endcodeblock %}

ok，就这样map的废弃Entry被clear，size返回为0
 
## 参考
- [understanding-weak-references](http://weblogs.java.net/blog/2006/05/04/understanding-weak-references )
- [when-would-you-use-a-weakhashmap-or-a-weakreference](http://stackoverflow.com/questions/154724/when-would-you-use-a-weakhashmap-or-a-weakreference )  

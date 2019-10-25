---
title: ThreadLocal

categories: 
- Java

tags:
- 线程
---

ThreadLocal是一个关于创建线程局部变量的类。

通常情况下，我们创建的变量是可以被任何一个线程访问并修改的。而使用ThreadLocal创建的变量只能被当前线程访问，其他线程则无法访问和修改。

ThreadLocal支持泛型，创建跟一般类一样new一个对象就可以了。创建完对象后就可以用set方法设置值

<!--more-->
```

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

```


ThreadLocalMap是ThreadLocal的内部类


```
    static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        
        private Entry[] table;
    }
```

ThreadLocalMap用Entry类来进行存储,
我们的值都是存储到这个Map上的，key是当前ThreadLocal对象！
如果该Map不存在，则初始化一个,如果该Map存在，则从Thread中获取！


```
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

```
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

ThreadLocalMap是ThreadLocal的内部类，但对象的引用是在Thread中

```
public class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

Thread为每个线程维护了ThreadLocalMap这么一个Map，而ThreadLocalMap的key是LocalThread对象本身，value则是要存储的对象



再来稍微深入了解下，之前看到真正的值是存在ThreadLocalMap中的，ThreadLocal的set方法是调用了ThreadLocalMap的set方法


```
    private void set(ThreadLocal<?> key, Object value) {

        // We don't use a fast path as with get() because it is at
        // least as common to use set() to create new entries as
        // it is to replace existing ones, in which case, a fast
        // path would fail more often than not.

        ThreadLocal.ThreadLocalMap.Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);

        for (ThreadLocal.ThreadLocalMap.Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            ThreadLocal<?> k = e.get();

            if (k == key) {
                e.value = value;
                return;
            }

            if (k == null) {
                replaceStaleEntry(key, value, i);
                return;
            }
        }

        tab[i] = new ThreadLocal.ThreadLocalMap.Entry(key, value);
        int sz = ++size;
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
    }
```


set方法就是通过hash算出当前ThreadLocal对象在table（Entry数组）中的位置，然后进行如下判断和操作


1. 如果当前位置的Entry不为空
    1. 如果key（其实不是key，是Entry的父类WeakReference所继承的Reference的成员变量referent，这里我觉得就当作key比较好理解）是当前ThreadLocal对象，那么就对Entry的value重新赋值
    2. 如果key为空则进行调用replaceStaleEntry方法将当前Entry的key和value都进行替换
    3. 以上两中条件都不满足则重新定位直到下一位置重复以上判断直到符合条件或者结束for循环
2. 如果当前位置的Entry为空则直接赋值
---
title: CountDownLatch和AQS
date: 2020-07-30
categories: 
- Java

tags:
- 并发
---

### 简介

CountDownLatch 的作用是：当一个线程需要另外一个或多个线程完成后，再开始执行。比如主线程要等待一个子线程完成环境相关配置的加载工作，主线程才继续执行，就可以利用 CountDownLatch 来实现。

<!--more-->

之前写过一个例子


```
public class CountDownLatchTest {

    private static CountDownLatch countDownLatch = new CountDownLatch(2);

    public static void main(String[] args) throws InterruptedException {

        new Thread(() -> {
            System.out.println(1);
            countDownLatch.countDown();
        }).start();
        
        new Thread(() -> {
            System.out.println(2);
            countDownLatch.countDown();
        }).start();

        countDownLatch.await();

        System.out.println(3);
    }
}
```


执行countDownLatch.await()会阻塞主线程直到其他线程执行完毕。


### await

CountDownLatch就是通过AQS来实现的，CountDownLatch有一个内部类继承了AQS并实现了其中某些方法。


```
public class CountDownLatch {
    
    private static final class Sync extends AbstractQueuedSynchronizer {
        
    }
}
```


主线程会在await()方法后进入阻塞状态等待唤起，实际是调用了AQS的acquireSharedInterruptibly(int arg)方法。


```
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    
    // AQS
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

acquireSharedInterruptibly(int arg)会先判断当前线程是否被中断，如果未被中断的话则进行tryAcquireShared(arg)方法判断，Sync重写了tryAcquireShared(arg)方法

```
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
```


当子线程未执行完时返回-1执行doAcquireSharedInterruptibly(arg)获取共享锁。


```
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```


### countDown

当调用CountDownLatch的countDown()方法时，会去调用Sync的releaseShared()方法，这个方法时public final类型的，实际是在AQS中


```
public class CountDownLatch {
    public void countDown() {
        sync.releaseShared(1);
    }
}

public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    
        public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
}
```


而Sync重写了tryReleaseShared(arg)方法


```
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```

当state不为0的时候，通过CAS将state减1直至成功为止。当我们第一次调用CountDownLatch的countDown()方法时， nextc == 1所以返回false，执行结束。 而第二次调用的时候，nextc == 0返回true，这个时候按照逻辑会执行AQS的doReleaseShared()方法。


```
    private void doReleaseShared() {
        
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

doReleaseShared()是一个释放共享锁的操作，当CountDownLatch的state为0时就表示所有子线程的操作已经执行完毕，可以唤起主线程。


### 总结

CountDownLatch主要由Sync实现，Sync的主要逻辑是由AQS实现的，不过它自身重写了其中一部分。


```
public class CountDownLatch {
    
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
}
```


CountDownLatch是基于共享锁实现的并发控制功能，首先在构造器初始化CountDownLatch的时候，就会给AQS中的state赋值，表示共享锁已经被获取了N次；然后每执行一次countDown则共享锁释放一次，直到释放完；await方法是加锁的逻辑，但加锁条件是state==0时才会加锁成功，否则挂起；最后，当通过countDown的调用将state减为0后，会唤醒处于阻塞状态的主线程，让其获取到锁并执行。


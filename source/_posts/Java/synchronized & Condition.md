---
title: synchronized和Condition实现生产者和消费者
date: 2019-07-02
categories: 
- Java

tags:
- 线程
---
# 使用synchronized和Condition分别实现生产者和消费者的消息队列
## synchronized实现

<!--more-->
---

```

package condition;

import java.util.LinkedList;
import java.util.concurrent.TimeUnit;


public class MyContainer1<T> {
    final private LinkedList<T> list = new LinkedList<>();
    final private int MAX = 10; //最多10个元素
    private int count = 0;


    public synchronized void put(T t) {
        while (list.size() == MAX) {
            try {
                this.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        list.add(t);
        ++count;
        System.out.println("生产者容器中的物品数: " + count);
        this.notifyAll(); //通知消费者线程进行消费
    }

    public synchronized T get() {
        T t = null;
        while (list.size() == 0) {
            try {
                this.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        t = list.removeFirst();
        count--;
        System.out.println("消费者容器中的物品数: " + count);
        this.notifyAll(); //通知生产者进行生产
        return t;
    }

    public static void main(String[] args) {
        MyContainer1<String> c = new MyContainer1<>();
        //启动消费者线程
        for(int i=0; i<10; i++) {
            new Thread(()->{
                for(int j=0; j<5; j++)
                    c.get();
                //System.out.println(c.get());
            }, "c" + i).start();
        }

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        //启动生产者线程
        for(int i=0; i<2; i++) {
            new Thread(()->{
                for(int j=0; j<25; j++)
                    c.put(Thread.currentThread().getName() + " " + j);
            }, "p" + i).start();
        }

    }
}

```


---
## Condition实现

---

```

package condition;

import java.util.LinkedList;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;


public class MyContainer2<T> {

    final private LinkedList<T> list = new LinkedList<>();
    final private int MAX = 10; //最多10个元素
    private int count = 0;

    private Lock lock = new ReentrantLock();
    private Condition producer = lock.newCondition();
    private Condition consumer = lock.newCondition();

    public void put(T t) {
        try {
            lock.lock();
            while (list.size() == MAX) {
                producer.await();
            }

            list.add(t);
            ++count;
            //通知消费者线程进行消费
            consumer.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public T get() {
        T t = null;
        try {
            lock.lock();
            while (list.size() == 0) {
                consumer.await();
            }
            t = list.removeFirst();
            count--;
            //通知生产者进行生产
            producer.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
        return t;
    }

    public static void main(String[] args) {
        MyContainer2<String> c = new MyContainer2<>();
        //启动消费者线程
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                for (int j = 0; j < 5; j++)
                    System.out.println(Thread.currentThread().getName() + "取出" + c.get());
            }, "c" + i).start();
        }

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        //启动生产者线程
        for (int i = 0; i < 2; i++) {
            new Thread(() -> {
                for (int j = 0; j < 25; j++) {
                    System.out.println(Thread.currentThread().getName() + "插入" + j);
                    c.put(Thread.currentThread().getName() + " " + j);
                }
            }, "p" + i).start();
        }
    }
}



```

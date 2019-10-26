---
title: 并发工具类CountDownLatch和CyclicBarrier
date: 2019-01-02
categories: 
- Java

tags:
- 线程
- 并发
---


# CountDownLatch

CountDownLatch允许一个或多个线程等待其它线程操作完成，比如登录后返回结果前要获取用户头像用户昵称就可以分开进行

<!--more-->


```

public class CountDownLatchTest {

    private static CountDownLatch countDownLatch = new CountDownLatch(2);

    public static void main(String[] args) throws InterruptedException {

        new Thread(() -> {
            System.out.println(1);
            countDownLatch.countDown();
            System.out.println(2);
            countDownLatch.countDown();
        }).start();

        countDownLatch.await();

        System.out.println(3);
    }
}

```

countDown的构造器接收一个int参数作为计数器，想等N个点完成就传入N，CountDownLatch的countDown方法会使计数器减1。上面这段代码， System.out.println(3)语句一定是最后执行的， CountDownLatch的await方法会阻塞当前线程，直到N变为0为止


# CyclicBarrier

CyclicBarrier的作用是让一组线程达到屏障时（也可以叫同步点）被阻塞，直到最后一个线程达到屏障时，屏障被打开，所以被屏障阻塞的线程运行


```

public class CyclicBarrierTest {

    private static CyclicBarrier cyclicBarrier = new CyclicBarrier(3);
//    private static CyclicBarrier cyclicBarrier = new CyclicBarrier(2);

    public static void main(String[] args) {

        new Thread(() -> {
            try {
                cyclicBarrier.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }

            System.out.println(1);
        }).start();


        try {
            cyclicBarrier.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }

        System.out.println(2);
    }
}

```

上面代码主线程和子线程会永远等待，因为没有第三个线程执行await方法，即没有第三个线程达到屏障，需要把cyclicBarrier构造器参数变成2才行


CountDownLatch和CyclicBarrier的区别是CountDownLatch计数器只能使用一次，CyclicBarrier的计数器可以用reset()方法重置
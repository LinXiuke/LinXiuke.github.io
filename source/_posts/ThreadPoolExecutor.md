---
title: ThreadPoolExecutor

categories: 
- Java

tags:
- 线程
---


ThreadPoolExecutor是Executor框架最核心的类，也是线程池的实现类，有以下4个组件构成。

1. corePool：核心线程池大小 
2. maximumPool：最大线程池大小
3. BlockingQueue：用来暂时保存任务的工作队列
4. RejectedExecutionHandler：当ThreadPoolExecutor已经关闭或者已经饱和（达到最大线程池大小并且工作队列已满），execute()方法将调用Handler
    
<!--more-->

通过工具类Executors可以创建三种类型的ThreadPoolExecutor

1. FixedThreadPool
2. SingleThreadExecutors
3. CacheThreadPool


## 1. **FixedThreadPool**

FixedThreadPool被称为可重用固定线程数的线程池

```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
}
```

可以看到FixedThreadPool的其实是通过创建ThreadPoolExecutor来实现的

```
new  ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue)
```

当线程池的线程数大于corePoolSize的时候，keepAliveTime为多余的空闲线程等待任务的最长时间，超过时间后多余的线程被终止。FixedThreadPool的创建中把keepAliveTime为多余的空闲线程等待任务的最长时间设置为0L表示多余的空闲线程立即被终止。


## 2.**SingleThreadExecutor**

SingleThreadExecutor是使用单个worker线程的Executor，下面是SingleThreadExecutor的源码实现

```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService(new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>()));
}
```
SingleThreadExecutor的corePoolSize和maximumPoolSize设置为1，其他参数和newFixedThreadPool一样。

SingleThreadExecutor的execute()方法

1. 如果当前运行线程数少于corePollSize（即线程池中无运行的线程），则创建一个新的线程执行任务。
2. 在线程池完成预热之后（当前线程中有一个运行线程）将任务加入LinkedBlockingQueue。
3. 线程执行完1中的任务后，会在一个无限循环中反复从LinkedBlockingQueue获取任务来完成。
 

## 3. **CacheThreaPool**

CacheThreaPool是一个会根据需要创建新线程的线程池。下面是创建CacheThreaPool的源码。

```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue<Runnable>());
}
```
CacheThreaPool的corePoolSize被设置为0，即corePool为空，maximumPoolSize设置为Integer.MAX_VALUE，表示maximumPool是无界的。
CacheThreaPool使用的是没有容量的SynchronousQueue作为线程池的工资队列，但是maximumPool是无界的，这意味着当主线程提交任务的速度高于maximumPool中线程处理任务的速度时，CacheThreaPool会不断创建新的线程，极端情况下，CacheThreaPool会耗尽CPU和内存资源。


## ScheduledThreadPoolExecutor
ScheduledThreadPoolExecutor继承自ThreadPoolExecutor，主要用来在给定的延迟之后执行任务或者定期执行任务。
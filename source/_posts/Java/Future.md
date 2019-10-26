---
title: Future的使用
date: 2019-06-04
categories: 
- Java

tags:
- 线程
---

Java 通过 ThreadPoolExecutor 提供的3个submit()方法和FutureTask来获取多线程任务结果

####  ThreadPoolExecutor
3个submit()方法如下

<!--more-->

```
// 提交 Runnable 任务
Future<?> submit(Runnable task);

// 提交 Callable 任务
<T> Future<T> submit(Callable<T> task);

// 提交 Runnable 任务及结果引用  
<T> Future<T> submit(Runnable task, T result);

```
这3个方法的区别主要在与提交参数的不同

1. submit(Runnable task)  方法的参数是一个Runable接口，Runable接口的run()方法时没有返回值的，所以该方法只能用来断言任务结束，类似Thread.join()
2. submit(Callable<T> task)方法参数是一个Callable接口，Callable接口只有call()方法并且有返回值，所以返回的Future对象是可以通过调用get()方法来获取结果
3. submit(Runnable task, T result) 这个方法返回的Future调用其get()方法获取的就是参数中的result，使用该方法要注意Runable接口的实现类要能对result参数进行操作，这样result可以在主线程和子线程之间共享数据
 


```
public class FutureTest {

    public static void main(String[] args) throws InterruptedException, ExecutionException {

        ExecutorService executorService = Executors.newFixedThreadPool(1);
        Result result = new Result();
        Future<Result> future = executorService.submit(new Task(result), result);
        Result futureResult = future.get();

        System.out.println(futureResult == result); //ture
        System.out.println(futureResult.getName()); //over

        executorService.shutdown();
    }
}

class Task implements Runnable {

    private Result result;

    public Task(Result result) {
        this.result = result;
    }

    @Override
    public void run() {
        result.setName("over");
    }
}

class Result {

    private String name;

    String getName() {
        return name;
    }

    void setName(String name) {
        this.name = name;
    }
}
```

Future一共有五个方法，其余四个如下

```
// 取消任务
boolean cancel( boolean mayInterruptIfRunning);

// 判断任务是否已取消  
boolean isCancelled();

// 判断任务是否已结束
boolean isDone();

// 获得任务执行结果
get();

// 获得任务执行结果，支持超时
get(long timeout, TimeUnit unit);

```


#### FutureTask

FutureTask是一个工具类，有两个构造方法

```
FutureTask(Callable<V> callable);
FutureTask(Runnable runnable, V result);
```
这两个构造方法类似于ThreadPoolExecutor的submit()方法


FutureTask实现了Runable接口和Future接口
1. 实现了Runable接口所有可以提交给ThreadPoolExecutor执行，也可以直接通过Thread执行
2. 实现了Future接口所以能用来获取任务的执行结果


提交给ThreadPoolExecuto实现

```
// 创建 FutureTask
FutureTask<Integer> futureTask = new FutureTask<>(()-> 1+2);
// 创建线程池
ExecutorService es = Executors.newCachedThreadPool();
// 提交 FutureTask 
es.submit(futureTask);
// 获取计算结果
Integer result = futureTask.get();

```

直接被Thread执行

```
// 创建 FutureTask
FutureTask<Integer> futureTask = new FutureTask<>(()-> 1+2);
// 创建并启动线程
Thread T1 = new Thread(futureTask);
T1.start();
// 获取计算结果
Integer result = futureTask.get();

```

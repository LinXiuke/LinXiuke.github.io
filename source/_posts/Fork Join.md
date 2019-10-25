---
title: Fork/Join 框架

categories: 
- Java

tags:
- 线程
---

Fork/Join框架首先要考虑的是分割任务，当任务计算过大时分割成两个子任务分别计算

<!--more-->

```
//因为要获取计算结果所以要继承RecursiveTask， RecursiveTask则是继承了ForkJoinTask

public class CountTask extends RecursiveTask<Integer> {

    //阈值
    private static final int THRESHOLD = 2;

    private int start;
    private int end;

    public CountTask(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {

        int sum = 0;

        boolean canCompute = (end - start) <= THRESHOLD;
        if (canCompute) {
            //任务足够小可以进行计算
            for (int i = start; i < end; i++) {
                sum += i;
            }
        } else {
            // 分裂成两个子任务
            int middle = (start + end) / 2;
            CountTask leftTask = new CountTask(start, middle);
            CountTask rightTask = new CountTask(middle + 1, end);

            //执行子任务
            leftTask.fork();
            rightTask.fork();

            int leftResult = leftTask.join();
            int rightResult = rightTask.join();

            sum = leftResult + rightResult;
        }

        return sum;
    }

    public static void main(String[] args) {

        ForkJoinPool forkJoinPool = new ForkJoinPool();
        CountTask task = new CountTask(1, 4);
        //执行任务
        Future<Integer> result = forkJoinPool.submit(task);

        try {
            System.out.println(result.get());
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }
}

```


ForkJoinTask需要实现compute方法，在这个方法里首先要判断任务是否足够小，不够小就必须分割成两个子任务，每个子任务在调用fork方法时又会进入compute方法。使用join方法会等待子任务执行完成并获取结果


当我们调用fork方法的时候程序会调用ForkJoinWorkerThread的push方法并返回结果


```
    //Java8
    
    public final ForkJoinTask<V> fork() {
        Thread t;
        if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
            ((ForkJoinWorkerThread)t).workQueue.push(this);
        else
            ForkJoinPool.common.externalPush(this);
        return this;
    }
```


push方法是将当前任务放在ForkJoinTask数组队列里， 然后再调用ForkJoinPool的signalWork唤醒或者创建一个新的线程来执行任务


```
    final void push(ForkJoinTask<?> task) {
        ForkJoinTask<?>[] a; ForkJoinPool p;
        int b = base, s = top, n;
        if ((a = array) != null) {    // ignore if queue removed
            int m = a.length - 1;     // fenced write for task visibility
            U.putOrderedObject(a, ((m & s) << ASHIFT) + ABASE, task);
            U.putOrderedInt(this, QTOP, s + 1);
            if ((n = s - b) <= 1) {
                if ((p = pool) != null)
                    p.signalWork(p.workQueues, this);
            }
            else if (n >= m)
                growArray();
        }
    }
```


join方法主要是阻塞当前线程并等待获取结果


```
    public final V join() {
        int s;
        if ((s = doJoin() & DONE_MASK) != NORMAL)
            reportException(s);
        return getRawResult();
    }
    
    private int doJoin() {
        int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
        return (s = status) < 0 ? s :
            ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
            (w = (wt = (ForkJoinWorkerThread)t).workQueue).
            tryUnpush(this) && (s = doExec()) < 0 ? s :
            wt.pool.awaitJoin(w, this, 0L) :
            externalAwaitDone();
    }
    
    private void reportException(int s) {
        if (s == CANCELLED)
            throw new CancellationException();
        if (s == EXCEPTIONAL)
            rethrow(getThrowableException());
    }
```


通过doJoin方法来得到当前任务的状态判断返回什么结果，任务状态：NORMAL，CANCELLED，SIGNAL，EXCEPTIONAL。
当前任务如果执行完成则直接返回结果，为完成就执行，执行完成状态改为NORMAL， 出现异常则记录异常并将状态改为EXCEPTIONAL。
---
title: Redisson

date: 2021-10-15

categories:
- Redis

---

## Java锁

- synchronized
- Lock


单机锁，不适合分布式场景

## 分布式场景下的锁

### 锁的种类

1. MySQL
2. Zookeeper
3. Reids


### MySQL

自带悲观锁for update关键字；
- 优点：理解起来简单，不需要维护额外的第三方中间件(比如Redis,Zk)
- 缺点：需要自己考虑锁超时，加事务等等，性能局限于数据库

### Zookeeper

Zookeeper提供一个多层级的节点命名空间（节点称为znode），每个节点都用一个以斜杠（/）分隔的路径表示，而且每个节点都有父节点（根节点除外），类似于文件系统。

#### 有序节点

客户端申请向ZK加一个key为“my_lock”锁直接在“my_lock”这个锁节点下，创建一个顺序节点，这个顺序节点有zk内部自行维护的一个节点序号。比如客户端A申请加锁，会创建一个“xxx-0001”的节点，客户端B申请加锁创建一个“xxx-002”的节点，创建节点成功后，客户端会判断序号最小的节点，如果是则加锁成功。

#### 临时节点

考虑这么个场景：假如客户端A当前创建的子节点为序号最小的节点，获得锁之后客户端所在机器宕机了，客户端没有主动删除子节点；如果创建的是永久的节点，那么这个锁永远不会释放，导致死锁；由于创建的是临时节点，客户端宕机后，过了一定时间zookeeper没有收到客户端的心跳包判断会话失效，将临时节点删除从而释放锁。

#### 事件监听

如果获得锁的客户端释放锁时将其他客户端全部唤醒，当客户端数量较多时容易阻塞其他操作。所有客户端都被唤醒，这种情况被称之为“羊群效应”。

最好的情况应该只唤醒新的最小节点对应的客户端，做法就是通过事件监听，每一个客户端监听他所创建节点的前一个节点。


### Redis

提供了SETNX(set if not exists)这样具有互斥性的指令

## Redis锁

### 使用

Java自带的关键字synchronized和jdk中的Lock类只适合单个应用，不适合分布式场景，通过Redis可以设置分布式锁，spring集成Redis后的RedisTemplate提供了API

```
@Component
public class RedisLockValue {
    
    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    /**
     * 加锁
     * @param key
     * @return
     */
    public boolean tryLock(String key) {
        return stringRedisTemplate.opsForValue().setIfAbsent(key, "value");
    }

    /**
     * 释放锁
     * @param key
     */
    public void releaseLock(String key) {
        stringRedisTemplate.delete(key);
    }

}
```

### 引发的问题

#### 锁未释放

当服务A获取的Redis锁后因为某些意外情况挂了，这个时候服务A还没来得及删除Redis锁，这样的话其他服务就都无法获取锁，这个时候时候就要给锁设置上超时时间


```
    /**
     * 加锁
     * @param key
     * @return
     */
    public boolean tryLock(String key) {
        return stringRedisTemplate.opsForValue().setIfAbsent(key, "value", 30, TimeUnit.SECONDS);
    }

    /**
     * 释放锁
     * @param key
     */
    public void releaseLock(String key) {
        stringRedisTemplate.delete(key);
    }
```

但是设置了超时时间后又引出了另一个问题：当执行的业务逻辑过于复杂或者其他原因，执行时间超出了锁的时间，导致其他线程提前获取了锁。这种情况下就要设置合理的超时时间但是又不能过长，这个时候就需要每隔一段时间延长加锁时间。


```
    /**
     * 加锁
     *
     * @param key
     * @return
     */
    public boolean tryLock(String key) {
        boolean success = stringRedisTemplate.opsForValue().setIfAbsent(key, "value", 30, TimeUnit.SECONDS);
        // 加锁成功，异步续命
        if (success) {
            scheduleExpirationRenewal(key);
        }

        return success;
    }

    /**
     * 异步延长加锁时间
     *
     * @param key
     */
    public void scheduleExpirationRenewal(String key) {
        new Thread(() -> {
            while (true) {
                try {
                    Thread.sleep(10000);
                    if (Objects.nonNull(stringRedisTemplate.opsForValue().get(key))) {
                        stringRedisTemplate.expire(key, 30, TimeUnit.SECONDS);
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
```



#### 锁释放的安全性

Redis的加锁实际上是通过判断set一个key成功来判断的，释放锁则是将这个key删除。

当使用synchronized或者Lock持有锁后，释放锁的动作也只能有持有锁的线程来执行，但是现在这种情况我们只要调用RedisLock#releaseLock方法就可以删除Redis的key从而释放锁。为了保证释放锁的安全我们要记录下加锁的线程，只有加锁的线程才能手动释放锁。

```
    /**
     * 加锁，并记录线程id
     *
     * @param key
     * @return
     */
    public boolean tryLock1(String key) {
        long threadId = Thread.currentThread().getId();
        boolean success = stringRedisTemplate.opsForValue().setIfAbsent(key, threadId + "", 30, TimeUnit.SECONDS);
        // 加锁成功，异步续命
        if (success) {
            scheduleExpirationRenewal(key);
        }
        return success;
    }

    /**
     * 释放锁，先判断当前线程是否为加锁线程
     *
     * @param key
     */
    public void releaseLock1(String key) {
        String value = stringRedisTemplate.opsForValue().get(key);
        if (value != null && Long.valueOf(value).equals(Thread.currentThread().getId())) {
            stringRedisTemplate.delete(key);
        }
    }
```

#### 可重入

synchronized或者ReentrantLock都是可重入锁，即当前线程可以重复持有锁，同时加锁多次就需要解锁多次。AQS中有个state变量作为计数器，这里也需要一个计数器来记录加锁次数

```
    /**
     * 可重入加锁
     *
     * @param key
     * @return
     */
    public boolean tryLock2(String key) {

        Long currentThreadId = Thread.currentThread().getId();
        boolean success = false;

        String value = stringRedisTemplate.opsForValue().get(key);
        // 未加锁，可直接加锁
        if (value == null) {
            success = stringRedisTemplate.opsForValue().setIfAbsent(key, currentThreadId + "-" + 1, 30, TimeUnit.SECONDS);
        //已加锁
        } else {
            String[] values = value.split("-");
            // 当前线程是之前加锁的线程
            if (currentThreadId.equals(Long.valueOf(values[0]))) {
                int count = Integer.parseInt(values[1]);
                // 加锁计数
                count++;
                success = stringRedisTemplate.opsForValue().setIfAbsent(key, currentThreadId + "-" + count, 30, TimeUnit.SECONDS);
            }
        }

        // 加锁成功，异步续命
        if (success) {
            scheduleExpirationRenewal(key);
        }
        return success;
    }

    /**
     * 可重入释放锁
     *
     * @param key
     */
    public void releaseLock2(String key) {

        Long currentThreadId = Thread.currentThread().getId();
        String value = stringRedisTemplate.opsForValue().get(key);
        // 未加锁无需释放
        if (value == null) {
            return;
        }

        String[] values = value.split("-");
        // 当前线程是之前加锁的线程
        if (currentThreadId.equals(Long.valueOf(values[0]))) {
            int count = Integer.parseInt(values[1]);
            // 加锁次数大于1，计数-1
            if (count > 1) {
                count--;
                stringRedisTemplate.opsForValue().set(key, currentThreadId + "-" + count, 30, TimeUnit.SECONDS);
            // 只加了一次锁，直接释放
            } else {
                stringRedisTemplate.delete(key);
            }
        }
    }

```


#### 阻塞

synchronized竞争锁是阻塞性的，ReentrantLock也实现了阻塞性的lock方法


```
    /**
     * 阻塞加锁
     *
     * @param key
     */
    public void lock(String key) {

        Long currentThreadId = Thread.currentThread().getId();
        boolean success = false;

        String value = stringRedisTemplate.opsForValue().get(key);
        // 未加锁，可直接加锁
        if (value == null) {
            success = stringRedisTemplate.opsForValue().setIfAbsent(key, currentThreadId + "-" + 1, 30, TimeUnit.SECONDS);
            //已加锁
        } else {
            String[] values = value.split("-");
            // 当前线程是之前加锁的线程
            if (currentThreadId.equals(Long.valueOf(values[0]))) {
                int count = Integer.parseInt(values[1]);
                // 加锁计数
                count++;
                success = stringRedisTemplate.opsForValue().setIfAbsent(key, currentThreadId + "-" + count, 30, TimeUnit.SECONDS);
            }
        }

        // 加锁失败再次尝试直到成功
        if (!success) {
            while (true) {
                success = stringRedisTemplate.opsForValue().setIfAbsent(key, currentThreadId + "-" + 1, 30, TimeUnit.SECONDS);
                if (success) {
                    break;
                }
            }
        }

        // 加锁成功，异步续命
        scheduleExpirationRenewal(key);
    }
```


## Redisson锁

Redisson锁其实就是解决了上面所说的问题。源码大概流程（补充了一些自己的注释）：

### 加锁&可重入

```
	/**
     * 加锁
     *
     * @param leaseTime     加锁到期时间, -1 使用默认值 30 秒
     * @param unit          时间单位
     * @param interruptibly 是否可被中断标识
     * @throws InterruptedException
     */
    private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
        long threadId = Thread.currentThread().getId();
        // 尝试加锁
        Long ttl = tryAcquire(-1, leaseTime, unit, threadId);
        // 加锁成功
        if (ttl == null) {
            return;
        }

        //...
    }
    
    private Long tryAcquire(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
        return get(tryAcquireAsync(waitTime, leaseTime, unit, threadId));
    }
    
    private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
        RFuture<Long> ttlRemainingFuture;
        // 执行tryLock(long waitTime, long leaseTime, TimeUnit unit) tryLock(long waitTime, TimeUnit unit) 才会进入
        if (leaseTime != -1) {
            ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
        } else {
            // 尝试异步获取锁, 获取锁成功返回空, 否则返回锁剩余过期时间
            // 异步加锁，并使用lua脚本保证原子性
            ttlRemainingFuture = tryLockInnerAsync(waitTime,  internalLockLeaseTime,
                    TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
        }
        // ...
    }
    
    <T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
                // 如果key不存在，直接set并设置过期时间，返回空值
                "if (redis.call('exists', KEYS[1]) == 0) then " +
                        // 当 KEYS[1] == 0 时代表当前没有锁
                        // 使用 hincrby 命令发现 KEYS[1] 不存在并新建一个 hash
                        // ARGV[2] (uuid+线程id) 就作为 hash 的第一个key, value 为 1
                        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                        "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                        "return nil; " +
                        "end; " +
                        // 如果key存在且是当前线程，锁次数+1 并重置过期时间，返回空值
                        "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                        "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                        "return nil; " +
                        "end; " +
                        // 加锁失败，返回过期时间
                        "return redis.call('pttl', KEYS[1]);",
                Collections.singletonList(getRawName()), unit.toMillis(leaseTime), getLockName(threadId));
    }
```

### 异步续时

```
    private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
        
        // 省略加锁代码
        
        ttlRemainingFuture.onComplete((ttlRemaining, e) -> {
            if (e != null) {
                return;
            }

            // lock acquired
            if (ttlRemaining == null) {
                if (leaseTime != -1) {
                    internalLockLeaseTime = unit.toMillis(leaseTime);
                } else {
                    // 获取到锁后执行续命操作
                    scheduleExpirationRenewal(threadId);
                }
            }
        });
        return ttlRemainingFuture;
    }
    
    protected void scheduleExpirationRenewal(long threadId) {
        ExpirationEntry entry = new ExpirationEntry();
        ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);
        if (oldEntry != null) {
            oldEntry.addThreadId(threadId);
        } else {
            entry.addThreadId(threadId);
            try {
                renewExpiration();
            } finally {
                if (Thread.currentThread().isInterrupted()) {
                    cancelExpirationRenewal(threadId);
                }
            }
        }
    }
    
        private void renewExpiration() {
        ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
        if (ee == null) {
            return;
        }
        
        Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) throws Exception {
                ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());
                if (ent == null) {
                    return;
                }
                Long threadId = ent.getFirstThreadId();
                if (threadId == null) {
                    return;
                }
                
                RFuture<Boolean> future = renewExpirationAsync(threadId);
                future.onComplete((res, e) -> {
                    if (e != null) {
                        log.error("Can't update lock " + getRawName() + " expiration", e);
                        EXPIRATION_RENEWAL_MAP.remove(getEntryName());
                        return;
                    }
                    
                    if (res) {
                        // 调用自身
                        renewExpiration();
                    } else {
                        cancelExpirationRenewal(null);
                    }
                });
            }
        // 每个 internalLockLeaseTime / 3 时间续时 
        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
        
        ee.setTimeout(task);
    }
```

### 阻塞

```
    /**
     * 加锁
     *
     * @param leaseTime     加锁到期时间, -1 使用默认值 30 秒
     * @param unit          时间单位
     * @param interruptibly 是否可被中断标识
     * @throws InterruptedException
     */
    private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
        long threadId = Thread.currentThread().getId();
        // 尝试加锁
        Long ttl = tryAcquire(-1, leaseTime, unit, threadId);
        // 加锁成功
        if (ttl == null) {
            return;
        }

        // 订阅分布式锁, 解锁时进行通知
        RFuture<RedissonLockEntry> future = subscribe(threadId);
        if (interruptibly) {
            commandExecutor.syncSubscriptionInterrupted(future);
        } else {
            commandExecutor.syncSubscription(future);
        }

        try {
            while (true) {
                // 再次尝试获取锁
                ttl = tryAcquire(-1, leaseTime, unit, threadId);
                // 加锁成功
                if (ttl == null) {
                    break;
                }

                // 锁过期时间如果大于零
                if (ttl >= 0) {
                    try {
                        // 阻塞，等待解锁时信号量通知
                        future.getNow().getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    } catch (InterruptedException e) {
                        if (interruptibly) {
                            throw e;
                        }
                        future.getNow().getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    }
                // 锁过期时间小于零
                } else {
                    if (interruptibly) {
                        future.getNow().getLatch().acquire();
                    } else {
                        future.getNow().getLatch().acquireUninterruptibly();
                    }
                }
            }
        } finally {
            unsubscribe(future, threadId);
        }
    }
```



## RedLock （红锁）

### Redisson锁的问题

在Redis主从的情况下，我们加锁是在加主库上，然后主库异步复制到从库。

但是如果发生了主从切换可能就存在锁丢失的风险：
1. 客户端加锁成功
2. 主库发生异常宕机，key未同步到从库
3. 从库被升级为新的主库，key丢失（锁丢失）

为此，Redis的作者提出的RedLock的解决方案

### RedLock的前提

Redlock 的方案基于 2 个前提：

1. 不再需要部署从库和哨兵实例，只部署主库
2. 但主库要部署多个，官方推荐至少 5 个实例

### RedLock的流程

1. 客户端先获取「当前时间戳T1」
2. 客户端依次向这 5 个 Redis 实例发起加锁请求，且每个请求会设置超时时间（毫秒级，要远小于锁的有效时间），如果某一个实例加锁失败（包括网络超时、锁被其它人持有等各种异常情况），就立即向下一个 Redis 实例申请加锁
3. 如果客户端从 >=3 个（半数以上） Redis 实例加锁成功，则再次获取「当前时间戳T2」，如果 T2 - T1 < 锁的过期时间，此时，认为客户端加锁成功，否则认为加锁失败
4. 加锁成功，去操作共享资源
5. 加锁失败，向「全部节点」发起释放锁请求


```
@Component
public class RedLock {

    @Autowired
    private Redisson redisson;

    public void lock() {
        RLock lock1 = redisson.getLock("lockName");
        RLock lock2 = redisson.getLock("lockName");
        RLock lock3 = redisson.getLock("lockName");
        RLock lock4 = redisson.getLock("lockName");
        RLock lock5 = redisson.getLock("lockName");
        RedissonRedLock redissonRedLock = new RedissonRedLock(lock1, lock2, lock3, lock4, lock5);
        redissonRedLock.lock();
    }
}
```

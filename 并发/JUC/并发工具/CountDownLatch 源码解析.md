#### CountDownLatch

在完成一组正在其他线程中执行的操作之前，它允许一个或多个线程一直等待

* 实际上是就是共享锁，内部静态类 Sync 继承了 AQS，实现了共享锁的 tryAcquireShared、tryReleaseShared
* 但是实现是反过来的
  * 在 tryAcquireShared 获取同步状态的时候， state > 0 返回 false，state == 0，返回 true，从而达到如果 count 没减到零，那么调用 await 的线程全部进 CLH 去挂起
  * 在 tryReleaseShared 释放同步状态的时候，state >  0 返回 false，state == 0 ，返回 true，从而达到调用 countDown 的线程已经够了，然后就会去 CLH 队列唤醒，因为是共享型的，所以唤醒一个之后，它会继续在 doAquireShared 的 setHeadAndProgate 中继续传播下去唤醒，从而所有 awite 的线程就可以继续执行了。



##### Sync

```java
private static final class Sync extends AbstractQueuedSynchronizer {
        
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        // 获取同步状态
        int getCount() {
            return getState();
        }

        // 获取同步状态
        @Override
        protected int tryAcquireShared(int acquires) {
            // 如果没有减为 0，那么进 CLH 等待
            return (getState() == 0) ? 1 : -1;
        }

        // 释放同步状态
        @Override
        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    // countDown 的线程数够了，那就返回 true，然后 doAquireShared 里 head 后第一个线程就可以执行了。
                    return nextc == 0;
            }
        }
    }
    
}
```



##### 方法

###### await

```java
public void await() throws InterruptedException {
    // 中断敏感
    sync.acquireSharedInterruptibly(1);
}

public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    // 中断敏感 + 超时
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```



###### countDown

```java
public void countDown() {
    sync.releaseShared(1);
}
```



##### 应用场景

一个等其他多个线程，或者多个等其他多个的场景



#### Semaphore

限制可以访问某些资源（物理或逻辑的）的线程数目

* 实际上是就是普通共享锁，内部静态类 Sync 继承了 AQS，实现了共享锁的 tryAcquireShared、tryReleaseShared。

* 逻辑就是正常共享锁逻辑，同时还可以实现公平和非公平
* 可以说是 ReentrantLock 的共享版，所以注意**要在 finally 里面调用 semaphore.release() 释放锁**

##### Sync 抽象类

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }

    	// 父类提供了非公平的 tryAcquireShared 方法
        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

    	// 和 tryReleaseShared 一个效果，但是不会唤醒 CLH 的线程，不过下一次 tryReleaseShared 唤醒后就可以用这个释放的。
        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

    	// 重置为 0，但是就算重置为 0，后面通过 aquirceShared 也就能进 permits 个
        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
}
```



##### Sync 实现类

* NonfairSync

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -2694183684443567898L;

    NonfairSync(int permits) {
        super(permits);
    }
       
    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}
```

* FairSync

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = 2014338818796000944L;

    FairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        for (;;) {
            if (hasQueuedPredecessors()) // 多了个判断 CLH 队列还有没有等待的节点
                return -1;
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
```



##### 方法

###### 构造方法

```java
public Semaphore(int permits) {
    // 默认选择非公平锁
    sync = new NonfairSync(permits);
}

public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```



###### acquire

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```



###### release

```java
public void release() {
    sync.releaseShared(1);
}
```



###### 应用场景

就是限制访问同步块的线程数

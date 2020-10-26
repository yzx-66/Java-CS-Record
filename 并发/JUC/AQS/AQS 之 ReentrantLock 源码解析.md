#### ReentrantLock

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015094249902.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)

* ReentrantLock 还提供了**公平锁**和**非公平锁**的选择，通过构造方法接受一个可选的 `fair` 参数（默认非公平锁）：当设置为 true 时，表示公平锁；否则为非公平锁。
  * 非公平的原因是，在 AQS 的 acquire 方法中，如果 tryAcquire 成功了那么直接获得同步状态，不用加入 CLH 队列等待，因此非公平。



##### Sync 抽象类

*  ReentrantLock 的内部静态类，实现 AbstractQueuedSynchronizer 抽象类。



###### 已实现方法

* 非公平获取锁

```java
final boolean nonfairTryAcquire(int acquires) {
    //当前线程
    final Thread current = Thread.currentThread();
    //获取同步状态
    int c = getState();
    //state == 0,表示没有该锁处于空闲状态
    if (c == 0) {
        //获取锁成功，设置为当前线程所有
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //线程重入
    //判断锁持有的线程是否为当前线程
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```



* 释放同步状态

```java
protected final boolean tryRelease(int releases) {
    // 减掉releases
    int c = getState() - releases;
    // 如果释放的不是持有锁的线程，抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    // state == 0 表示已经释放完全了，其他线程可以获取同步状态了
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

* 其他实现方法

```java
// 是否当前线程独占
@Override
protected final boolean isHeldExclusively() {
    return getExclusiveOwnerThread() == Thread.currentThread();
}

// 新生成条件
final ConditionObject newCondition() {
    return new ConditionObject();
}

// 获得占用同步状态的线程
final Thread getOwner() {
    return getState() == 0 ? null : getExclusiveOwnerThread();
}

// 获得当前线程持有锁的数量
final int getHoldCount() {
    return isHeldExclusively() ? getState() : 0;
}

// 是否被锁定
final boolean isLocked() {
    return getState() != 0;
}

/**
 * Reconstitutes the instance from a stream (that is, deserializes it).
 * 自定义反序列化逻辑
 */
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    s.defaultReadObject();
    setState(0); // reset to unlocked state
}
```





###### 未实现方法

```java
/**
 * Performs {@link Lock#lock}. The main reason for subclassing
 * is to allow fast path for nonfair version.
 */
abstract void lock();
```



##### Sync 实现类

###### NonfairSync

* lock

```java
@Override
final void lock() {
    // 非公平锁的特点:此时有可能有 N + 1 个线程正在获得锁，其中 1 个线程在获得到锁后的释放瞬间，恰好被新的线程抢夺到，而不是排队的 N 个线程。
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```



* tryAcquire

```java
protected final boolean tryAcquire(int acquires) {
    // 非公平的方式
    return nonfairTryAcquire(acquires);
}
```



###### FairSync

* lock

````java
final void lock() {
    // 直接执行 AQS 的正常的同步状态获取逻辑。
    acquire(1);
}
````



* tryAcquire

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() && // 与非公平锁唯一的区别，在可以获取同步状态时，判断是否有前序节点
                compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

/**
 * 注意：这个是 AQS 的方法
 */
public final boolean hasQueuedPredecessors() {
    Node t = tail;  //尾节点
    Node h = head;  //头节点
    Node s;

    //头节点 != 尾节点
    //同步队列第一个节点不为null
    /**
     * 为什么是 head.next:因为 head 势必是当前正在执行的线程，那么 next 必然是下一个被唤醒去获得锁的线程，那么如果存在 next，就不可以插队去tryaquire
     */
    return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

##### 实现

###### Lock 接口

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015094230264.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




###### Lock 实现

* 构造方法

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```



* lock

```java
@Override
public void lock() {
    sync.lock();
}
```



* tryLock

```java
@Override
public boolean tryLock() {
    /**
     * tryLock() 在实现时，希望能快速的获得是否能够获得到锁，因此即使在设置为 fair = true ( 使用公平锁 )，依然调用 Sync#nonfairTryAcquire。
     * 如果真的希望 #tryLock() 还是按照是否公平锁的方式来，可以调用 #tryLock(0, TimeUnit) 方法来实现。
     *
     * 补充说明：即使公平锁因为 tryLock 非公平的竞争到了锁也没有影响，因为等待的最终目的也是获得同步状态
     */
    return sync.nonfairTryAcquire(1);
}
```



* tryLock

```java
@Override
public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
```



* unlock

```java
@Override
public void unlock() {
    sync.release(1);
}
```



* newCondition

```java
@Override
public Condition newCondition() {
    return sync.newCondition();
}
```

* 其他方法

```java
public int getHoldCount() {
    return sync.getHoldCount();
}
public boolean isHeldByCurrentThread() {
    return sync.isHeldExclusively();
}
public boolean isLocked() {
    return sync.isLocked();
}


public final boolean isFair() {
    return sync instanceof FairSync;
}

protected Thread getOwner() {
    return sync.getOwner();
}
public final boolean hasQueuedThreads() {
    return sync.hasQueuedThreads();
}
public final boolean hasQueuedThread(Thread thread) {
    return sync.isQueued(thread);
}
public final int getQueueLength() {
    return sync.getQueueLength();
}
protected Collection<Thread> getQueuedThreads() {
    return sync.getQueuedThreads();
}

public boolean hasWaiters(Condition condition) {
    if (condition == null)
        throw new NullPointerException();
    if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
        throw new IllegalArgumentException("not owner");
    return sync.hasWaiters((AbstractQueuedSynchronizer.ConditionObject)condition);
}

public int getWaitQueueLength(Condition condition) {
    if (condition == null)
        throw new NullPointerException();
    if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
        throw new IllegalArgumentException("not owner");
    return sync.getWaitQueueLength((AbstractQueuedSynchronizer.ConditionObject)condition);
}

protected Collection<Thread> getWaitingThreads(Condition condition) {
    if (condition == null)
        throw new NullPointerException();
    if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
        throw new IllegalArgumentException("not owner");
    return sync.getWaitingThreads((AbstractQueuedSynchronizer.ConditionObject)condition);
}
```



###### 对比 synchronized 

首先他们肯定具有相同的功能和内存语义

* synchronized 会在释放锁把共享变量刷到主内存，获取锁时会让本地内存失效
* volatile 是根据 happens-before 的 程序次序规则 +  volatile 规则 + 传递规则 ，保证同步代码块里的共享变量一定每次都能看到最新的

再说性能差别

* synchronized 在 jdk 1.6 已经做了底层的优化，性能已经和 Lock 在多线程时相当，都是先 cas，不成功再加入阻塞队列再等待唤醒。

区别

* Synchronized是 JVM 层次的锁实现，ReentrantLock 是 JDK 层次的锁实现；

* Synchronized是非公平锁，ReentrantLock是可以是公平也可以是非公平的

  * Synchronized在特定的情况下**对于已经在等待的线程**是后来的线程先获得锁；
  * 而ReentrantLock对于**已经在等待的线程**一定是先来的线程先获得锁；
* 在同步块里发生异常时，Synchronized会自动释放锁（由javac编译时自动实现），而ReentrantLock需要开发者在finally块中显示释放锁；
* Lock 的功能更丰富
  * Synchronized 是对中断是不敏感的，而`ReentrantLock#lockInterruptibly`方法是可以对中断敏感的；
  * Synchronized 是不可以设置等待超时时间的，而`ReentrantLock#tryAcquireNanos`方法可以；
  * Synchronized 不可以只尝试获取一次锁，而`ReentrantLock#tryLock` 方法可以一次获取锁失败就返回
  * Synchronized 的锁状态是无法在代码中直接判断的，而 `ReentrantLock#isLocked` 可以判断；
  * Synchronized 的 wait 队列一个 waitset，而 ReentrantLock 可以获得多个 Condition，每个 Condition 都有自己的等待队列
  * Synchrnoized 的等待唤醒，比 Condition 灵活性差





#### ReentrantReadWriteLock

* **主要特性**

  公平性：支持公平性和非公平性。

  重入性：支持重入。读写锁最多支持 65535 个递归写入锁和 65535 个递归读取锁。

  锁降级：遵循获取**写**锁，再获取**读**锁，最后释放**写**锁的次序，如此写锁能够**降级**成为读锁。



![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015094340925.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



* 读锁、写锁都是通唯一的一个 Sync 来实现的。所以 ReentrantReadWriteLock 实际上只有一个锁（都是通过 reentrantReadWriteLock.sync 获取 sync 对象作为自己的锁）。
* 读锁、写锁只是调用的 sync 对象的方法不同，读锁调用的共享那些方法，写锁调用的独占那些方法



##### Sync 抽象类

###### 读、写状态记录

* 状态 c 的 高 16 位存储读状态，低 16 位存储写状态。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/2020101509435899.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


````java
static final int SHARED_SHIFT   = 16; // 位数
static final int SHARED_UNIT    = (1 << SHARED_SHIFT); // 0000 0000 0000 0001 0000 0000 0000 0000（即：高 16 加 1 时的 1）
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1; // 每个锁的最大重入次数，65535
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1; // 0000 0000 0000 0000 1111 1111 1111 1111（即：按位与后，高 16 位失效）

// 获得读锁的数量，右移 16 位后，低 16 位（写锁部分）失效
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
// 获得写锁的重入次数，与 EXCLUSIVE_MASK 按位与后，高 16 位失效
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
````



###### HoldCounter

* 由于高 16 位的读状态只记录了获得读锁的总数量，所以当前线程对读锁的重入次数就用  ThreadLocalHoldCounter 记录（当前线程对 ThreadLocalHoldCounter 进行 get() 时，就会获得当前线程对应的 HoldCounter）

```java
static final class HoldCounter {
    // 当前线程重入次数计数器
    int count = 0;
    // 绑定线程的 id，为什么不和 Thread 直接绑定，因为为了垃圾回收
    final long tid = getThreadId(Thread.currentThread());
}

// 为什么不直接使用 thread.getId()，因为防止 getId() 被重写，获取到错误的 id
static final long getThreadId(Thread thread) {
    return UNSAFE.getLongVolatile(thread, TID_OFFSET);
}

// threadLocal 的子类，每个线程在 get 和 set 时，会操作自己线程对应的
static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
    // 该方法是重写 ThreadLocal 的，当 get() 到的值为空时，会返回该方法的返回值。
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}

// 为了使用 threadLocal 获得当前线程绑定的 HoldCounter
private transient ThreadLocalHoldCounter readHolds;

/**
 * 下面三个属性都是为了性能进行的缓存，因为 ThreadLocal 进行 get 时还要再计算当前线程 hash 然后到其内部的 map 查找，是个耗时操作
 */
// 最后一个获取锁的线程的 HoldCounter 缓存，这个会一直随着线程访问改变
private transient HoldCounter cachedHoldCounter;
// 第一个获取锁的线程的缓存，这个确定后不会改变
private transient Thread firstReader = null;
// 第一个获取锁的线程的重入次数，这个只会是第一个获取锁的线程的。
private transient int firstReaderHoldCount;
```



###### 读、写锁通用方法

* 构造方法

```java
Sync() {
    readHolds = new ThreadLocalHoldCounter();
    setState(getState()); // AQS 的 state
}
```



* isHeldExclusively

```java
protected final boolean isHeldExclusively() {
    // While we must in general read state before owner,
    // we don't need to do so to check if current thread is owner
    return getExclusiveOwnerThread() == Thread.currentThread();
}
```

* newCondition

```java
final ConditionObject newCondition() {
    return new ConditionObject();
}
```



* 其他实现方法

```java
final Thread getOwner() {
    // Must read state before owner to ensure memory consistency
    return ((exclusiveCount(getState()) == 0) ?
            null :
            getExclusiveOwnerThread());
}

final int getReadLockCount() {
    return sharedCount(getState());
}

final boolean isWriteLocked() {
    return exclusiveCount(getState()) != 0;
}

final int getWriteHoldCount() {
    return isHeldExclusively() ? exclusiveCount(getState()) : 0;
}

final int getReadHoldCount() {
    if (getReadLockCount() == 0)
        return 0;

    Thread current = Thread.currentThread();
    if (firstReader == current)
        return firstReaderHoldCount;

    HoldCounter rh = cachedHoldCounter;
    if (rh != null && rh.tid == getThreadId(current))
        return rh.count;

    int count = readHolds.get().count;
    if (count == 0) readHolds.remove();
    return count;
}

/**
 * Reconstitutes the instance from a stream (that is, deserializes it).
 */
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    s.defaultReadObject();
    readHolds = new ThreadLocalHoldCounter();
    setState(0); // reset to unlocked state
}

final int getCount() { return getState(); }
```



###### 写锁使用方法

* tryAcquire

```java
@Override
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    // 当前状态（如果存在读锁或者写锁，那么 c 肯定不为 0）
    int c = getState();
    // 写锁的重入次数（不存在写锁为 0）
    int w = exclusiveCount(c);
    if (c != 0) {
        //c != 0 && w == 0 表示存在读锁
        //当前线程不是已经获取写锁的线程
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        //超出最大范围
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        setState(c + acquires);
        return true;
    }
    // 是否需要阻塞
    if (writerShouldBlock()/**该方法公平和非公平实现不同**/ || 
            !compareAndSetState(c, c + acquires))
        return false;
    //设置获取锁的线程为当前线程
    setExclusiveOwnerThread(current);
    return true;
}
```

* tryRealse

```java
protected final boolean tryRelease(int releases) {
    //释放的线程不为锁的持有者
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    //若写锁的新线程数为0，则将锁的持有者设置为null
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```



* tryWriteLock（子类不重写，都是非公平，不排队）

```java
final boolean tryWriteLock(){
    Thread current = Thread.currentThread();
    int c = getState();
    if(c != 0){
        int w = exclusiveCount(c); // 获得现在写锁获取的数量
        if(w == 0 || current != getExclusiveOwnerThread()){  // 判断是否是其他的线程获取了写锁。若是，返回 false
            return false;
        }
        if(w == MAX_COUNT){ // 超过写锁上限，抛出 Error 错误
            throw new Error("Maximum lock count exceeded");
        }
    }

    if(!compareAndSetState(c, c + 1)){ //  CAS 设置同步状态，尝试获取写锁。若失败，返回 false，不调用父类 AQS 加入队列
        return false;
    }
    setExclusiveOwnerThread(current); // 设置持有写锁为当前线程
    return true;
}
```



###### 读锁使用方法

* tryAcquireShared

```java
protected final int tryAcquireShared(int unused) {
    //当前线程
    Thread current = Thread.currentThread();
    int c = getState();
    // exclusiveCount(c)计算写锁
    // 如果存在写锁，且锁的持有者不是当前线程，直接返回-1
	// 如果是当前线程的话，说明是锁降级（写锁里面获取读锁）
    if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
        return -1;
    
    // 获取读锁的线程数
    int r = sharedCount(c);

    /*
     * readerShouldBlock():读锁是否需要等待（公平锁原则）
     * r < MAX_COUNT：持有线程小于最大数（65535）
     * compareAndSetState(c, c + SHARED_UNIT)：设置读取锁状态
     */
    if (!readerShouldBlock() &&
            r < MAX_COUNT &&
            compareAndSetState(c, c + SHARED_UNIT)) { //修改高16位的状态，所以要加上2^16
        // 如果是第一个获取读锁的线程
        if (r == 0) {
            // 就让 first 缓存当前线程的信息
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) { // 如果当前线程是第一个线程
            firstReaderHoldCount++; // 就让缓存的第一个线程重入数加一
        } else {
            HoldCounter rh = cachedHoldCounter; // 获得最后一个获取读锁的线程 Counter 缓存
            if (rh == null || rh.tid != getThreadId(current)) // 如果 rh 不是当前线程
                cachedHoldCounter = rh = readHolds.get(); // 那就让 cachedHoldCounter 和 rh 都指向当前线程的 counter
            else if (rh.count == 0) // 如果 cachedHoldCounter 的是当前线程的，但是重入数已经为 0
                readHolds.set(rh);  
            rh.count++; // 当前线程重入数加一
        }
        return 1;
    }
    // 没有设置 cas 成功 或者 要 block，执行下面的方法
    return fullTryAcquireShared(current);
}


final int fullTryAcquireShared(Thread current) {
   HoldCounter rh = null;
   // 自旋，确保 cas 更新 state 成功
   for (;;) {
       int c = getState();
       // 有写锁，但不是锁降级，则返回 -1
       if (exclusiveCount(c) != 0) {
           if (getExclusiveOwnerThread() != current)
               return -1;
       }
       // 没有写锁，但读锁需要阻塞，那么失败前再判断下是否当前线程已经获取到读锁
       else if (readerShouldBlock()) {
           //列头为当前线程，那么肯定获得了锁，继续往下执行 cas 设置 state
           if (firstReader == current) {
           }
           else {
               if (rh == null) {
                   rh = cachedHoldCounter;
                   if (rh == null || rh.tid != getThreadId(current)) {
                       rh = readHolds.get(); // 获得当前现成的 counter
                       if (rh.count == 0) // 计数为 0 ，说明没得到读锁，清空线程变量
                           readHolds.remove();
                   }
               }
               if (rh.count == 0) // 说明获取还要等待一段时间，且没得到读锁，那么返回获取失败，然后加入队列即可
                   return -1;
           }
       }
       //读锁超出最大范围
       if (sharedCount(c) == MAX_COUNT)
           throw new Error("Maximum lock count exceeded");
       // CAS 把读锁线程数加一成功
       if (compareAndSetState(c, c + SHARED_UNIT)) { //修改高16位的状态，所以要加上2^16  
           //如果是第1次获取“读取锁”，则更新firstReader和firstReaderHoldCount
           if (sharedCount(c) == 0) {
               firstReader = current;
               firstReaderHoldCount = 1;
           }
           //如果想要获取锁的线程 ( current ) 是第 1 个获取锁 ( firstReader ) 的线程,则将 firstReaderHoldCount + 1
           else if (firstReader == current) {
               firstReaderHoldCount++;
           } else {
               if (rh == null) 
                   rh = cachedHoldCounter;
               if (rh == null || rh.tid != getThreadId(current))
                   rh = readHolds.get(); // 更新 rh
               else if (rh.count == 0)
                   readHolds.set(rh);
               //更新线程的获取“读取锁”的共享计数
               rh.count++;
               cachedHoldCounter = rh; // 更新 cachedHoldCounter
           }
           return 1;
       }
   }
}
```

* tryRealseShared

```java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    //如果想要释放锁的线程为第一个获取锁的线程
    if (firstReader == current) {
        //仅获取了一次，则需要将firstReader 设置null，否则 firstReaderHoldCount - 1
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    }
    //获取rh对象，并更新“当前线程获取锁的信息”
    else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    //CAS更新同步状态
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

* tryReadLock（子类不重写，都是非公平，不排队）

```java
final boolean tryReadLock() {
    Thread current = Thread.currentThread();
    for (;;) {
        int c = getState();
       //exclusiveCount(c)计算写锁
        //如果存在写锁，且锁的持有者不是当前线程，直接返回-1
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return false;
        // 读锁线程数
        int r = sharedCount(c);
        
        if (r == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // cas 新增读锁线程数成功
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (r == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                // 当前线程读锁重入数 + 1
                rh.count++;
            }
            return true;
        }
    }
}
```



###### 未实现方法

* writerShouldBlock

```java
abstract boolean writerShouldBlock(); // 获取写锁时，如果有前序节点也获得锁时，是否阻塞
```

* readerShouldBlock

```java
abstract boolean readerShouldBlock(); // 获取读锁时，如果有前序节点也获得锁时，是否阻塞。
```

##### Sync 实现类

###### NonfairSync

* writerShouldBlock

```java
@Override
final boolean writerShouldBlock() {
    // 非公平的写锁获取前不用判断是不是要等待前面的节点
    return false; // writers can always barge
}
```



* readerShouldBlock

```java
@Override
final boolean readerShouldBlock() {
   // 判断是否当前写锁已经被获取
   return apparentlyFirstQueuedIsExclusive();
}
    

/**
 * 注意：AQS 的 #apparentlyFirstQueuedIsExclusive() 方法
 */
final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&
        (s = h.next)  != null &&
        /** 
         * 比 reentrantLock 的非公平锁多考虑了下面这一点，
         * head 正在执行，但 head 后面紧跟的是要获取写锁的节点，那么就必须等写锁获取完
         * 因此，如果 head 的下一个是读请求的节点，那么可以让获取读锁的线程无限加塞
         */
        !s.isShared()         && 
        s.thread != null;
}
```



###### FairSync

* writerShouldBloc

```java
@Override
final boolean writerShouldBlock() {
    return hasQueuedPredecessors();
}

/**
 * 注意：AQS 的 #hasQueuedPredecessors
 */
public final boolean hasQueuedPredecessors() {
    Node t = tail;  //尾节点
    Node h = head;  //头节点
    Node s;

    //头节点 != 尾节点
    //同步队列第一个节点不为null
    /**
     * 和 apparentlyFirstQueuedIsExclusive 的区别是，
     * 不管 head 后紧跟的下一个要被唤醒的节点是不是共享节点，都不可以加塞，只要 head 后面有排队的，那么就要等待
     */
    return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

* readerShouldBlock

```java
@Override
final boolean readerShouldBlock() {
    return hasQueuedPredecessors();
}
```



##### WriteLock

* 构造方法

```java
private final Sync sync;

protected ReadLock(ReentrantReadWriteLock lock) {
    sync = lock.sync;
}
```



* lock

```java
@Override
public void lock() {
    // 独占式
    sync.acquire(1);
}
```



* lockInterruptibly

```java
@Override
public void lockInterruptibly() throws InterruptedException {
    // 独占式
    sync.acquireInterruptibly(1);
}
```

* tryLock

```java
@Override
 public boolean tryLock( ) {
    // 希望能快速的获得是否能够获得到锁，因此即使在设置为 fair = true (使用公平锁)，依然调用 Sync#tryWriteLock() 方法（不排队）。
    // 真的希望 #tryLock() 还是按照是否公平锁的方式来，可以调用 #tryLock(0, TimeUnit) 方法来实现
    return sync.tryWriteLock();
}
```

* tryLock

```java
@Override 
public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
```

* unlock

```java
@Override
public void unlock() {
    // 独占或
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



* isHeldByCurrentThread

```java
public boolean isHeldByCurrentThread() {
    return sync.isHeldExclusively();
}
```

* getHoldCount

```java
public int getHoldCount() {
    return sync.getWriteHoldCount();
}
```



##### ReadLock

* 构造方法

```java
private final Sync sync;

protected ReadLock(ReentrantReadWriteLock lock) {
    sync = lock.sync;
}
```

* lock

```java
@Override
public void lock() {
    // 共享式
    sync.acquireShared(1);
}
```

* lockInterruptibly

```java
@Override
public void lockInterruptibly() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

* tryLock

```java
@Override
public boolean tryLock() {
    // 不排队
    return sync.tryReadLock();
}
```

* tryLock

```java
@Override
public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
    // 共享式
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

* unlock

```java
@Override
public void unlock() {
    // 共享式
    sync.releaseShared(1);
}
```

* newCondition

```java
@Override
public Condition newCondition() {
    // 
    throw new UnsupportedOperationException();
}
```



##### 实现

###### ReadWriteLock 父类

```java
Lock readLock();

Lock writeLock();
```



###### 父类实现

```java
// 内部类  读锁 
private final ReentrantReadWriteLock.ReadLock readerLock;
// 内部类  写锁 
private final ReentrantReadWriteLock.WriteLock writerLock;

final Sync sync;

// 使用默认（非公平）的排序属性创建一个新的 ReentrantReadWriteLock 
public ReentrantReadWriteLock() {
    this(false);
}

// 使用给定的公平策略创建一个新的 ReentrantReadWriteLock 
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    // 读锁、写锁都是通唯一的一个 Sync 来实现的。所以 ReentrantReadWriteLock 实际上只有一个锁
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}

// 返回写锁 
@Override
public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }

// 返回读锁 
@Override
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```


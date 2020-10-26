#### Condition

##### Condition 接口

```java
// ========== 阻塞 ==========

// 造成当前线程在接到信号或被中断之前一直处于等待状态。
void await() throws InterruptedException; 

// 造成当前线程在接到信号之前一直处于等待状态。【注意：该方法对中断不敏感】。
void awaitUninterruptibly(); 

// 造成当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。
// 返回值表示剩余时间，如果在`nanosTimeout` 之前唤醒，那么返回值 `= nanosTimeout - 消耗时间` ，如果返回值 `<= 0` ,则可以认定它已经超时了。
long awaitNanos(long nanosTimeout) throws InterruptedException; 

// 造成当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。
// 如果没有到指定时间就被通知，则返回 true ，否则表示到了指定时间，返回返回 false 。
boolean await(long time, TimeUnit unit) throws InterruptedException; 

// 造成当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态。
// 如果没有到指定时间就被通知，则返回 true ，否则表示到了指定时间，返回返回 false 。
boolean awaitUntil(Date deadline) throws InterruptedException; 


// ========== 唤醒 ==========

// 唤醒一个等待线程。该线程从等待方法返回前必须获得与Condition相关的锁。
void signal(); 

// 唤醒所有等待线程。能够从等待方法返回的线程必须获得与Condition相关的锁。
void signalAll(); 
```

和 synchronized 的 await / notify 对比

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015094455110.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




#####  ConditionObject

是 AQS 的内部类，因为需要使用到 AQS 的 CHL 队列 与相关操作

* `#await()` 就是在当前线程持有锁的基础上释放锁资源，并新建 Condition 节点加入到 Condition 的队列尾部，阻塞当前线程 。
* `#signal()` 就是将 Condition 的头节点移动到 AQS 等待节点尾部，让其等待再次获取锁。
* 和 CHL 队列异同
  * Node 定义与 AQS 的 CLH 同步队列的节点使用的都是同一个类（AbstractQueuedSynchronized#Node 静态内部类）。
  * Node 的 Mode 都是 Node.CONDITION
  * 是单向链表，通过 nextWaiter 属性连接下一个节点。

```java
// 注意：不是静态的，所以有 AQS 的引用
public class ConditionObject implements Condition, java.io.Serializable {
    /** First node of condition queue. */
    private transient Node firstWaiter; // 头节点
    /** Last node of condition queue. */
    private transient Node lastWaiter; // 尾节点
    
    public ConditionObject() {
    }

    // ... 省略内部代码
}
```



###### await

注意：中断敏感

```java
/**
 * 使当前线程进入等待状态，同时会加入到 Condition 等待队列，并且同时释放锁。
 * 当从 #await() 方法结束时，当前线程一定是重新在 CLH 队列排队，然后获得了同步状态
 */
public final void await() throws InterruptedException {
    // 当前线程中断
    if (Thread.interrupted())
        throw new InterruptedException();
    //当前线程加入等待队列
    Node node = addConditionWaiter();
    //释放锁
    long savedState = fullyRelease(node);
    int interruptMode = 0;
    /**
     * 检测此节点的线程是否在同步队上，如果不在，则说明该线程还不具备竞争锁的资格，则继续等待直到检测到此节点在同步队列上。
     * 放在 while 循环主要是防止意外唤醒。
     */
    while (!isOnSyncQueue(node)) {
        //线程挂起，等待 unpark 后继续执行。
        // unpark 会发生在重新入队 CLH 后，排队轮到它后，被头节点唤醒
        LockSupport.park(this);
        //如果已经中断了，则退出
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //竞争同步状态
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    // 清理下条件队列中的不是在等待条件的节点
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        // 中断敏感，抛异常
        reportInterruptAfterWait(interruptMode);
}

/**
 * 加入 condition 队列
 */
private Node addConditionWaiter() {
    Node t = lastWaiter;    //尾节点
    // Node 的节点状态如果不为 CONDITION ，则表示该节点不处于等待状态，需要清除节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        //清除条件队列中所有状态不为Condition的节点
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    
    // 为当前线程新建节点，状态 CONDITION
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    
    if (t == null)
        firstWaiter = node;
    else
        // 注意：使用的 nextWaiter 属性进行连接
        t.nextWaiter = node;
    // 将该节点加入到条件队列中最后一个位置
    lastWaiter = node;
    return node;
}

/**
 * 完全释放该线程持有的锁，重入的也要释放完，不然tryRelease() 返回 false 的话就没法从 CLH 队列移除
 * 这里重入的指的 ReentrantLock 和 WriteLock，因为 ReadLock 不支持 Condition
 */
final long fullyRelease(Node node) {
    boolean failed = true;
    try {
        // 节点状态--其实就是持有锁的数量
        long savedState = getState();
        // 释放锁（把重入的次数全部释放掉，然后从 CLH 队列移除）
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            // 状态变为 CANCLE 之后会被其后面的节点移除，或者被前面的节点唤醒了其后的节点导致 cancle 节点移除
            node.waitStatus = Node.CANCELLED;
    }
}

/**
 * 如果一个节点回到同步队列上了，那么返回 true 
 */
final boolean isOnSyncQueue(Node node) {
    // 状态为 Condition，或者前驱节点为 null（因为 clh 队列的每个节点的 prev 肯定不为 null） ，返回 false
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    // 后继节点不为 null，肯定在 CLH 同步队列中
    if (node.next != null)
        return true;

    // 遍历 CLH 队列
    return findNodeFromTail(node);
}
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
       if (t == node)
           return true;
       if (t == null)
           return false;
       // 遍历 clh 队列
       // 用 prev 指针遍历是为了防止 cancelAcquire() 中的 node.next = node.
       t = t.prev;
    }
}

/**
 * 将条件队列中状态不为 Condition 的节点删除（单链表删除操作）
 */
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null; // 用于中间不需要跳过时，记录上一个 Node 节点
    while (t != null) {
        Node next = t.nextWaiter;
        // 如果节点的状态不是 Node.CONDITION 的话，这个节点就是被取消的
        if (t.waitStatus != Node.CONDITION) {
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
```



###### signal

```java
/** 
 * 唤醒在等待队列中等待最长时间的节点（条件队列里的首节点），在唤醒节点前，会将节点移到CLH同步队列中。
 */
public final void signal() {
    // 检查要去执行唤醒操作的线程是否是占有锁的线程（lock 自己实现）
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    //头节点，唤醒条件队列中的第一个节点
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);    //唤醒
}

private void doSignal(Node first) {
    do {
        //修改头结点，完成旧头结点的移出工作
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
        // 如果加入CLH 队列失败，那么继续尝试下一个节点
        // 失败是因为节点不是 Node.CONDITION 状态了，那肯定是 cancle 状态，因为 conditionObject 只在 fullyRelease 把是失败的节点修改为 cancle
    } while (!transferForSignal(first) &&   
            (first = firstWaiter) != null);
}

/**
 * 将节点移动到 CLH 同步队列中
 */
final boolean transferForSignal(Node node) {
   // 将该节点从状态 CONDITION 改变为初始状态 0,
   if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
       return false;

   // 将节点加入到 syn 队列中去，返回的是 syn 队列中 node 节点前面的一个节点
   Node p = enq(node);
   int ws = p.waitStatus;
   // 如果结点p的状态为 cancel 或者修改 waitStatus 失败，则直接唤醒
   // 如果在这唤醒，那么被唤醒线程已经在 clh 队列了，会继续在 await 方法执行 aquireQueue(),在这个方法会继续其修改前面节点的状态
   if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
       LockSupport.unpark(node.thread);
   return true;
}
```



##### 示例

```java
public class ConditionTest {
    private LinkedList<String> buffer;    //容器
    private int maxSize ;           //容器最大
    private Lock lock;
    private Condition notFull;
    private Condition notEmpty;

    ConditionTest(int maxSize){
        this.maxSize = maxSize;
        buffer = new LinkedList<String>();
        lock = new ReentrantLock();
        notFull = lock.newCondition();
        notEmpty = lock.newCondition();
    }

    public void set(String string) throws InterruptedException {
        lock.lock();    //获取锁
        try {
            while (maxSize == buffer.size()){
                notFull.await();       //满了，添加的线程进入等待状态
            }

            buffer.add(string);
            notEmpty.signal();
        } finally {
            lock.unlock();      //释放锁
        }
    }

    public String get() throws InterruptedException {
        String string;
        lock.lock();
        try {
            while (buffer.size() == 0){
                notEmpty.await();
            }
            string = buffer.poll();
            notFull.signal();
        } finally {
            lock.unlock();
        }
        return string;
    }
}

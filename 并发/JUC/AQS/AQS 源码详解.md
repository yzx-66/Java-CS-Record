### AQS

* AbstractQueuedSynchronizer ，即**队列同步器**。 
* AQS 使用一个 `int` 类型的成员变量 `state` 来**表示同步状态**：当 `state > 0` 时，表示已经获取了锁；当 `state = 0` 时，表示释放了锁。
* 通过内置的 FIFO 同步队列 CLH 来完成资源获取线程的**排队工作**。



#### 源码

##### 主要方法

1、状态相关操作

- `#getState()`：返回同步状态的当前值。
- 【可重写】`#isHeldExclusively()`：当前同步器是否在独占式模式下被线程占用，一般该方法表示是否被当前线程所独占。
- `#setState(int newState)`：设置当前同步状态。
- `#compareAndSetState(int expect, int update)`：使用 CAS 设置当前状态，该方法能够保证状态设置的原子性。



2、**独占式**获取与释放同步状态

- `acquire(int arg)`：独占式获取同步状态。如果当前线程获取同步状态成功，则由该方法返回；否则，将会进入同步队列等待。该方法将会调用**可重写**的 `#tryAcquire(int arg)` 方法；
- `#acquireInterruptibly(int arg)`：与 `#acquire(int arg)` 相同，但是该方法响应中断。当前线程为获取到同步状态而进入到同步队列中，如果当前线程被中断，则该方法会抛出InterruptedException 异常并返回。
- 【可重写】`#tryAcquire(int arg)`：独占式获取同步状态，获取同步状态成功后，其他线程需要等待该线程释放同步状态才能获取同步状态。
- `#tryAcquireNanos(int arg, long nanos)`：超时获取同步状态。如果当前线程在 nanos 时间内没有获取到同步状态，那么将会返回 false ，已经获取则返回 true 。
- `#release(int arg)`：独占式释放同步状态，该方法会在释放同步状态之后，将同步队列中第一个节点包含的线程唤醒。
- 【可重写】`#tryRelease(int arg)`：独占式释放同步状态。



3、**共享式**获取与释放同步状态

- `#acquireShared(int arg)`：共享式获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式的主要区别是在同一时刻可以有多个线程获取到同步状态；
- `#acquireSharedInterruptibly(int arg)`：共享式获取同步状态，响应中断。
- 【可重写】`#tryAcquireShared(int arg)`：共享式获取同步状态，返回值大于等于 0 ，则表示获取成功；否则，获取失败。
- `#tryAcquireSharedNanos(int arg, long nanosTimeout)`：共享式获取同步状态，增加超时限制。
- `#releaseShared(int arg)`：共享式释放同步状态
- 【可重写】`#tryReleaseShared(int arg)`：共享式释放同步状态。



##### CLH 队列

###### Node

```java
static final class Node {

    // 共享状态标志
    static final Node SHARED = new Node();
    // 独占状态标志
    static final Node EXCLUSIVE = null;

    // 因为超时或者中断，节点会被设置为取消状态，被取消的节点时不会参与到竞争中的，他会一直保持取消状态不会转变为其他状态，等待其他节点发现时将其剔除
    static final int CANCELLED =  1;
    
    // 后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行
    static final int SIGNAL    = -1;
    
    // 节点在等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()后，该节点将会从等待队列中转移到同步队列中
    static final int CONDITION = -2;
    
    // 只在共享模式下使用，表示下一次共享式同步状态获取，将会无条件地传播下去
    static final int PROPAGATE = -3;

    /** 
     * 等待状态，初始状态为 0 
     * 该状态不仅仅指的是 Node 自己的线程的等待状态，也可以是下一个节点的线程的等待状态。
     */
    volatile int waitStatus;

    /** 前驱节点，当节点添加到同步队列时被设置（尾部添加） */
    volatile Node prev;

    /** 后继节点 */
    volatile Node next;

    /** 节点类型（独占和共享） */
    Node nextWaiter;
    
    /** 获取同步状态的线程 */
    volatile Thread thread;

    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() { // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) { // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
    
}
```



###### 入队

AbstractQueuedSynchronizer#addWaiter()

* `tail` 指向新节点。
* 新节点的 `prev` 指向当前最后的节点。
* 当前最后一个节点的 `next` 指向新节点

```java
private Node addWaiter(Node mode) {
      // 新建节点
      Node node = new Node(Thread.currentThread(), mode);
      // 记录原尾节点
      Node pred = tail;
      // 快速尝试，添加新节点为尾节点
      if (pred != null) {
          // 设置新 Node 节点的尾节点为原尾节点
          node.prev = pred;
         // CAS 设置新的尾节点
         if (compareAndSetTail(pred, node)) {
             // 成功，原尾节点的下一个节点为新节点
             pred.next = node;
             return node;
         }
     }
     // 失败，多次尝试，直到成功
     enq(node);
     return node;
}


private Node enq(final Node node) {
     // 多次尝试，直到成功为止
     for (;;) {
         // 记录原尾节点
         Node t = tail;
         /**
          * 原尾节点不存在，说明队列为空，那么创建首和尾节点都为 new Node()
          * 不把自己设置为首节点是因为，唤醒的规则是看自己的 prev 是否是 head，如果把自己直接 head，那么和后面的逻辑相违背
          */
         if (t == null) {
             if (compareAndSetHead(new Node()))
                 tail = head;
         // 原尾节点存在，添加新节点为尾节点
         } else {
             //设置为尾节点
             node.prev = t;
             // CAS 设置新的尾节点
             if (compareAndSetTail(t, node)) {
                 // 成功，原尾节点的下一个节点为新节点
                 t.next = node;
                 return t;
             }
         }
     }
}
```



###### 出队

AbstractQueuedSynchronizer#setHead()

* 过程非常简单，`head` 执行该节点并断开原首节点的 `next` 和当前节点的 `prev` 即可。
* 注意，在这个过程是**不需要使用 CAS 来保证**的，因为**只有一个**线程，能够成功获取到同步状态。

```java
private void setHead(Node node) {
    head = node;
    // node 的 thread 就是当前线程。
    node.thread = null;
    node.prev = null;
}
```



##### 独占式获取

###### acquire

* 该方法对**中断不敏感**。也就是说，由于线程获取同步状态失败而加入到 CLH 同步队列中，后续对该线程进行中断操作时，线程**不会**从 CLH 同步队列中**移除**。只有成功出队之后，才知道该线程被调用过 interrupt。

```java
public final void acquire(int arg) {
        // 调用 tryAcquire(int arg) 方法，去尝试获取同步状态，获取成功直接返回
    if (!tryAcquire(arg) &&
        /**
         * tryAcquire 获取失败会执行到这里
         * 先调用 addWaiter 加入到 CLH 队列尾部
         * 然后调用 acquireQueued 自旋 和 park 。当返回 true 时，表示在这个过程中，发生过线程中断。
         */
         acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
        // 恢复线程中断的标识
        selfInterrupt();
}

/**
 * 需要自定义同步组件自己实现，该方法必须要保证线程安全的获取同步状态
 */
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}


final boolean acquireQueued(final Node node, int arg) {
    // 记录是否获取同步状态成功
    boolean failed = true;
    try {
        // 记录过程中，是否发生线程中断
        boolean interrupted = false;
        // 自旋
        for (;;) {
            // 当前线程的前驱节点
            final Node p = node.predecessor();
            // 当前线程的前驱节点是头结点，且获取同步状态成功
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            /**
             * 走到这里说明：前一个节点不是头节点 或者 是头节点但还没释放同步状态
             */
            if (shouldParkAfterFailedAcquire(p, node) && // 根据 waitstatu 决定是否 park
                parkAndCheckInterrupt()) // 在这里执行当前线程的 park 并等待被 unpark（头节点 release 的时候会 unpark 它后面的节点）。 
            interrupted = true;       
        }    
    } finally {
        // 获取同步状态发生异常，取消获取。    
        if (failed)   
        	cancelAcquire(node);    
    } 
}


static void selfInterrupt() {
    Thread.currentThread().interrupt();
}

```



* **公用方法** 

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
      
    // 获得前一个节点的等待状态   
    int ws = pred.waitStatus;    
    if (ws == Node.SIGNAL) //  Node.SIGNAL      
        // 如果说它的前一个节点已经承诺了，它会在释放同步块或者被取消的时候唤醒自己，那就可以安心 park 了。
   	 	return true;    
    
    if (ws > 0) { // Node.CANCEL        
        // 如果前一个节点已经被取消了，那就移除前面所有被取消的节点   
        do {             
            node.prev = pred = pred.prev;         
        } while (pred.waitStatus > 0);
        // 让最后一个不是 cancel 的节点连接自己
        pred.next = node;     
    } else { // 0 或者 Node.PROPAGATE        
        // 如果前面的节点是 0（没有初始化）、-3（共享向后传播），那就把其改为 signal，然后在 for 循环的下一次，当前节点就可以安心 park
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);     
    }     
    return false;
}


private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}


private void cancelAcquire(Node node) {      
    // node 不存在则直接返回     
    if (node == null)            
        return;
        
    node.thread = null;        
    // prev 指针到最后一个没被取消的节点
    Node pred = node.prev;    
    while (pred.waitStatus > 0)      
        node.prev = pred = pred.prev;

    /**
     * 这里补充说明一下，此处为什么没有让 pred.next = node，只让 node 状态设为 cancle 即可：
     * 	  和上面的 shouldParkAfterFailedAcquire 方法对应，每个入队后获取同步状态失败的节点都会执行那个方法，其会从由 prev 指针从后向前遍历，
     *  让 node 的 prev 连接前面最后一个不是 cancle 的节点，并且完成 pred.next = node 的双向连接，从而丢弃中间那一段节点。
     *    并且还有一种情况，就算node.next 一直阻塞没有执行 shouldParkAfterFailedAcquire，但只要 pred 获取同步状态成功，因为 unparkSuccessor() 
     *  只会会先离 head 最近的非 cancle 节点，这个节点肯定在 node 后面，那 node 就直接被丢弃了。
     */
    
     
    // 后面 cas 要用       
    Node predNext = pred.next;
        
    // 设置为 cancle 即可
    node.waitStatus = Node.CANCELLED;

           
    if (node == tail && compareAndSetTail(node, pred)) {   
        // 如果是尾节点，那直接删除就可以
        compareAndSetNext(pred, predNext, null);    
    } else {             
        int ws;     
        
        if (pred != head &&    // node 前面的节点不是头节点
            ((ws = pred.waitStatus) == Node.SIGNAL ||  // 判断 pred 的下一个节点是否需要被唤醒，因为 node 已经 cancle，所以必须唤醒 next了。
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) && // 或者 node 本来要唤醒下一个 那就看可否把唤醒任务交给 pred
            pred.thread != null) {  // 防止在上面的判断后 pred 变成了 head        
            Node next = node.next;           
            if (next != null && next.waitStatus <= 0)   
                // 即：next 需要被 pred 唤醒，所以让 pred 连接 next。
                // 不然在共享模式的 setHeadAndPropagate() 进行传播时，可能会错误使用 cancle 的节点进行判断
                compareAndSetNext(pred, predNext, next);           
        } else {      
            // 前面的节点是头节点，那么唤醒头节点后第一个不是 cancle 的节点，因为 node 的下一个节点的阻塞等待的。
            unparkSuccessor(node);        
        }       
       
        node.next = node; // help GC   
        
        // prev 指针：next -> node -> pred
        // next 指针: pred -> next
    } 
}
```



###### acquireInterruptibly

* 如果当前线程被中断了，会**立刻**响应中断，并抛出 InterruptedException 异常。

```java
public final void acquireInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}


private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 中断的话不用等出队，直接抛出异常
                throw new InterruptedException(); // <1>
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



###### tryAcquireNanos

* 该方法为 `#acquireInterruptibly(int arg)` 方法的进一步增强，它除了响应中断外，还有**超时控制**。即如果当前线程没有在指定时间内获取同步状态，则会返回 false ，否则返回 true 。

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}

// park 的最短时间
static final long spinForTimeoutThreshold = 1000L;
 
private boolean doAcquireNanos(int arg, long nanosTimeout)        
		throws InterruptedException {    
    
    // nanosTimeout <= 0    
    if (nanosTimeout <= 0L)       
    return false;     
    // 超时时间
    final long deadline = System.nanoTime() + nanosTimeout; 
    // 新增 Node 节点 
    final Node node = addWaiter(Node.EXCLUSIVE); 
    boolean failed = true;  
    try {    
        // 自旋   
        for (;;) {     
            final Node p = node.predecessor();         
            // 获取同步状态成功          
            if (p == head && tryAcquire(arg)) {          
                setHead(node);          
                p.next = null; // help GC          
                failed = false;            
                return true;           
            }          
                      
            // 获取失败，重新计算需要休眠的时间          
            nanosTimeout = deadline - System.nanoTime();          
            // 已经超时，返回false           
            if (nanosTimeout <= 0L)         
            	return false;  
            
            // 如果没有超时，则等待nanosTimeout纳秒           
            // 说明为什么 park 全部剩余时间：因为如果获取失败说明 pred 不是头节点，那么只有等 head 去唤醒，如果没有被 head 唤醒就说明没轮到 node
            if (shouldParkAfterFailedAcquire(p, node) &&              
                	nanosTimeout > spinForTimeoutThreshold)              
            	LockSupport.parkNanos(this, nanosTimeout);     
            
            // 线程是否已经中断了          
            if (Thread.interrupted())            
            throw new InterruptedException();       
        }    
    } finally {     
        if (failed)      
        cancelAcquire(node); 
    }
}
```



##### 独占式释放

###### release

* 三种 aquire 都是同一个 release，都是唤醒 head 后第一个不是 cancle 的节点

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        // head 不为空即队列不为空，waitStatus != 0 是要保证不是尾节点，只有尾节点没有后一个节点去改变它的状态
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
```



* **公用方法**

```java
private void unparkSuccessor(Node node) {

    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    // 如果下一个节点是 cancle 的，那么从后向前找到最后一个不是cancle的节点
    // 从后向前是因为，在 cancelAcquire() 里，让 node.next = next 了，所以防止死循环
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
             if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
##### 共享式获取

###### acquireShared

```java
public final void acquireShared(int arg) {
     if (tryAcquireShared(arg) < 0)
     doAcquireShared(arg);
}

protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}

private void doAcquireShared(int arg) {
    // 共享式节点
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 前驱节点
            final Node p = node.predecessor();
            // 如果其前驱节点，获取同步状态            
            if (p == head) {
               // 尝试获取同步
               int r = tryAcquireShared(arg);
               if (r >= 0) {
                   /** 设置新的首节点，并根据条件，唤醒下一个节点。会不断改变头节点然后唤醒其下一个节点 */
                   setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            // 前面不是head，或者 head 还没有释放同步状态
            if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

* **公用方法**

```java
/**
 * 共享式的核心，在只 setHeader 的基础上，继续唤醒后面的节点，独占式只在释放的时候唤醒后面的节点。
 */
private void setHeadAndPropagate(Node node, int propagate) {
	// 记录原来的首节点 h
    Node h = head; 
    // 设置 node 为新的首节点
    setHead(node);
        
    if (propagate > 0 /** tryAcquireShared() 返回结果 */ || h == null || h.waitStatus < 0 || 
        (h = head) == null || h.waitStatus < 0) { // 原来的或者新的首节点，等待状态为 Node.PROPAGATE 或者 Node.SIGNAL 时，可以继续向下唤醒
        Node s = node.next;
        
        if (s == null || s.isShared())
            // 会唤醒 node（head）的下一个节点，因为 propagate > 0，所以下一个节点 tryAcquireShared() 大概率会成功
            //（不是百分百是因为可能被来的线程在 acquireShared() 方法直接抢走。）
            doReleaseShared();// 共享式释放时说明
    }
}

final boolean isShared() {
     return nextWaiter == SHARED;
}
```



###### acquireSharedInterruptibly

* 共享式获取响应中断

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) { 
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 中断了直接抛异常
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



###### tryAcquireSharedNanos

```java
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquireShared(arg) >= 0 ||
        doAcquireSharedNanos(arg, nanosTimeout);
}

private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                // 只休眠剩余的时间，不会一直休眠等待头节点唤醒
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



##### 共享式释放

###### releaseShared

```java
public final boolean releaseShared(int arg) {
     if (tryReleaseShared(arg)) {
         doReleaseShared();
         return true;
     }
     return false;
}

// 模板方法
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}

private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            // 如果没有唤醒过后面的几点
            if (ws == Node.SIGNAL) {
                // 防止多线程，已经唤醒该节点的下个节点后又来唤醒一次
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    // 重试
                    continue;            // loop to recheck cases
                // 唤醒后面的节点
                unparkSuccessor(h);
            }
            // 如果只有 head 一个节点了，那么其状态就会是 0，那就设置为传播状态，在 setHeadAndPropagate() 会用到判断是否向后唤醒
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        // 走到这说明已经完成了上面的状态设置，如果 head 没有变更的话就不用再重新设置，退出即可。
        if (h == head)                   // loop if head changed
            break;
    }
}
```

##### LockSupport

###### 基本特征

重入性：

* LockSupport是非重入锁（不可 park 两次）

面向线程锁：

* 这是LockSupport很重要的一个特征，也是与synchronized ,reentrantlock最大的不同，也就没有公平锁和非公平的区别（因为只是针对一个线程的操作）。

锁park与解锁unpark顺序可颠倒性：

* 这个特征虽然是 Java 锁中比较独特是特征，其实这个特征也是基于LockSupport的 状态，只要 counter 计数器是 1 ，直接不 wait 。

解锁unpark的重复性：

* unpark可重复是指在解锁的时候可以重复调用unpark；同样因为LockSupport基于二元锁状态重复调用unpark并不会影响到下一次park操作；



###### 底层实现

Unsafe 中的相关实现是通过 os_posix.cpp 中的 `Parker::park()` 及 `Parker::unpark()` 实现的。

实现的关键是：

* Mutex 互斥锁（保证 counter 线程的安全，不会出现丢失 park 或者 unpark） 、

* Condition 信号量 （线程 wait 和 signal 的信号量）
* _counter计数器（park 和 unpark 的计数器）



```c++
void Parker::park(bool isAbsolute, jlong time) {

  // 检查，确保无锁
  if (Atomic::xchg(0, &_counter) > 0) return;

  Thread* thread = Thread::current();
  assert(thread->is_Java_thread(), "Must be JavaThread");
  JavaThread *jt = (JavaThread *)thread;

  // 线程中断的话返回
  if (Thread::is_interrupted(thread, false)) {
    return;
  }

  // Next, demultiplex/decode time arguments
  struct timespec absTime;
  if (time < 0 || (isAbsolute && time == 0)) { // don't wait at all
    return;
  }
  if (time > 0) {
    to_abstime(&absTime, time, isAbsolute, false);
  }

  // Enter safepoint region
  // Beware of deadlocks such as 6317397.
  // The per-thread Parker:: mutex is a classic leaf-lock.
  // In particular a thread must never block on the Threads_lock while
  // holding the Parker:: mutex.  If safepoints are pending both the
  // the ThreadBlockInVM() CTOR and DTOR may grab Threads_lock.
  ThreadBlockInVM tbivm(jt);

 
  if (Thread::is_interrupted(thread, false) || // 中断直接返回
      pthread_mutex_trylock(_mutex) != 0) { // 修改计数器前先获取锁，没获取取到就直接返回
    return;
  }

  int status;
  // 已经被调用过 unpark
  if (_counter > 0)  { // no wait needed
    _counter = 0;
    // 释放 mutex 互斥锁
    status = pthread_mutex_unlock(_mutex);
    assert_status(status == 0, status, "invariant");
    // Paranoia to ensure our locked and lock-free paths interact
    // correctly with each other and Java-level accesses.
    OrderAccess::fence();
    return;
  }

  OSThreadWaitState osts(thread->osthread(), false /* not Object.wait() */);
  jt->set_suspend_equivalent();

  assert(_cur_index == -1, "invariant");
  if (time == 0) {
    _cur_index = REL_INDEX; // arbitrary choice when not timed
    // 调用 pthread_cond_wait 等待
    status = pthread_cond_wait(&_cond[_cur_index], _mutex);
    assert_status(status == 0, status, "cond_timedwait");
  }
  else {
    _cur_index = isAbsolute ? ABS_INDEX : REL_INDEX;
    // 调用 pthread_cond_timedwait 等待
    status = pthread_cond_timedwait(&_cond[_cur_index], _mutex, &absTime);
    assert_status(status == 0 || status == ETIMEDOUT,
                  status, "cond_timedwait");
  }
  _cur_index = -1;

  // cond wait 被 signal 唤醒，设置 counter 为 0，还原状态
  _counter = 0;
  // 修改完计数器，那么释放锁
  status = pthread_mutex_unlock(_mutex);
  assert_status(status == 0, status, "invariant");
  // Paranoia to ensure our locked and lock-free paths interact
  // correctly with each other and Java-level accesses.
  OrderAccess::fence();

  // If externally suspended while waiting, re-suspend
  if (jt->handle_special_suspend_equivalent_condition()) {
    jt->java_suspend_self();
  }
}
```

```c++
void Parker::unpark() {
  // 修改计数器前获取锁
  int status = pthread_mutex_lock(_mutex);
  assert_status(status == 0, status, "invariant");
  const int s = _counter;
  // 计数器设置为 1
  _counter = 1;
  // must capture correct index before unlocking
  int index = _cur_index;
  // 修改完计数器就释放锁
  status = pthread_mutex_unlock(_mutex);
  assert_status(status == 0, status, "invariant");


  if (s < 1 && index != -1) {
    // 唤醒 park 的线程
    status = pthread_cond_signal(&_cond[index]);
    assert_status(status == 0, status, "invariant");
  }
}
```

park 主要流程

* <1> 当_ counter  >  0，则直接调用 pthread_mutex_unlock 解锁并返回；
* <2.1> 当超时时间 time >0，则调用 pthread_cond_timedwait 进行超时等待，直到超时时间到达；
* <2.2> 当超时时间 time = 0，则调用 pthread_cond_wait 等待；
* <3> 当wait返回时设置 _counter = 0，并调用 pthread_mutex_unlock 解锁；

unpark 主要流程

* <1> 调用 pthread_mutex_lock 获取锁，设置 _counter=1，调用 pthread_mutex_unlock 解锁；
* <2> 若 _counter 的原值等于0，则调用 pthread_cond_signal 进行通知处理；

#### DelayQueue

* 一个**支持延时**获取元素的**无界**阻塞队列，里面的元素全部都是“可延期”的元素，**列头的元素必须是最先“到期”**的元素，如果列头不出队，其他元素是无法出队的。
* 主要用于两个方面：缓存（清掉缓存中超时的缓存数据）、任务超时处理
* 底层实现是优先队列，在出队时调用了内部优先队列 peek 方法先查询其队首元素，如果到期了才调用优先队列的 poll 让队首出队，否则就阻塞等队首到期，也就是说优先队列中只有前一个元素到期并出队后，后面的元素才能出队。并发安全也是通过 Lock。



##### 元素

* 元素必须实现 Delayed 接口

```java
// 实现该接口的getDelay()方法，同时定义compareTo()方法即可。
// 注意：ompareTo 方法提供的排序，必须与过期时间一致，不然就会出现最小堆堆顶并不是最先过期的元素
public interface Delayed extends Comparable<Delayed> {
    // 返回与此对象相关的的剩余时间
    long getDelay(TimeUnit unit);
}
```



##### 属性

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
        implements BlockingQueue<E> {
    /** 可重入锁 */
    private final transient ReentrantLock lock = new ReentrantLock();
    /** 支持优先级的队列，用的也是最小堆，无界非阻塞 */
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    /** leader 会超时等待，通常为第一个查询到未到期头节点的线程 */
    private Thread leader = null;
    /** 等待有可出队元素 */
    private final Condition available = lock.newCondition();

    /**
     * 省略很多代码
     */
}
```



##### 入队

```java
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        // 向 PriorityQueue中插入元素
        q.offer(e);
        // 如果当前元素的对首元素（优先级最高），leader设置为空，唤醒所有等待线程
        // peek 是查询而非出队
        if (q.peek() == e) {
            // 设置为 null 防止内存泄漏
            leader = null;
            available.signal();
        }
        // 无界队列，永远返回true
        return true;
    } finally {
        lock.unlock();
    }
}
```



##### 出队

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lockInterruptibly();
    try {
        for (;;) {
            // 查询队首元素
            E first = q.peek();
            // 队首为空，阻塞，等待off()操作唤醒
            if (first == null)
                available.await();
            else {
                // 获取对首元素的超时时间
                long delay = first.getDelay(NANOSECONDS);
                // <=0 表示已过期，出对，return
                if (delay <= 0)
                    // 只有这一个出口，只有队首元素到期队才可以出队
                    return q.poll();
                
                // 不设为 null 会引起内存泄漏
                // 因为可能有多个线程来获取队首，结果没到期全都阻塞，那么这么多个线程都持有了队首元素的引用
                // 但是队首到期成功出队后，如果其他那么多个线程还在阻塞，那这个队首使用完就无法被回收
                first = null; // don't retain ref while waiting
                
                // leader != null 证明有比它来的早的线程已经在超时等待队首到期了
                if (leader != null)
                    // 所以不设置等待超时时间，因为队首到期出对后，那个线程会唤醒下一个等待的线程
                    available.await();
                else {
                    // leader 为空说明是队首的元素未到期，就将leader 设置为当前线程
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        // 超时阻塞，时间为队首到期时间
                        available.awaitNanos(delay);
                    } finally {
                        // 释放leader，后面的元素就可以使用超时等待了
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        // 唤醒阻塞线程（一般唤醒的是队首后面阻塞等待出队的元素）
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
```


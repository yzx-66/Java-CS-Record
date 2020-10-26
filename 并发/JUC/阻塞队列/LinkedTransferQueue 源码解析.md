#### LinkedTransferQueue

* LinkedTransferQueue是下面三者的集合体

  * LinkedBlockingQueue（支持阻塞，即失败入队，但不再支持获取 Lock 的阻塞了，因为是基于 cas 的）
  * SynchronousQueue（公平模式，实现了 Transfer 接口）
  * ConcurrentLinkedQueue （支持非阻塞，因为传统的阻塞队列都是使用的 Lock，想要 offer 或者 poll 等操作必须先获得 lock，而非阻塞是 直接cas 一次，失败就直接返回，不用阻塞获得锁）
* 四种模式

  * NOW，立即返回，没有匹配到立即返回，不做入队操作     

    * 对应的方法有：poll()、tryTransfer(e) 

  * ASYNC，异步，元素入队但当前线程不会阻塞（相当于无界LinkedBlockingQueue的元素入队）     

    * 对应的方法有：add(e)、offer(e)、put(e)、offer(e, timeout, unit) 

  * SYNC，同步，元素入队后当前线程阻塞，等待被匹配到     

    * 对应的方法有：take()、transfer(e)

  * TIMED，有超时，元素入队后等待一段时间被匹配，时间到了还没匹配到就返回元素本身    

    *  对应的方法有：poll(timeout, unit)、tryTransfer(e, timeout, unit)



* 底层：所有方法最终调用的都是 xfer()，这个方法和 synchronousQueue 的 transfer 方法一样，都是基于 cas 无锁的，会先检查是否符合匹配规则，即队列所有节点是否是数据节点和请求不同，如果不同的话会遍历匹配，遍历是为了防止并发下被其他线程先匹配了，如果请求和队列都是数据节点或者都不是数据节点，则不能进行匹配的话，如果是 Now 模式就立即返回失败了，其他模式会把该模式的这个请求入队，入队采用 cas 保证一定成功，在入队成功之后如果是 Async 模式（即入队请求），那么也就直接返回了，剩下的 Time 和 Sync 模式，会阻塞等待匹配的结果返回，即使用 LockSupport 定时阻塞，然后检查 item 是否发生了交换，发生了就返回



##### 属性

```java
public class LinkedTransferQueue<E> extends AbstractQueue<E>
    implements TransferQueue<E>, java.io.Serializable {
        // 头节点
        transient volatile Node head;
        // 尾节点
        private transient volatile Node tail;
        // 放取元素的几种方式：
        // 立即返回，用于非超时的poll()和tryTransfer()方法中
        private static final int NOW   = 0; // for untimed poll, tryTransfer
        // 异步，不会阻塞，用于放元素时，因为内部使用无界单链表存储元素，不会阻塞放元素的过程
        private static final int ASYNC = 1; // for offer, put, add
        // 同步，调用的时候如果没有匹配到会阻塞直到匹配到为止
        private static final int SYNC  = 2; // for transfer, take
        // 超时，用于有超时的poll()和tryTransfer()方法中
        private static final int TIMED = 3; // for timed poll, tryTransfer


        static final class Node {
            // 是否是数据节点（也就标识了是生产者还是消费者）
            final boolean isData;   // false if this is a request node
            // 元素的值
            volatile Object item;   // initially non-null if isData; CASed to match
            // 下一个节点
            volatile Node next;
            // 持有元素的线程
            volatile Thread waiter; // null until waiting
        }
    
    	// ...
```



##### 构造方法

```java
public LinkedTransferQueue() {
}

public LinkedTransferQueue(Collection<? extends E> c) {
    this();
    addAll(c);
}
```



##### 入队

四个方法都是一样的，使用 异步的方式 调用xfer()方法，因为是无界队列，当然不会等待了

```java
public void put(E e) {
    // 异步模式，不会阻塞，不会超时
    // 因为是放元素，单链表存储，会一直往后加
    xfer(e, true, ASYNC, 0);
}

public boolean offer(E e, long timeout, TimeUnit unit) {
    xfer(e, true, ASYNC, 0);
    return true;
}

public boolean offer(E e) {
    xfer(e, true, ASYNC, 0);
    return true;
}

public boolean add(E e) {
    xfer(e, true, ASYNC, 0);
    return true;
}
```



##### 出队

出队的四个方法也是直接或间接的调用 xfer() 方法，只是模式不同

```java
public E remove() {
    E x = poll();
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}
public E take() throws InterruptedException {
    // 同步模式，会阻塞直到取到元素
    E e = xfer(null, false, SYNC, 0);
    if (e != null)
        return e;
    Thread.interrupted();
    throw new InterruptedException();
}

public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    // 有超时时间
    E e = xfer(null, false, TIMED, unit.toNanos(timeout));
    if (e != null || !Thread.interrupted())
        return e;
    throw new InterruptedException();
}

public E poll() {
    // 立即返回，没取到元素返回null
    return xfer(null, false, NOW, 0);
}

```



##### 移交元素

```java
public boolean tryTransfer(E e) {
    // 立即返回
    return xfer(e, true, NOW, 0) == null;
}

public void transfer(E e) throws InterruptedException {
    // 同步模式
    if (xfer(e, true, SYNC, 0) != null) {
        Thread.interrupted(); // failure possible only due to interrupt
        throw new InterruptedException();
    }
}

public boolean tryTransfer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    // 有超时时间
    if (xfer(e, true, TIMED, unit.toNanos(timeout)) == null)
        return true;
    if (!Thread.interrupted())
        return false;
    throw new InterruptedException();
}

```



##### xfer()

```java
/**
  * e：表示元素；
  * haveData：表示是否是数据节点，
  * how：表示放取元素的方式，上面提到的四种，NOW、ASYNC、SYNC、TIMED；
  * nanos：表示超时时间；
  */
private E xfer(E e, boolean haveData, int how, long nanos) {
    // 不允许放入空元素
    if (haveData && (e == null))
        throw new NullPointerException();
    
    Node s = null;                        // the node to append, if needed
    
    // 外层循环，自旋，失败就重试
    retry:
    for (;;) {                            // restart on append race
        // 队列中节点肯定全是数据节点或者全都不是数据节点，所以如果当前请求和队列现在是否数据节点匹配（即不同），
        // 那么就遍历进行匹配，因为可能有并发导致匹配失败，所以要遍历。
        for (Node h = head, p = h; p != null;) { // find & match first node
            // p节点的模式
            boolean isData = p.isData;
            // p节点的值
            Object item = p.item;
            
            // p 没有被取消，再校验一下是否已经被其他线程匹配
            if (item != p && (item != null) == isData) { // unmatched
                // 如果两者都是数据节点或者都不是数据节点，则不能匹配，直接挑出循环后尝试入队
                if (isData == haveData)   // can't match
                    break;
                
                // 否则一个数据节点，一个非数据节点，则尝试匹配，即 cas 更新 item
                if (p.casItem(item, e)) { // match
                    // 没被其他线程竞争，匹配成功，下面更新 head，即下一次要匹配的节点。
                    for (Node q = p; q != h;) {
                        // 当前节点是 q ，下一个要匹配的节点是 q.next
                        Node n = q.next;  // update by 2 unless singleton
                        // head 已被当前线程匹配，将其更新为下一个
                        if (head == h && casHead(h, n == null ? q : n)) {
                            h.forgetNext();
                            break;
                        }                 // advance and retry
                        // 更新 head 失败会执行下面
                        // 如果新的头节点为空，或者其next为空，或者其next未匹配，就重试
                        if ((h = head)   == null ||
                            (q = h.next) == null || !q.isMatched())
                            break;        // unless slack < 2
                    }
                    // 唤醒p中等待的线程
                    LockSupport.unpark(p.waiter);
                    // 并返回匹配到的元素
                    return LinkedTransferQueue.<E>cast(item);
                }
            }
            
            // 执行到这说明， p 已经被匹配了或者尝试匹配的时候失败了
            Node n = p.next;
            p = (p != n) ? n : (h = head); // Use head if p offlist
        }

        /**
         * 到这里肯定是队列中存储的节点类型和自己一样，或者队列中没有元素了，那么就入队（不管放元素还是取元素都得入队）
         * 入队又分成四种情况
         */
        // 如果不是立即返回
        if (how != NOW) {                 // No matches available
            // 新建s节点
            if (s == null)
                s = new Node(e, haveData);
            // 尝试入队
            Node pred = tryAppend(s, haveData);
            // 入队失败，重试
            if (pred == null)
                continue retry;           // lost race vs opposite mode
            // 如果不是异步（同步或者有超时），就等待匹配，与返回匹配的结果
            // 该方法同 SynchrnousQueue，会调用 LookSupport 定时阻塞，然后检查当前节点的 item 属性是否发声了交换
            if (how != ASYNC)
                return awaitMatch(s, pred, e, (how == TIMED), nanos);
        }
        return e; // not waiting
    }
}
```



LinkedTransferQueue与SynchronousQueue（公平模式）有什么异同呢？

（1）在java8中两者的实现方式基本一致，都是使用的双重队列；

（2）前者完全实现了后者，但比后者更灵活；

（3）后者不管放元素还是取元素，如果没有可匹配的元素，所在的线程都会阻塞；

（4）前者可以自己控制放元素是否需要阻塞线程，比如使用四个添加元素的方法就不会阻塞线程，只入队元素，使用transfer()会阻塞线程；

（5）取元素两者基本一样，都会阻塞等待有新的元素进入被匹配到

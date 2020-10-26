#### SynchronousQueue

* 特性
  * SynchronousQueue **没有容量**。与其他 BlockingQueue 不同，SynchronousQueue 是一个不存储元素的 BlockingQueue。**可以有多个 put ，但每一个put操作必须要等待一个take操作，否则该线程会一直阻塞在 put，反之亦然**。
  * 因为没有容量，所以对应 peek, contains, clear, isEmpty ... 等方法其实是无效的。例如 clear 是不执行任何操作的，contains 始终返回 false，peek 始终返回 null。
  * SynchronousQueue 分为公平和非公平，默认情况下采用非公平性访问策略，当然也可以通过构造函数来设置为公平性访问策略（为true即可）。
* 底层
  * 每个节点都有自己的模式，用来判断是请求节点（出队请求）还是数据节点（入队请求），如果队首节点的模式和这次 transfer 的模式不同，那就匹配上了，那么就可以让队首出队并唤醒阻塞在队首上的线程，如果不匹配那么就如入队等待。注意正是因为这种特性，队列中等待的节点永远可以保证是同一模式。
    * 公平：用队列实现，如果是同种模式要入队阻塞，那么就放到队尾（casNext）
    * 非公平：用栈实现，如果是同种模式要入栈阻塞，因为栈的特性，是放到第一个位置（casHead），所以后入栈，但先出栈匹配
  * 并发安全是通过 CAS 保证并发安全，入队等待（模式相同，自旋 + casNext ，失败则从循环开始处重新执行）和节点匹配（模式不同，自旋 + casItem，节点的 item 属性是两个线程数据交换的场地，一旦并发竞争这个节点被其他线程抢先匹配成功，item 就会 cas 失败，那么就也是从循环开始处再来，下次不一定还会模式不同，因为一旦 cas 成功会让被匹配的节点出队，所以不存在下次循环还匹配这次失败的节点）。阻塞是通过 LockSupport



##### Transferer

###### 定义

```java
abstract static class Transferer<E> {
    // 如果e != null，相当于将一个数据交给消费者
    // 如果e == null，则相当于从一个生产者接收一个消费者交出的数据。
    abstract E transfer(E e, boolean timed, long nanos);
}
```



###### TransferQueue

TransferQueue是实现公平性策略的核心类，其节点为QNode

```java
static final class TransferQueue<E> extends Transferer<E> {
    /** 头节点 */
    transient volatile QNode head;
    /** 尾节点 */
    transient volatile QNode tail;
    // 指向一个取消的结点
    //当一个节点中最后一个插入时，它被取消了但是可能还没有离开队列
    transient volatile QNode cleanMe;
    
    
    TransferQueue() {
    	QNode h = new QNode(null, false); // initialize to dummy node.
   	 	head = h;
    	tail = h;
	}
    
    
    static final class QNode {
        // next 域
        volatile QNode next;
        // item数据项
        volatile Object item;
        //  等待线程，用于park/unpark
        volatile Thread waiter;      
        //模式，表示当前是数据还是请求，只有当匹配的模式相匹配时才会交换
        final boolean isData;

        QNode(Object item, boolean isData) {
            this.item = item;
            this.isData = isData;
        }

        // CAS next域，在TransferQueue中用于向next推进
        boolean casNext(QNode cmp, QNode val) {
            return next == cmp &&
                    UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }

        // CAS itme数据项
        boolean casItem(Object cmp, Object val) {
            return item == cmp &&
                    UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
        }

        // 取消本结点，将item域设置为自身作为标志
        void tryCancel(Object cmp) {
            UNSAFE.compareAndSwapObject(this, itemOffset, cmp, this);
        }

        // 是否被取消，与tryCancel相照应只需要判断item释放等于自身即可
        boolean isCancelled() {
            return item == this;
        }


        boolean isOffList() {
            return next == this;
        }

        private static final sun.misc.Unsafe UNSAFE;
        private static final long itemOffset;
        private static final long nextOffset;

        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> k = QNode.class;
                itemOffset = UNSAFE.objectFieldOffset
                        (k.getDeclaredField("item"));
                nextOffset = UNSAFE.objectFieldOffset
                        (k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
	}

    /**
     * 省略很多代码
     */
}
```



###### TransferStack

TransferStack用于实现非公平性

````java
static final class TransferStack<E> extends Transferer<E> {

    // 三种 SNode 的 mode
    // REQUEST表示消费数据的消费者
    static final int REQUEST    = 0;
	// DATA表示生产数据的生产者
    static final int DATA       = 1;
	// FULFILLING，表示匹配另一个生产者或消费者
    static final int FULFILLING = 2;

    volatile SNode head;
    
    static final class SNode {
        // next 域
        volatile SNode next;
        // 相匹配的节点
        volatile SNode match;
        // 等待的线程
        volatile Thread waiter;
        // item 域
        Object item;                // data; or null for REQUESTs
        // 模式
        int mode;

        SNode(Object item) { this.item = item; }

        boolean casNext(SNode cmp, SNode val) {
            return cmp == next && UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val); 
        }

        // 将 s 结点设置成匹配的节点，若匹配成功，则unpark等待线程
        boolean tryMatch(SNode s) {
            if (match == null &&
                    UNSAFE.compareAndSwapObject(this, matchOffset, null, s)) {
                Thread w = waiter;
                if (w != null) {    // waiters need at most one unpark
                    waiter = null;
                    LockSupport.unpark(w);
                }
                return true;
            }
            return match == s;
        }

        // cancle 是把 match 的对象设置自己
        void tryCancel() { UNSAFE.compareAndSwapObject(this, matchOffset, null, this); }

        boolean isCancelled() { return match == this; }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long matchOffset;
        private static final long nextOffset;

        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> k = SNode.class;
                matchOffset = UNSAFE.objectFieldOffset(k.getDeclaredField("match"));
                nextOffset = UNSAFE.objectFieldOffset(k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
	}

    /**
     * 省略一堆代码
     */

}
````



##### 构造方法

```java
public class SynchronousQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable{
    
	public SynchronousQueue() {
        this(false);
    }

    public SynchronousQueue(boolean fair) {
        // 通过 fair 值来决定公平性和非公平性
        // 公平性使用TransferQueue，非公平性采用TransferStack
        transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
    }    
    
    //...
}

```



##### put & get

都是直接调用的 transfer

```java
// put操作
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    if (transferer.transfer(e, false, 0) == null) {
        Thread.interrupted();
        throw new InterruptedException();
    }
}

// take操作
public E take() throws InterruptedException {
    E e = transferer.transfer(null, false, 0);
    if (e != null)
        return e;
    Thread.interrupted();
    throw new InterruptedException();
}
```



##### TransferQueue

```java
E transfer(E e, boolean timed, long nanos) {
    QNode s = null;
    // 当前请求模式
    boolean isData = (e != null);

    for (;;) {
        QNode t = tail;
        QNode h = head;
        // 头、尾节点 为null，没有初始化
        if (t == null || h == null)
            continue;

        // 头尾节点相等（队列为null） 或者当前节点和队列节点模式一样，那么就要入队等待了
        if (h == t || t.isData == isData) {
            // tn = t.next
            QNode tn = t.next;
            // t != tail表示已有其他线程操作了，修改了tail，重新再来
            if (t != tail)
                continue;
            
            // tn != null，表示已经有其他线程添加了节点，tn 推进，重新处理
            if (tn != null) {
                // 当前线程帮忙推进尾节点，就是尝试将tn设置为尾节点
                advanceTail(t, tn);
                continue;
            }
            
            //  调用的方法的 wait 类型的, 并且 超时了, 直接返回 null
            // timed 在take操作阐述
            if (timed && nanos <= 0)
                return null;

            // s == null，构建一个新节点Node
            if (s == null)
                s = new QNode(e, isData);

            // 将新建的节点加入到队列中，如果插入失败，则重新自旋
            if (!t.casNext(null, s))
                continue;

            // 替换尾节点（不成功无所谓的，因为连接时会找到真正的尾节点，这个只是一个推进作用）
            advanceTail(t, s);

            // 调用 awaitFulfill, 进行阻塞，并等待返回结果
            Object x = awaitFulfill(s, e, timed, nanos);

            // 若返回的 x == s，表示当前线程已经超时或者中断，不然的话s == null 或者 是匹配的节点
            if (x == s) {
                // 清理节点S
                clean(t, s);
                return null;
            }

            // isOffList：用于判断节点是否已经从队列中离开了
            if (!s.isOffList()) {
                // 尝试将S节点设置为head，移出t
                advanceHead(t, s);
                if (x != null)
                    s.item = s;
                // 释放线程 ref
                s.waiter = null;
            }

            // 返回
            return (x != null) ? (E)x : e;

        }

        // 这里是就说明这个请求的模式 和队列里节点的模式不同，那么就可以匹配了
        else {
            // 第一个元素节点（head 指向的节点并不是第一个元素节点，可能是初始化节点或者已经匹配完了的节点）
            QNode m = h.next;

            // 不一致读，重新开始
            // 有其他线程更改了线程结构
            if (t != tail || m == null || h != head)
                continue;

            /**
             * 下面是生产者producer和消费者consumer匹配操作
             */
            Object x = m.item;
            // isData == (x != null)：判断isData与x的模式是否相同，相同表示已经匹配了
            // x == m ：m节点被取消了（取消的标志）
            // !m.casItem(x, e)：如果尝试将数据e设置到m上失败，说明这个节点被其他线程匹配过了
            if (isData == (x != null) ||  x  == m || !m.casItem(x, e)) {
                // 将m设置为头结点，h出列，再从头开始
                advanceHead(h, m);
                continue;
            }

            // 成功匹配了，m 设置为头结点h出列，向前推进
            advanceHead(h, m);
            // 唤醒 m 上的等待线程
            LockSupport.unpark(m.waiter);
            return (x != null) ? (E)x : e;
        }
    }
}

Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {

    // 超时控制
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    // 自旋次数
    // 如果节点Node恰好是第一个数据节点，则自旋一段时间，这里主要是为了效率问题，如果里面阻塞，会存在唤醒、线程上下文切换的问题
    // 如果生产者、消费者者里面到来的话，就避免了这个阻塞的过程
    int spins = ((head.next == s) ?
            (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    
    // 自旋
    for (;;) {
        // 线程中断了，剔除当前节点
        if (w.isInterrupted())
            s.tryCancel(e);

        // 如果被其他线程匹配了，直接返回当前 item 即可
        // 实际上 item 就是 put 、get 进行数据交换的地方
        Object x = s.item;
        if (x != e)
            return x;
        
        // 超时判断
        if (timed) {
            nanos = deadline - System.nanoTime();
            // 如果超时了，取消节点,continue，在if(x != e)肯定会成立，直接返回x
            if (nanos <= 0L) {
                s.tryCancel(e);
                continue;
            }
        }

        // 自旋- 1
        if (spins > 0)
            --spins;

        // 等待线程
        else if (s.waiter == null)
            s.waiter = w;

        // 进行没有超时的 park
        else if (!timed)
            LockSupport.park(this);

        // 自旋次数过了, 直接 + timeout 方式 park
        else if (nanos > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanos);
    }
}
```



##### TransferStack

和 TransferQueue 区别不大，只是变成了每次模式相同入队是从头入

```java
E transfer(E e, boolean timed, long nanos) {
    SNode s = null; // constructed/reused as needed
    int mode = (e == null) ? REQUEST : DATA;

    for (;;) {
        SNode h = head;
        // 栈为空 或者 当前节点模式与头节点模式一样，将节点压入栈内，等待匹配
        if (h == null || h.mode == mode) {
            // 超时
            if (timed && nanos <= 0) {
                // 节点被取消了，向前推进
                if (h != null && h.isCancelled())
                    //  重新设置头结点（弹出之前的头结点）
                    casHead(h, h.next);
                else
                    return null;
            }
            // 不超时
            // 生成一个SNode节点，并尝试替换掉头节点head (head -> s)
            // 头插失败的话还会重新自旋
            else if (casHead(h, s = snode(s, e, h, mode))) {
                // 自旋，等待匹配成功的节点唤醒
                SNode m = awaitFulfill(s, timed, nanos);
                // 返回的m == s 表示该节点被取消了或者超时、中断了
                if (m == s) {
                    // 清理节点S，return null
                    clean(s);
                    return null;
                }

                // 因为通过前面一步头插成功 s 成了 head ，如果h.next == s ，则表示有其他节点插入到 s 前面了,
                // 且该节点就是与节点 s 匹配的节点
                if ((h = head) != null && h.next == s)
                    // 将s.next节点设置为head，相当于取消节点h、s
                    casHead(h, s.next);

                // 如果是请求则返回匹配的域，否则返回节点 S 的域
                return (E) ((mode == REQUEST) ? m.item : s.item);
            }
        }

        // 如果栈不为null，且两者模式不匹配（h != null && h.mode != mode）
        // 说明他们是一队对等匹配的节点，尝试用当前节点s来满足h节点
        else if (!isFulfilling(h.mode)) {
            // head 节点已经取消了，向前推进
            if (h.isCancelled())
                casHead(h, h.next);

            // 尝试将当前节点打上"正在匹配"的标记，并设置为 head
            else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                // 循环loop
                for (;;) {
                    // s为当前节点，m是s的next节点，
                    // m节点是s节点的匹配节点
                    SNode m = s.next;
                    // m == null，其他节点把m节点匹配走了
                    if (m == null) {
                        // 将s弹出
                        casHead(s, null);
                        // 将s置空，下轮循环的时候还会新建
                        s = null;
                        // 退出该循环，继续主循环
                        break;
                    }
                    // 获取m的next节点
                    SNode mn = m.next;
                    // 尝试匹配
                    if (m.tryMatch(s)) {
                        // 匹配成功，将s 、 m弹出
                        casHead(s, mn);     // pop both s and m
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    } else
                        // 如果没有匹配成功，说明有其他线程已经匹配了，把m移出
                        s.casNext(m, mn);
                }
            }
        }
        // 到这最后一步说明节点正在匹配阶段
        else {
            // head 的next的节点，是正在匹配的节点，m 和 h配对
            SNode m = h.next;

            // m == null 其他线程把m节点抢走了，弹出h节点
            if (m == null)
                casHead(h, null);
            else {
                SNode mn = m.next;
                if (m.tryMatch(h))
                    casHead(h, mn);
                else
                    h.casNext(m, mn);
            }
        }
    }
}

SNode awaitFulfill(SNode s, boolean timed, long nanos) {
    // 超时
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    // 当前线程
    Thread w = Thread.currentThread();

    // 自旋次数
    // shouldSpin 用于检测当前节点是否需要自旋
    // 如果栈为空、该节点是首节点或者该节点是匹配节点，则先采用自旋，否则阻塞
    int spins = (shouldSpin(s) ?
            (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    
    for (;;) {
        // 线程中断了，取消该节点
        if (w.isInterrupted())
            s.tryCancel();

        // 匹配节点
        SNode m = s.match;

        // 如果匹配节点m不为空，则表示匹配成功，直接返回
        if (m != null)
            return m;
        // 超时
        if (timed) {
            nanos = deadline - System.nanoTime();
            // 节点超时，取消
            if (nanos <= 0L) {
                s.tryCancel();
                continue;
            }
        }

        // 自旋;每次自旋的时候都需要检查自身是否满足自旋条件，满足就 - 1，否则为0
        if (spins > 0)
            spins = shouldSpin(s) ? (spins-1) : 0;

        // 第一次阻塞时，会将当前线程设置到s上
        else if (s.waiter == null)
            s.waiter = w;

        // 阻塞 当前线程
        else if (!timed)
            LockSupport.park(this);
        
        // 超时
        else if (nanos > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanos);
    }
}
```

#### Exchanger

线程之间可以进行元素交换（了解就行了）

* 实际就是 Exchanger 在没有线程竞争的时候，提供了一个 Node 叫 slot 当作交换的中间场地，让你这个线程和另外一个线程把值放在 Node#item 进行交换
  * 交换流程是，如果你去交换了，但是 Node 里面没有 item 可以让你交换，那你就把你的 item 用 cas 放到 Node 里面，然后把你这个线程保存在 Node 里面，你挂起就行了，等下一个来交换的话线程，把你的 item 取走，然后把它的 item 放到 Node，再把你唤醒，然后你取走它的 item 自己返回就行了
* 但是如果多个线程都来交换了，那一个 Node 效率太低，所以就提供了个 Node 数组叫 arena 让线程们当作场地来交换。那么现在交换的话就要在循环中进行了，因为槽位多了，就有很多时候没有交换对象或者被别的线程抢走了。



##### 属性

```java
@sun.misc.Contended static final class Node {
    int index;              // arena的下标
    int bound;              // 上一次记录的Exchanger.bound
    int collides;           // 在当前bound下CAS失败的次数
    int hash;               // 伪随机数，用于自旋
    Object item;            // 这个线程的当前项，也就是需要交换的数据
    volatile Object match;  // 做releasing操作的线程传递的项
    volatile Thread parked; // 挂起时设置线程值，其他情况下为null
}

// Participant 的作用就是为每个线程保留唯一的一个Node节点，它继承ThreadLocal，同时在Node节点中记录在arena中的下标index。
private final Participant participant;
// 有线程竞争时的交换场地
private volatile Node[] arena;
// 没有线程竞争时的交换场地
private volatile Node slot;
```



##### 方法

###### exchange(V x)

```java
public V exchange(V x) throws InterruptedException {
    Object v;
    Object item = (x == null) ? NULL_ITEM : x; // translate null args
    if ((arena != null ||
         (v = slotExchange(item, false, 0L)) == null) && // arena为数组槽，如果为null，则执行slotExchange()方法
        ((Thread.interrupted() || // disambiguates null return
          (v = arenaExchange(item, false, 0L)) == null))) // 如果slotExchange()方法执行失败了就执行arenaExchange()方法
        throw new InterruptedException();
    return (v == NULL_ITEM) ? null : (V)v;
}
```



```java
private final Object slotExchange(Object item, boolean timed, long ns) {
        // 获取当前线程的节点 p
        Node p = participant.get();
        // 当前线程
        Thread t = Thread.currentThread();
        // 线程中断，直接返回
        if (t.isInterrupted())
            return null;
        // 自旋
        for (Node q;;) {
            //slot != null
            if ((q = slot) != null) {
                //尝试CAS替换
                if (U.compareAndSwapObject(this, SLOT, q, null)) {
                    Object v = q.item;      // 当前线程的项，也就是交换的数据
                    q.match = item;         // 做releasing操作的线程传递的项
                    Thread w = q.parked;    // 挂起时设置线程值
                    // 挂起线程不为null，线程挂起
                    if (w != null)
                        U.unpark(w);
                    return v;
                }
                //如果失败了，则创建arena
                //bound 则是上次Exchanger.bound
                if (NCPU > 1 && bound == 0 &&
                        U.compareAndSwapInt(this, BOUND, 0, SEQ))
                    arena = new Node[(FULL + 2) << ASHIFT];
            }
            //如果arena != null，直接返回，进入arenaExchange逻辑处理
            else if (arena != null)
                return null;
            else {
                p.item = item;
                if (U.compareAndSwapObject(this, SLOT, null, p))
                    break;
                p.item = null;
            }
        }

        /*
         * 等待 release
         * 进入spin+block模式
         */
        int h = p.hash;
        long end = timed ? System.nanoTime() + ns : 0L;
        int spins = (NCPU > 1) ? SPINS : 1;
        Object v;
        
        while ((v = p.match) == null) {
            if (spins > 0) {
                h ^= h << 1; h ^= h >>> 3; h ^= h << 10;
                if (h == 0)
                    h = SPINS | (int)t.getId();
                else if (h < 0 && (--spins & ((SPINS >>> 1) - 1)) == 0)
                    Thread.yield();
            }
            else if (slot != p)
                // 重置自旋数并重试
                spins = SPINS;
            else if (!t.isInterrupted() && arena == null &&
                    (!timed || (ns = end - System.nanoTime()) > 0L)) {
                U.putObject(t, BLOCKER, this);
                p.parked = t;
                if (slot == p)
                    // 挂起当前线程在 Node 节点，等下一个使用该节点交换的线程唤醒
                    U.park(false, ns);
                p.parked = null;
                U.putObject(t, BLOCKER, null);
            }
            else if (U.compareAndSwapObject(this, SLOT, p, null)) {
                v = timed && ns <= 0L && !t.isInterrupted() ? TIMED_OUT : null;
                break;
            }
        }
        U.putOrderedObject(p, MATCH, null);
        p.item = null;
        p.hash = h;
        return v;
    }
```

```java
/**
 * 比较绕，大概看看
 */
private final Object arenaExchange(Object item, boolean timed, long ns) {
    Node[] a = arena;
    Node p = participant.get();
    for (int i = p.index;;) {                      // access slot at i
        int b, m, c; long j;                       // j is raw array offset
        Node q = (Node)U.getObjectVolatile(a, j = (i << ASHIFT) + ABASE);
        // 可以交换
        if (q != null && U.compareAndSwapObject(a, j, q, null)) {
            Object v = q.item;                     // release
            q.match = item;
            Thread w = q.parked;
            if (w != null)
                U.unpark(w);
            return v;
        }
        else if (i <= (m = (b = bound) & MMASK) && q == null) {
            p.item = item;                         // offer
            if (U.compareAndSwapObject(a, j, null, p)) {
                long end = (timed && m == 0) ? System.nanoTime() + ns : 0L;
                Thread t = Thread.currentThread(); // wait
                // 自旋，情况还比较多
                for (int h = p.hash, spins = SPINS;;) {
                    Object v = p.match;
                    if (v != null) {
                        U.putOrderedObject(p, MATCH, null);
                        p.item = null;             // clear for next use
                        p.hash = h;
                        return v;
                    }
                    else if (spins > 0) {
                        h ^= h << 1; h ^= h >>> 3; h ^= h << 10; // xorshift
                        if (h == 0)                // initialize hash
                            h = SPINS | (int)t.getId();
                        else if (h < 0 &&          // approx 50% true
                                 (--spins & ((SPINS >>> 1) - 1)) == 0)
                            Thread.yield();        // two yields per wait
                    }
                    else if (U.getObjectVolatile(a, j) != p)
                        spins = SPINS;       // releaser hasn't set match yet
                    else if (!t.isInterrupted() && m == 0 &&
                             (!timed ||
                              (ns = end - System.nanoTime()) > 0L)) {
                        U.putObject(t, BLOCKER, this); // emulate LockSupport
                        p.parked = t;              // minimize window
                        if (U.getObjectVolatile(a, j) == p)
                            U.park(false, ns);
                        p.parked = null;
                        U.putObject(t, BLOCKER, null);
                    }
                    else if (U.getObjectVolatile(a, j) == p &&
                             U.compareAndSwapObject(a, j, p, null)) {
                        if (m != 0)                // try to shrink
                            U.compareAndSwapInt(this, BOUND, b, b + SEQ - 1);
                        p.item = null;
                        p.hash = h;
                        i = p.index >>>= 1;        // descend
                        if (Thread.interrupted())
                            return null;
                        if (timed && m == 0 && ns <= 0L)
                            return TIMED_OUT;
                        break;                     // expired; restart
                    }
                }
            }
            else
                p.item = null;                     // clear offer
        }
        else {
            if (p.bound != b) {                    // stale; reset
                p.bound = b;
                p.collides = 0;
                i = (i != m || m == 0) ? m : m - 1;
            }
            else if ((c = p.collides) < m || m == FULL ||
                     !U.compareAndSwapInt(this, BOUND, b, b + SEQ + 1)) {
                p.collides = c + 1;
                i = (i == 0) ? m : i - 1;          // cyclically traverse
            }
            else
                i = m + 1;                         // grow
            p.index = i;
        }
    }
}
```

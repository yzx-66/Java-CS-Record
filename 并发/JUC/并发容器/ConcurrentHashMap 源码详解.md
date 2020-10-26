#### ConcurrentHashMap

##### hash 槽节点类型

###### Node

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;             //带有volatile，保证可见性
    volatile Node<K,V> next;    //下一个节点的指针

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }

    public final K getKey()       { return key; }
    public final V getValue()     { return val; }
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
    public final String toString(){ return key + "=" + val; }
    /** 不允许修改value的值 */
    public final V setValue(V value) {
        throw new UnsupportedOperationException();
    }

    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = val) || v.equals(u)));
    }

    /**  赋值get()方法 */
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```

###### TreeBin

* 树节点，但不是 hash 槽位保存的节点，hash 槽位保存的是树，因为如果槽位保存树节点，而不是整棵树的话，那么在进行红黑树的调整时候，可能会让 hash 槽位保存的 TreeNode 变为另外一个，那么就有并发问题了，没法保证这个槽位不会再被其他线程同时进行写操作。

```java
static final class TreeNode<K,V> extends Node<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;

        TreeNode(int hash, K key, V val, Node<K,V> next,
                 TreeNode<K,V> parent) {
            super(hash, key, val, next);
            this.parent = parent;
        }


        Node<K,V> find(int h, Object k) {
            return findTreeNode(h, k, null);
        }

        //查找hash为h，key为k的节点
        final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
            if (k != null) {
                TreeNode<K,V> p = this;
                do  {
                    int ph, dir; K pk; TreeNode<K,V> q;
                    TreeNode<K,V> pl = p.left, pr = p.right;
                    if ((ph = p.hash) > h)
                        p = pl;
                    else if (ph < h)
                        p = pr;
                    else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                        return p;
                    else if (pl == null)
                        p = pr;
                    else if (pr == null)
                        p = pl;
                    else if ((kc != null ||
                            (kc = comparableClassFor(k)) != null) &&
                            (dir = compareComparables(kc, k, pk)) != 0)
                        p = (dir < 0) ? pl : pr;
                    else if ((q = pr.findTreeNode(h, k, kc)) != null)
                        return q;
                    else
                        p = pl;
                } while (p != null);
            }
            return null;
        }
  }
```

* 一颗树，hash 槽位保存的就是 treeBin

```java
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K, V> root; // 树的根节点
    volatile TreeNode<K, V> first; 
    volatile Thread waiter;
    volatile int lockState;
    static final int WRITER = 1; // set while holding write lock
    static final int WAITER = 2; // set when waiting for write lock
    static final int READER = 4; // increment value for setting read lock

    // 构造方法，可以看出来是构造红黑树的过程
    TreeBin(TreeNode<K, V> b) {
        super(TREEBIN, null, null, null);
        this.first = b;
        TreeNode<K, V> r = null;
        for (TreeNode<K, V> x = b, next; x != null; x = next) {
            next = (TreeNode<K, V>) x.next;
            x.left = x.right = null;
            if (r == null) {
                x.parent = null;
                x.red = false;
                r = x;
            } else {
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                for (TreeNode<K, V> p = r; ; ) {
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h)
                        dir = -1;
                    else if (ph < h)
                        dir = 1;
                    else if ((kc == null &&
                            (kc = comparableClassFor(k)) == null) ||
                            (dir = compareComparables(kc, k, pk)) == 0)
                        dir = tieBreakOrder(k, pk);
                    TreeNode<K, V> xp = p;
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        x.parent = xp;
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        r = balanceInsertion(r, x);
                        break;
                    }
                }
            }
        }
        this.root = r;
        assert checkInvariants(root);
    }

    /** 省略很多代码 */
}
```

###### ForwardingNode

* 这是一个真正的辅助类，该类仅仅只存活在ConcurrentHashMap扩容操作时，如果在扩容时，某个 hash 槽位的节点是 ForwardingNode ，那么就标志着这个节点已经被搬运到了 nextTable，然后就要通过这个节点去 newTable 进行查找，所以这个节点构造方法的参数就是 newTable，起过渡作用。

```java
static final class ForwardingNode<K,V> extends Node<K,V> {
       final Node<K,V>[] nextTable;
       ForwardingNode(Node<K,V>[] tab) {
           super(MOVED, null, null, null);
           this.nextTable = tab;
       }

       Node<K,V> find(int h, Object k) {
           // loop to avoid arbitrarily deep recursion on forwarding nodes
           outer: for (Node<K,V>[] tab = nextTable;;) {
               Node<K,V> e; int n;
               if (k == null || tab == null || (n = tab.length) == 0 ||
                       (e = tabAt(tab, (n - 1) & h)) == null)
                   return null;
               for (;;) {
                   int eh; K ek;
                   if ((eh = e.hash) == h &&
                           ((ek = e.key) == k || (ek != null && k.equals(ek))))
                       return e;
                   if (eh < 0) {
                       if (e instanceof ForwardingNode) {
                           tab = ((ForwardingNode<K,V>)e).nextTable;
                           continue outer;
                       }
                       else
                           return e.find(h, k);
                   }
                   if ((e = e.next) == null)
                       return null;
               }
           }
       }
   }
```



##### 字段定义

```java
// 最大容量
private static final int MAXIMUM_CAPACITY = 1 << 30;
  
// 默认初始容量
private static final int DEFAULT_CAPACITY = 16;
  
// 数组的最大容量,防止抛出OOM
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
  
// 最大并行度，仅用于兼容JDK1.7以前版本
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
  
// 负载因子
private static final float LOAD_FACTOR = 0.75f;
  
// 链表转红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;
  
// 红黑树退化阈值
static final int UNTREEIFY_THRESHOLD = 6;
  
// 达到最小的数组容量链表才会转化成红黑树,
static final int MIN_TREEIFY_CAPACITY = 64;
  
// 扩容搬运时批量搬运的最小槽位数（即一个线程在扩容时每次最少分配这么多个槽位）
private static final int MIN_TRANSFER_STRIDE = 16;
  
// 当前待扩容table的邮戳位,通常是高16位（用这个高 16 位，来标记是否还正在扩容）
private static final int RESIZE_STAMP_BITS = 16;

// 搬运线程数的标识位，通常是低16位
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
  
// 同时搬运的线程数的最大值
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
  
// 下面这几个常量是对应的 NODE#hash
// 如果就是普通的 链表节点 Node，那么其 hash 就是正常计算出来的正数
static final int MOVED     = -1; // 说明是forwardingNode
static final int TREEBIN   = -2; // 红黑树
static final int RESERVED  = -3; // 原子计算的占位Node
static final int HASH_BITS = 0x7fffffff; // 保证hashcode扰动计算结果为正数
  
// 当前哈希表
transient volatile Node<K,V>[] table;
  
// 下一个哈希表（扩容时使用的）
private transient volatile Node<K,V>[] nextTable;
  
// 计数的基准值（记的元素的个数，但只是基准值，基于 cas，失败的话就不会被记进去）
private transient volatile long baseCount;
  
/**
 * 控制变量，在不同的地方有不同的用途，其值也不同，所代表的含义也不同
 *  负数代表正在进行初始化或扩容操作
 *   -1 代表正在初始化
 *   -N 表示有N-1个线程正在进行扩容操作（更准确的说，应该是低 16 位表示的正在扩容的线程数，其值等于 第十六位 - 1
 *  正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小
 */
private transient volatile int sizeCtl;
  
// 并发搬运过程中 CAS 获取区段的下限值，注意在搬运时，transferIndex 是从倒序开始的，这样的目的就是为了 剩余槽位数 和 索引位置+1 相同
// 并发扩容搬运时，因为一个线程一次分配任务都是 stride 个，所以 transferIndex 用来记录当前线程扩容区段的下限值，也可以说是下一个线程获取任务槽位区段的上限值
private transient volatile int transferIndex;
  
// 计数cell初始化或者扩容时基于此字段使用自旋锁
private transient volatile int cellsBusy;
  
// 加速多核 CPU 计数的 cell 数组（用来保存 baseCount 在put增加时，cas 失败元素的个数，采用数组是为了增加并发度）
private transient volatile CounterCell[] counterCells;
```



##### 构造函数

ConcurrentHashMap 在构造函数中并没有做什么事，仅仅只是设置了一些参数而已。其真正的初始化是发生在插入的时候，例如 put、merge、compute、computeIfAbsent、computeIfPresent操作时。

```java
public ConcurrentHashMap() {
}

public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}

public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}

public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```



##### initTable

ConcurrentHashMap的初始化主要由initTable()方法实现，一般在插入的时候，例如put、merge、compute、computeIfAbsent、computeIfPresent操作时如果未初始化则会调用

```java
// 初始化 table，通过对 sizeCtl 的变量赋值来保证数组只能被初始化一次
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // 通过自旋保证一定能初始化成功
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl 小于 0 代表有线程正在初始化，释放当前 CPU 的调度权，重新发起锁的竞争
        if ((sc = sizeCtl) < 0) 
            Thread.yield(); 
        // 运行到这一行表示没有现成正在初始化或者扩容，所以当前线程CAS修改 sizeCtl 为 -1
        // 第一次check，用cas保证了只有一个线程可以进行初始化
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                // 这里是第二次 check，很有可能执行到这里的时候，table 已经不为空了
                if ((tab = table) == null || tab.length == 0) {
                    // 进行初始化,若sizeCtl=0则用16进行初始化
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    // 创建大小为n的数组
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 将 sizeCtl 设置为 n*0.75，表示扩容阈值
                    // 注：这里的逻辑是 1 - 1/2/2 = 1 - 0.25 = 0.75
                    sc = n - (n >>> 2);
                }
            } finally {
                // 若初始化数组失败，则将sizeCtl重置
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```




##### size

为了更好地统计size，ConcurrentHashMap提供了 baseCount 、counterCells 两个辅助变量。

* 定义

```java
@sun.misc.Contended static final class CounterCell {
       volatile long value; // 该CounterCell 统计的元素个数
       CounterCell(long x) { value = x; }
   }

//ConcurrentHashMap中元素个数,但返回的不一定是当前 Map 的真实元素个数（基于CAS无锁更新，所以如果并发更新失败，那么就会把增加的个数保存在 contercell）
private transient volatile long baseCount;

private transient volatile CounterCell[] counterCells;
```

* 方法

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount; // 先初始化为 cas 成功的个数
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            // 遍历，所有counter求和 加在 baseCount 上
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```



##### get

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 计算hash
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
        // 搜索到的节点key与传入的key相同且不为null,直接返回这个节点
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 树
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 链表，遍历
        while ((e = e.next) != null) {
            if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

1、如果当前哈希表table为null

* 哈希表未初始化或者正在初始化未完成，直接返回null；虽然line5和line18之间其它线程可能经历了千山万水，至少在判断tab==null的时间点key肯定是不存在的，返回null符合某一时刻的客观事实。

2、如果读取的bin头节点为null

* 说明该槽位尚未有节点，直接返回null。

3、如果读取的bin是一个链表（说明头节点是个普通Node）。

* 如果正在发生链表向红黑树的treeify工作，因为treeify本身并不破坏旧的链表bin的结构，只是在全部treeify完成后将头节点一次性替换为新创建的TreeBin，可以放心读取。

* 如果正在发生resize且当前bin正在被transfer，因为transfer本身并不破坏旧的链表bin的结构，只是在全部transfer完成后将头节点一次性替换为ForwardingNode，可以放心读取。

* 如果其它线程正在操作链表，在当前线程遍历链表的任意一个时间点，都有可能同时在发生add/replace/remove操作。
  * 如果是 add 操作，因为链表的节点新增从 JDK8 以后都采用了后入式，无非是多遍历或者少遍历一个 tailNode。
  * 如果是 remove 操作，存在遍历到某个 Nod e时，正好有其它线程将其 remove，导致其孤立于整个链表之外；但因为其next引用未发生变更，整个链表并没有断开，还是可以照常遍历链表直到 tailNode。
  * 如果是 replace操作，链表的结构未变，只是某个 Node 的 value 发生了变化，没有安全问题。

* **结论：**对于链表这种线性数据结构，单线程写且插入操作保证是后入式的前提下，并发读取是安全的；不会存在误读、链表断开导致的漏读、读到环状链表等问题。

4.1（hash < 0 )、如果读取的bin是一个ForwardingNode（说明当前bin已迁移，调用其find方法到nextTable读取数据）。

* ForwardingNode中保存了nextTable的引用，会转向下一个哈希表进行检索。

4.2（hash < 0）、如果读取的bin是一个ReserveNode

* ReserveNode 用于 compute / computeIfAbsen t原子计算的方法，在 BIN 的头节点为 null 且计算尚未完成时，先在 bin 的头节点打上一个 ReserveNode 的占位标记。
* 读操作发现ReserveNode直接返回null，写操作会因为争夺ReserveNode的互斥锁而进入阻塞态，在compute完成后被唤醒后循环重试。

4.2（hash < 0 )、如果读取的bin是一个红黑树（说明头节点是个TreeBin节点）。

* 如果正在发生红黑树向链表的untreeify操作，因为untreeify本身并不破坏旧的红黑树结构，只是在全部untreeify完成后将头节点一次性替换为新创建的普通Node，可以放心读取。

* 如果正在发生resize且当前bin正在被transfer，因为transfer本身并不破坏旧的红黑树结构，只是在全部transfer完成后将头节点一次性替换为ForwardingNode，可以放心读取。

* 如果其他线程在写（改变结构）红黑树，在当前线程遍历红黑树的任意一个时间点，都可能有单个的其它线程发生add/replace/remove/红黑树的翻转等操作，参考下面的红黑树的读写锁实现。



###### TreeBin 读写锁

* 状态标志

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015094733590.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  * 为什么写锁和写锁等待只有一位：因为只可能有一个写线程成功进入该 TreeBin 节点，在 ConcurrentHashMap 的所有对某个槽位的写操作都使用Synchronized 锁了那个槽位的Node，所以同一时间即使通过不同的写方法，只要是针对同一槽位的Node，那么肯定就只有一个线程可以去写该节点





```java
TreeNode<K,V> root;
    volatile TreeNode<K,V> first;
    volatile Thread waiter;
    volatile int lockState;
    // 是否有读锁（即：二进制位最后一位，若当前 TreeBin 已经被那个唯一可能存在的写线程成功获取到锁，那么该位为 1）
    static final int WRITER = 1; 
	// 是否有waiter（即：二进制的倒数第二位是否为 1 ，标志着是否存在 waiter。因为存在读锁时要让那个唯一的写线程要阻塞
	// 时 并非把状态设置为waiter，而是把waiter标志位设置为1，读锁释放时也是看 waiter 标志位是否有等待，决定是否唤醒）
    static final int WAITER = 2; 
	// 有线程获取了读锁							 
    static final int READER = 4; 
  
    private final void lockRoot() {
        //如果一次性获取写锁失败，进入contendedLock循环体，循环获取写锁或者休眠等待
        if (!U.compareAndSetInt(this, LOCKSTATE, 0, WRITER))
            contendedLock(); // offload to separate method
    }
  
    private final void unlockRoot() {
        lockState = 0;
    }

    //对红黑树加互斥锁,也就是写锁
    private final void contendedLock() {
        boolean waiting = false;
        for (int s;;) {
            //如果当前 state 为 waiter，可以尝试直接获取写锁
            if (((s = lockState) & ~WAITER) == 0) {
                if (U.compareAndSetInt(this, LOCKSTATE, s, WRITER)) {
                    if (waiting)
                        waiter = null;
                    return;
                }
            }
            // 走到这就说明肯定有线程已经获取了读锁，那么把如果lockState第二位是0,表示当前没有线程在等待写锁
            else if ((s & WAITER) == 0) {
                //将 lockState 的第二位设置为1，相当于打上了waiter的标记，表示有线程在等待写锁
                if (U.compareAndSetInt(this, LOCKSTATE, s, s | WAITER)) {
                    waiting = true;
                    waiter = Thread.currentThread();
                }
            }
            //休眠当前线程
            else if (waiting)
                LockSupport.park(this);
        }
    }
      
    // 查找红黑树中的某个节点
    final Node<K,V> find(int h, Object k) {
        if (k != null) {
            for (Node<K,V> e = first; e != null; ) {
                int s; K ek;
                /**
                 * 如果当前有 waiter 或者有写锁，走线性检索，因为红黑树虽然替代了链表，但其内部依然保留了链表的结构，
                 * 虽然链表的查询性能一般，但根据先前的分析其读取的安全性有保证。
                 *
                 * 发现有写锁改走线性检索，是为了避免等待写锁释放花去太久时间; 
                 * 而发现有waiter说明有且仅有的那一个写线程在等待读锁的释放。之所以已经获取了读锁，为什么还是走线性检索，是为
                 * 了避免读锁叠加的太多，导致写锁线程需要等待太长的时间; 本质上都是为了减少读写碰撞
               	 * 
                 * 线性遍历的过程中，每遍历到下一个节点都做一次判断，一旦发现锁竞争的可能性减少就改走tree检索以提高性能
                 */
                if (((s = lockState) & (WAITER|WRITER)) != 0) {
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    e = e.next;
                }
                //对红黑树加读锁,CAS一次性增加4，也就是增加的只是3~32位
                else if (U.compareAndSetInt(this, LOCKSTATE, s,
                                             s + READER)) {
                    TreeNode<K,V> r, p;
                    try {
                        p = ((r = root) == null ? null :
                             r.findTreeNode(h, k, null));
                    } finally {
                        Thread w;
                        //释放读锁，如果释放完毕且有waiter,则将其唤醒
                        if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                            (READER|WAITER) && (w = waiter) != null)
                            LockSupport.unpark(w);
                    }
                    return p;
                }
            }
        }
        return null;
    }


    // 更新红黑树中的某个节点
    final TreeNode<K,V> putTreeVal(int h, K k, V v) {
        Class<?> kc = null;
        boolean searched = false;
        for (TreeNode<K,V> p = root;;) {
            int dir, ph; K pk;
            //...省略处理红黑树数据结构的代码若干         
                else {
                    //写操作前加互斥锁
                    lockRoot();
                    try {
                        root = balanceInsertion(root, x);
                    } finally {
                        //释放互斥锁
                        unlockRoot();
                    }
                }
                break;
            }
        }
        assert checkInvariants(root);
        return null;
    }
}
```

* 总结：
  * 读写操作是互斥的，允许多个线程同时读取，但不允许读写操作并行，同一时刻只允许一个线程进行写操作；这样任意时间点通过树检索读取的都是一个合法的红黑树。
  * TreeBin在find读操作检索时，在linearSearch（线性检索）和treeSearch（树检索）间做了折衷，前者性能差但并发安全，后者性能佳但要做并发控制，可能导致锁竞争；设计者使用线性检索来尽量避免读写碰撞导致的锁竞争，但评估到 race condition 已消失时，又立即趋向于改用树检索来提高性能，在安全和性能之间做到了极佳的平衡。
  * 由于有线性检索这样一个抄底方案，以及入口处该槽位头节点的 synchornized 机制，保证了进入到 TreeBin 整体代码块的写线程只有一个；所以不定义存放待唤醒线程的 threadQueue，以及存在读写竞争时，读线程仅会自旋使用线性检索而不会阻塞等等， 可以看做是特定条件下ReadWriteLock的简化版本。



##### Traverser 遍历器

遍历器实现的难点在于读操作与 transfer 可能并行，在扫描各个bin时如果遇到 forwadingNode 该如何处理的问题。

* 解决办法是引入了类似于方法调用堆栈的机制，在跳转到 nextTable 时记录下**当前 table 和 已经抵达的槽位 并进行入栈**操作，然后开始**遍历下一个 table 的 i 和 i+n 槽位**，如果遇到 forwadingNode 再一次入栈，周而复始循环往复；
* 每次如果 i+n 槽位如果到了右半段快要溢出的话就会遵循原来的入栈规则进行出栈，也就是回到上一个上下文节点，最终会回到初始的 table 也就是 initialTable 中的节点。

```java
static class Traverser<K,V> {
    Node<K,V>[] tab;        // current table; updated if resized
    Node<K,V> next;         // the next entry to use
    TableStack<K,V> stack, spare; // to save/restore on ForwardingNodes
    int index;              // index of bin to use next
    int baseIndex;          // current index of initial table
    int baseLimit;          // index bound for initial table
    final int baseSize;     // initial table size
  
    Traverser(Node<K,V>[] tab, int size, int index, int limit) {
        this.tab = tab;
        this.baseSize = size;
        this.baseIndex = this.index = index;
        this.baseLimit = limit;
        this.next = null;
    }
  
    /**
     * 返回下一个节点
     */
    final Node<K,V> advance() {
        Node<K,V> e;
        if ((e = next) != null)
            // 取 e.next（即使是 TreeBin 也是线性遍历）
            e = e.next;
        for (;;) {
            Node<K,V>[] t; int i, n;  // 局部变量保证稳定性
            
            if (e != null)
                // 记录下一个要返回的节点
                return next = e;
            
            // 是否已经遍历完了
            if (baseIndex >= baseLimit || (t = tab) == null ||
                (n = t.length) <= (i = index) || i < 0)
                return next = null;
            
            // 挨个从每个槽位去节点后，判断是不是 hash 小于 0 的节点
            if ((e = tabAt(t, i)) != null && e.hash < 0) {
                // 已经搬运过的节点
                if (e instanceof ForwardingNode) {
                    // 让下次便利的 table 等于 nextTable
                    tab = ((ForwardingNode<K,V>)e).nextTable;
                    e = null;
                    // 把 当前的table、索引位置、table 长度 压栈
                    pushState(t, i, n);
                    continue;
                }
                else if (e instanceof TreeBin)
                    e = ((TreeBin<K,V>)e).first;
                else
                    e = null;
            }
            //当前如果有跳转堆栈直接回放
            if (stack != null)
                recoverState(n);
            //没有跳转堆栈说明已经到 initalTable
            else if ((index = i + baseSize) >= n)
                index = ++baseIndex; // visit upper slots if present
        }
    }
  
    /**
     * 遇到ForwardingNode时保存当前上下文
     */
    private void pushState(Node<K,V>[] t, int i, int n) {
        TableStack<K,V> s = spare;  // reuse if possible
        if (s != null)
            spare = s.next;
        else
            s = new TableStack<K,V>();
        s.tab = t;
        s.length = n;
        s.index = i;
        s.next = stack;
        stack = s;
    }
  
    /**
     * 弹出上下文
     *
     */
    private void recoverState(int n) {
        TableStack<K,V> s; int len;
        //如果当前有堆栈，且index已经到达右半段后溢出当前table,说明该回去了
        //如果index还在左半段，则只辅助做index+=s.length操作
        while ((s = stack) != null && (index += (len = s.length)) >= n) {
            n = len;
            index = s.index;
            tab = s.tab;
            s.tab = null;
            TableStack<K,V> next = s.next;
            s.next = spare; // save for reuse
            stack = next;
            spare = s;
        }
        //已经到initialTable,索引自增
        if (s == null && (index += baseSize) >= n)
            index = ++baseIndex;
    }
}
```

##### put

```java
public V put(K key, V value) {
        return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    // 计算hash值：(h ^ (h >>> 16)) & HASH_BITS
    int hash = spread(key.hashCode());
    int binCount = 0; // binCount 有两个作用，第一个是判断是否已经完成插入（bincount > 0)，第二作用是记录链表节点数量
    // 自旋
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        
        // 情况一：table是空的，进行初始化
        // 初始化完后，再走下一轮循环，继续判断
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 情况二：如果当前索引位置没有值，直接创建
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // cas 在 i 位置创建新的元素，当 i 位置是空时，即能创建成功，结束for自旋，否则继续自旋
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 情况三：如果当前槽点是转移节点，表示该槽点正在扩容，就会一直等待扩容完成，之后再for进行新增
        // 转移节点的 hash 值是固定的，都是 MOVED
        else if ((fh = f.hash) == MOVED)
            // 帮助扩容（说白了就是按照字段定义那介绍的 transferIndex，看还能不能领一段槽位的任务，
            // 然后再把 transferIndex 减去领取的任务槽位数）
            // *** 注意：这里是拿到 newTable，因为已经正在扩容了，那肯定使用的扩容完后的 newTable 进行插入 ***
            tab = helpTransfer(tab, f);
        // 情况四：槽点上有值的，即出现Hash碰撞。下面要分成链表和红黑树两种情况
        else { // 第一次 check
            V oldVal = null;
            // 锁定当前槽点，其余线程不能操作，保证了安全
            synchronized (f) {
                // 第二次 check，判断 i 索引位置的数据没有被修改，因为可能在 synchrnozied 阻塞的时候这个槽位的值被修改
                if (tabAt(tab, i) == f) {
                    // 当前槽点上是链表
                    if (fh >= 0) {
                        binCount = 1; // binCount 计数这个链表的长度，后面判断要不要树化
                        // 遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 判断链表中是否已经有要新增的key
                            if (e.hash == hash &&
                                ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                // 获取oldValue
                                oldVal = e.val;
                                // 根据参数onlyIfAbsent判断是否覆盖
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break; // 退出链表遍历
                            }
                            // 链表中不存在要新增key，则将该node插入到链表最后
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break; // 退出链表遍历
                            }
                        }
                    }
                    // 红黑树，即 treebin
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2; // 标志被处理了
                        // 满足if的话，把老的值给oldVal
                        // 在putTreeVal方法里面，在给红黑树重新着色旋转的时候会锁住红黑树的根节点
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            
            // binCount 不为 0 说明已经新增成功了（因为还有一种情况，就是当前这个节点是正在迁移，就要拿到newTable然后插入）
            if (binCount != 0) {
                // 判断链表是否需要转化成红黑树，链表节点数已经大于8
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                // 完成插入，退出循环
                break;
            }
        }
    }
    
    // 增加计数，同时判断容器是否需要扩容，如果需要去扩容，调用 transfer 方法去扩容
    // bincount 用来标志是否要进行扩容检查
    //  if <0, don't check resize, if <= 1 only check if uncontended
    //  <= 1 对于上面有两种情况，如果是插入的槽位之前为空，那么 bincount = 0，如果插入前链表只有一个节点，那么bincount = 1
    addCount(1L, binCount);
    return null;
}
```

* 过程
  * 判空；ConcurrentHashMap的key、value都不允许为null
  * 计算hash，初始化结束标志 bincount。
  * 遍历table，进行节点插入操作，过程如下：
    * 如果table为空，则表示ConcurrentHashMap还没有初始化，则进行初始化操作：initTable()
    * 根据hash值获取节点的位置i，若该位置为空，则直接插入，这个过程是不需要加锁的。计算f位置：i=(n - 1) & hash
    * 如果检测到fh = f.hash == -1，则f是ForwardingNode节点，表示有其他线程正在进行扩容操作，则帮助线程一起进行扩容操作，然后等扩容结束，拿到 newTable ，再在循环里对 newTable 进行插入
    * 走到这说明是 链表或者树了，会对当前节点加锁，获取锁后还会进行**doubleCheck**
      * 如果f.hash >= 0 表示是链表结构，则遍历链表，如果存在当前key节点则替换value，否则插入到链表尾部。
      * 如果f是TreeBin类型节点，则按照红黑树的方法更新或者增加节点
      * 若链表长度 > TREEIFY_THRESHOLD(默认是8)，则将链表转换为红黑树结构
  * 调用addCount方法，ConcurrentHashMap的size + 1，同时判断是否扩容



##### 扩容

###### addCount

检查是否要扩容

```java
/**
 * 增加元素数量，并检查是否需要扩容
 */
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 如果计数盒子不是空 或着 修改 baseCount 失败
   if ((as = counterCells) != null ||
       !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
            
        if (as == null || // 如果计数盒子时空（尚未出现并发）
         (m = as.length - 1) < 0 || (a = as[ThreadLocalRandom.getProbe() & m]) == null || // 如果随机取余一个数组位置为空
            !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) { // 修改这个槽位的变量失败了（出现并发了）
                
           // 执行fullAddCount方法，并结束
           fullAddCount(x, uncontended);
           return;
       }
        if (check <= 1)
            return;
        s = sumCount();
    }
        
    // 如果需要检查，检查是否需要扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 如果 map.size()大于 sizeCtl（即达到扩容阈值）&& table非空 && table 长度小于最大容量
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
              (n = tab.length) < MAXIMUM_CAPACITY) {
            // 根据 length 得到一个扩容标识（即后面要设置 sizeCtl 的高 16 位），
            // 根据长度获取这 16 位还有一个好处就是可以标识出分代数
            int rs = resizeStamp(n);
            // sc小于0代表有线程在扩容
            if (sc < 0) {
                // 如果 sc 的高 16 位不等于当前长度算出的标识了，说明已经不是为这个长度在执行扩容
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || 
                   // 如果 sc == 标识符 + 1 （扩容结束了，不再有线程进行扩容）（因为默认第一个线程设置 sc == rs 左移 16 
                   // 位 + 2，当第一个线程结束扩容了，就会将 sc 减 1。这个时候，sc 就等于 rs + 1）
                   sc == rs + 1 || 
           		   // 如果 sc == 标识符 + 65535（帮助线程数已经达到最大）
                   sc == rs + MAX_RESIZERS || 
                   // 如果 nextTable == null（结束扩容了）
                   (nt = nextTable) == null || 
                   // 如果 transferIndex <= 0 (扩容结束了，已经没有要搬运的槽位了)
                   transferIndex <= 0) 
                    break;
				// 如果可以帮助扩容，那么将 sc 加 1. 表示多了一个线程在帮助扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            /**
             * 下面这个说明是是第一个执行扩容的线程
             *
             * 可以从 cas 看出，是把要扩容时的长度算出来 作为当前这一代的标识放到 sizeCtl 的高 16 位，低 16 位扩容线程个数
             * 初始化为 2。
             *   首个线程设置初始值为2的原因是：线程退出时会通过CAS操作将参与搬运的总线程数-1，如果初始值按照常规做法设置成
             *   1，那么减1后就会变为0。此时其它线程发现线程数为0时，无法区分是没有任何线程做过搬运，还是有线程做完搬运但都退
             *   出了，也就无法判断要不要加入搬运的行列。
             */
            else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))
             // 更新 sizeCtl 为负数后，开始扩容。
                transfer(tab, null); 
            s = sumCount();
        }
    }
}
```



###### transfer

搬运（扩容）

```java
private final void transfer(Node<K, V>[] tab, Node<K, V>[] nextTab) {
    int n = tab.length, stride;
    // <1> 根据长度和CPU的数量计算步长（一个线程一次搬运多少个槽位），最小是16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE;
    
    // <2> 首个搬运线程，负责初始化 nextTable 和 transferIndex
    if (nextTab == null) {          
        try {
            // 初始化 nt，
            Node<K, V>[] nt = (Node<K, V>[]) new Node<?, ?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {    
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        // 初始化要搬运索引的起始位置，是倒序（也是剩余要搬运的槽位数）
        transferIndex = n;
    }
    
    // <3> 开始搬运前准备工作
    int nextn = nextTab.length;
    // 公共的forwardingNode（用于标识那个槽位已经搬运到了新节点）
    ForwardingNode<K, V> fwd = new ForwardingNode<K, V>(nextTab);
    boolean advance = true; // 是否取到了要搬运槽位的索引
    boolean finishing = false; // 保证提交 nextTable 之前已遍历旧表的所有槽位
    
    // <4> 开始在循环里面搬运
    for (int i = 0, bound = 0; ; ) {
        Node<K, V> f;
        int fh;
        /**
         * <4.1> 循环CAS获取下一个搬运区段
         * 流程：
         *   如果是第一次开始搬运，那么会让 nextIndex = transferIndex，然后让 nextBound = nextIndex - stride，
         *   再把 transferIndex cas为 nextBound ，之后会让 i = nextIndex - 1，bound = nextBound，
         *   那么当前线程这次搬运的槽位就是[bound , i] = [transferIndex - stride , transferIndex - 1]
         *
         *   初始化过 bound 之后，如果下次来的时候， -- i >= bound ，就是说搬运任务还没执行完那就继续去搬运前一个槽位
         *   否则执行完了，就按上面那样再次用最新的 transferIndex 去计算 bound 和 i。
         */
        while (advance) {
            int nextIndex, nextBound;
            // 索引 - 1（因为是倒序），并判断搬运任务区段是否结束
            if (--i >= bound || finishing)
                advance = false;
            // 全部槽位是否已经搬运完毕
            else if ((nextIndex = transferIndex) <= 0) {
                // 把 i 设为 -1，好让在下面执行退出操作
                i = -1;
                advance = false;
            // 是否领取任务区段成功，通过 cas 保证一个区段只能被一个线程领取
            } else if (U.compareAndSwapInt
                    (this, TRANSFERINDEX, nextIndex,
                            nextBound = (nextIndex > stride ?    // 获取此次任务区段的边界
                                    nextIndex - stride : 0))) {
                  
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        
        // <4.2> 已经拿到了要搬运的槽位 i，下面执行搬运
        // 是否已经没有任务区段
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) { // 执行 finishing 操作的一定是最后一个线程重新全部检查完一遍后，然后从这里退出
                nextTable = null;
                table = nextTab; // 指向 nextTable
                sizeCtl = (n << 1) - (n >>> 1); // 修改扩容后的阀值，*2-0.25即现在容量的0.75倍
                return;
            }
            // 执行搬运任务的线程数 -1 
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 并非最后一个退出的线程
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;        
                // 是最后一个完成搬运任务的线程，那说明所有搬运任务都执行完了，那把 finishing 设为 true
                finishing = advance = true;
                // 异常巧妙的设计，最后一个线程退出前将 i 回退到最高位，等于是强制做最后一次的全表扫描（因为最后完成的任务区
                // 段，其 bound 一定是 0，因为是把槽位倒序的划分成区段分配给线程） ；所以会把 table 再从头遍历一遍，
                // 看哪个不是 ForwadingNode，可以视为抄底逻辑，虽然检测到漏掉槽位的概率基本是0
                i = n;
            }
          // 空槽位
        } else if ((f = tabAt(tab, i)) == null)
            // 空槽位直接打上forwardingNode标记，CAS失败下一次循环继续搬运该槽位，成功则进入下一个槽位
            advance = casTabAt(tab, i, null, fwd);
        // 如果这个槽位已经被搬运
        else if ((fh = f.hash) == MOVED)
            advance = true; 
        // 非空、且未搬运
        else {
            // 锁定该槽位的头节点
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K, V> ln, hn;
                    if (fh >= 0) {
                        //省略链表搬运代码:职责是将链表拆成两份,搬运到 nextTable 的 i 和 i+n 槽位（同 HashMap#resize）
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        //设置旧表对应槽位的头节点为forwardingNode
                        setTabAt(tab, i, fwd);
                        advance = true;
                    } else if (f instanceof TreeBin) {
                        //省略红黑树搬运代码:职责是将红黑树拆成两份,搬运到nextTable的i和i+n槽位（同 TreeNode#split）
                        //如果满足红黑树的退化条件，顺便将其退化为链表
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        //设置旧表对应槽位的头节点为forwardingNode
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```



###### helpTransfer

当一个线程要对当前槽点进行put操作，但该槽点已经被置为forwardingNode，即该节点已经被拷贝处理过了，需等到全部槽点完成拷贝拷贝才能进行put。因此，在这段等待时间内该线程会helpTransfer（协助扩容)

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    
    // 尝试帮助扩容判断
    if (tab != null &&  // 如果 table 不是空 
     (f instanceof ForwardingNode) && // 且 node 节点是转移类型，数据检验
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) { // 且 node 节点的 nextTable（新 table） 不空
        // 得到 length 的标识
        int rs = resizeStamp(tab.length);
        
        while (nextTab == nextTable && // 如果 nextTab 没有被并发修改 
         table == tab && // 且 tab 也没有被并发修改
               (sc = sizeCtl) < 0) { // 且 sizeCtl  < 0 （说明还在扩容）

            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || // 如果不是给这个 lenth 扩容
             sc == rs + 1 || // 扩容结束了，已经没有线程执行扩容
                sc == rs + MAX_RESIZERS ||  // 如果达到最大帮助线程的数量，即 65535
                 transferIndex <= 0) // 扩容结束，所有槽位已经搬运完
                // 结束循环，返回 table
                break;
            
            // 如果以上都不是, 将 sizeCtl + 1, （表示增加了一个线程帮助其扩容）
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                // 进行转移
                transfer(tab, nextTab);
                // 结束循环
                break;
            }
        }
        return nextTab;
    }
    return table;
}

```


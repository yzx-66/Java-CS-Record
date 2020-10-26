#### PriorityBlockingQueue

* 一个**支持优先级**的**无界**阻塞队列。默认情况下元素采用自然顺序升序排序，当然我们也可以通过构造函数来指定Comparator来对元素进行排序。需要注意的是PriorityBlockingQueue不能保证同优先级元素的顺序。
* 底层实现是数组，采用的堆的思想（最小堆），保证数组第零个元素是最小的元素，但是注意，其增加和删除元素的调整方法并非是堆排的调整方法，即不用满足最小堆的堆排所要求的左子节点小于右子节点，只用满足根元素是最小的元素。并发安全也是通过 Lock 保证，但扩容那里也使用了 CAS。



##### 属性

```java
public class PriorityBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable
    
    // 默认容量
    private static final int DEFAULT_INITIAL_CAPACITY = 11;

    // 最大容量
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    // 二叉堆数组
    private transient Object[] queue;

    // 队列元素的个数
    private transient int size;

    // 比较器，如果为空，则为自然顺序
    private transient Comparator<? super E> comparator;

    // 内部锁
    private final ReentrantLock lock;
	// 只有非空等待，因为二叉堆是无界的，会扩容
    private final Condition notEmpty;

    //
    private transient volatile int allocationSpinLock;

    // 优先队列：主要用于序列化，这是为了兼容之前的版本。只有在序列化和反序列化才非空
    private PriorityQueue<E> q;

```



##### 入队

* 只介绍put，其余add、offer 略

```java
public void put(E e) {
    offer(e); // never need to block
}

public boolean offer(E e) {
    // 不能为null
    if (e == null)
        throw new NullPointerException();
   
    final ReentrantLock lock = this.lock;
    // 获取锁
  	ck.lock();
    
    int n, cap;
    Object[] array;
    
    // 当 size > queue.lenth
    // 放在 while 循环里是因为扩容时用了 cas 锁，但是没有循环，所以在这里进行了自旋
    while ((n = size) >= (cap = (array = queue).length))
      // 扩容
      tryGrow(array, cap);
    
    try {
        Comparator<? super E> cmp = comparator;
        // 根据比较器是否为null，做不同的处理
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp);
        // 增加 size
        size = n + 1;
        
        // 唤醒正在等待的消费者线程
        notEmpty.signal();
    } finally {
        // 释放锁
        lock.unlock();
    }
    return true;
}

/**
 * 扩容
 */
private void tryGrow(Object[] array, int oldCap) {
	// 扩容操作使用自旋，不需要锁主锁，释放（因为下面只是创建一个新数组，并不会操作原数组）
    lock.unlock();      
    
    Object[] newArray = null;
    
    // CAS 锁（原理就是 cas 为另一个值后，如果不再结束时改为原先的值，那么其他其他线程就一直无法 cas 成功）
    if (allocationSpinLock == 0 && UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset, 0, 1)) {
        try {
            // 新容量 
            // oldCpa < 64: newCap = 2 OldCap + 2、
            // oldCap >= 64: newCap = 1.5 oldCap
            int newCap = oldCap + ((oldCap < 64) ? (oldCap + 2) :  (oldCap >> 1));

            // 超过
            if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                int minCap = oldCap + 1;
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;        // 最大容量
            }
            
            if (newCap > oldCap && queue == array)
                // 创建新数组
                newArray = new Object[newCap];
        } finally {
            allocationSpinLock = 0;     // 扩容后allocationSpinLock = 0 还原为原先的值（代表释放了自旋锁）
        }
    }
    
    // 到这里如果是本线程扩容newArray肯定是不为null，为null就是其他线程在处理扩容，那就让给别的线程处理
    if (newArray == null)
        Thread.yield();
    
    // 主锁获取锁，因为下面要进行老数组的拷贝了，所以要获取锁
    lock.lock();
    
    // 数组复制
    if (newArray != null && queue == array) {
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}

/**
 * 当比较器不为null时，采用所指定的比较器
 *
 * k：数组里面元素个数
 * x：要插入的元素
 * array：要插入的数组
 */
private static <T> void siftUpComparable(int k, T x, Object[] array) {
    Comparable<? super T> key = (Comparable<? super T>) x;
    
    // 寻找 x 插入的位置（其思想是，假定 x 插入在数组最后一个位置，然后一直跟父节点比较，如果 >= 父节点那么就可以作其子节点，
    // 否则就和父节点交换，然后继续再往上找）
    while (k > 0) {
        // 父级节点
        int parent = (k - 1) >>> 1;
        Object e = array[parent];

        // key >= parent 那么就可以做 parent 的子节点，安插在这个位置即可
        if (key.compareTo((T) e) >= 0)
            break;
        // key < parant 就替换接着往上比
        array[k] = e;
        k = parent;
    }
    array[k] = key;
}

/**
 * 当比较器不为null时，采用所指定的比较器
 */
private static <T> void siftUpUsingComparator(int k, T x, Object[] array,
                                   Comparator<? super T> cmp) {
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = array[parent];
        if (cmp.compare(x, (T) e) >= 0)
            break;
        array[k] = e;
        k = parent;
    }
    array[k] = x;
}
```



##### 出队

```java
public E poll() {
     final ReentrantLock lock = this.lock;
     lock.lock();
     try {
         return dequeue();
     } finally {
         lock.unlock();
     }
 }
 
private E dequeue() {
     // 没有元素 返回null
     int n = size - 1;
     if (n < 0)
         return null;
     else {
         Object[] array = queue;
         // 出队元素（第 0 号位置的最小元素）
         E result = (E) array[0];
         
         // 最后一个元素（也就是假定放到被删除位置的元素）
         E x = (E) array[n];
         array[n] = null;
         
         // 根据比较器释放为null，来执行不同的处理
         Comparator<? super E> cmp = comparator;
         if (cmp == null)
             siftDownComparable(0, x, array, n);
         else
             siftDownUsingComparator(0, x, array, n, cmp);
         size = n;
         return result;
     }
 }

/**
 * 如果比较器为null，则调用siftDownComparable来进行自然排序处理
 *
 * k:被删除节点位置索引
 * x：数组最后一个元素（此位置已经是 null 了，因为删了一个前面的元素，所以会找到一个新位置来放这个元素）
 * array：要调整的数组
 * n：数组元素个数（不包括 x 所在的最后一个位置了）
 */
private static <T> void siftDownComparable(int k, T x, Object[] array,
                                           int n) {
    if (n > 0) {
        Comparable<? super T> key = (Comparable<? super T>)x;   
        // 最后一个叶子节点的父节点位置
        int half = n >>> 1;
        
        // 思想是假定把 x 放到了被删除的位置，然后获得其左右节点节点中最小的值，如果 x 小于这个值，那么 x 就可以放在这，
        // 否则交换左右子节点中最小的交换位置，然后接着再和子节点比较，直到找到位置
        while (k < half) {
            int child = (k << 1) + 1;       // 待调整位置左节点位置
            Object c = array[child];        //左节点
            int right = child + 1;          //右节点

            //左右节点比较，取较小的作为 c
            if (right < n &&
                   ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                c = array[child = right];

            //如果待调整key最小，那就退出，直接赋值
            if (key.compareTo((T) c) <= 0)
                break;
            
            //如果key不是最小，那就取左右节点小的那个放到调整位置，然后小的那个节点位置开始再继续调整
            array[k] = c;
            k = child;
        }
        array[k] = key;
    }
}

/**
 * 如果指定了比较器，则采用比较器来进行调整
 */
private static <T> void siftDownUsingComparator(int k, T x, Object[] array,
                                                int n,
                                                Comparator<? super T> cmp) {
    if (n > 0) {
        int half = n >>> 1;
        while (k < half) {
            int child = (k << 1) + 1;
            Object c = array[child];
            int right = child + 1;
            if (right < n && cmp.compare((T) c, (T) array[right]) > 0)
                c = array[child = right];
            if (cmp.compare(x, (T) c) <= 0)
                break;
            array[k] = c;
            k = child;
        }
        array[k] = x;
    }
}
```

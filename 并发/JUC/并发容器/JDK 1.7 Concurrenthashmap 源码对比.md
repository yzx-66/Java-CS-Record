#### Concurrenthashmap1.8和1.7版本实现原理 

JDK 1.7： ReentrantLock + Segment + HashEntry 分段锁思想（简单点说就是 Map 的所有槽位并不分布在一个 table 上，而是分布在多个 table 上，每个 table 就是一个分段，并且锁的颗粒度不再是槽位，而是一个分段的 table）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015094827949.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)





**hash 槽只有 HashEntry 一种类型，因为不会树化所以没有树相关节点，并且因为是分段锁，锁的是一个包含部分槽位的 table，所以也不会出现扩容和读写同时发生**

```java
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;
		
    // 其余略
}
```



**每个 Segment 都是一个分段，其包含了该分段对应的一个 table，同时又继承了 ReentrantLock，所以在有写操作时可以直接获取 lock ，然后独占该分段的 table**

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {

    private static final long serialVersionUID = 2249069246763182397L;

    static final int MAX_SCAN_RETRIES =
        Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

    // 该分段对应的槽位数组
    transient volatile HashEntry<K,V>[] table;
	// 该分段的元素总个数
    transient int count;
	// 修改次数
    transient int modCount;
	// 扩容阈值
    transient int threshold;
	// 负载因子
    final float loadFactor;

    Segment(float lf, int threshold, HashEntry<K,V>[] tab) {
        this.loadFactor = lf;
        this.threshold = threshold;
        this.table = tab;
    }
    // 具体方法不在这里讨论...
}
```



**构造方法，计算出 segment 数组大小， 其等于 concurrencyLevel，默认为 16 ，然后创建 segment 数组，并初始化第一个 segment，以后其余 segment 初始化的时候会创建跟第一个 segment 大小相同的 table 数组，其余的懒初始化**

```java
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
    
        // 对于concurrencyLevel的理解, 可以理解为segments数组的长度，即理论上多线程并发数(分段锁), 默认16
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        
    	// segment 大小的 二进制位时 1 的偏移
    	// 比如 concurrencyLevel 是 17 ，那么 ssize = 32 ，
    	// 二进制为 0000 0000 0000 0000 0000 0000 0010 0000，所以 sshift 的值为 1 左移的次数，即 5
        int sshift = 0;
    	// segment 的大小，必须是 2 的 n 次 幂
        int ssize = 1;
    
    	// 计算 sszie 和 sshift
        // 默认concurrencyLevel = 16, 所以 ssize 在默认情况下也是16, 此时 sshift = 4
        // ssize = 2 ^ sshift ，即 ssize = 1 << sshift
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }
    
        // 段偏移量，32 是因为 hash 是 int 值，int 值 32 位，默认值情况下此时 segmentShift = 28
    	// 该值会在计算完 hash 值后，把 hash 右移 segmentShift 位，再和 segmentmask 取模算出 segment 索引
        this.segmentShift = 32 - sshift;
        // 散列算法的掩码，默认值情况下 segmentMask = 15,
	    // 定位segment 的时候需要根据 segment[] 长度取模, 即 hash & (ssize - 1) = hash & segmentMask
        this.segmentMask = ssize - 1;
    
    
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
    
        // 计算每个 segment 中 table 的容量。
	    // 初始容量=16, 并发数=16, 则 segment 中的 Entry[] 长度为1。
        int c = initialCapacity / ssize;
        // 处理无法整除的情况，取上限
        if (c * ssize < initialCapacity)
            ++c;
   		// 同样要求 segment 中 table 的长度为 2 的 n 次幂
        // MIN_SEGMENT_TABLE_CAPACITY 默认是2，cap是2的n次方
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)
            cap <<= 1;
    
        // create segments and segments[0]
        // 创建 segments 数组，并初始化第一个 segment,其余的segment延迟初始化
        Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             // 以后其他 segment 初始化的时候，会以第一个 segment 里 table 的长度来创建数组
                             (HashEntry<K,V>[])new HashEntry[cap]); 
    	// 创建 segment 数组
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
	    // 初始化 segment 数组第一个 segment
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    
    
        this.segments = ss;
    }
```



**put 方法，先计算在那个 segment ，通过对 key 的 hashcode 再计算一次 hash，然后因为 int 值的低位总是比高位先填满的，为保证元素在 segment 级别分布的尽量均匀，计算元素所在segment时，总是取 hash 值的高位进行计算，即 hash >>> segmentShift 位后的高位 和 sementMask 按位与 得到 segment 的索引。如果这个segment 还没有初始化的话，那么先初始化，然后调用该 segment 的 put 方法增加元素**

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    
    // 根据key的hash再次进行hash运算
    int hash = hash(key.hashCode());
    
    // 基于 hash 定位segment数组的索引。
    // hash 值是 int 值，32 bits。segmentShift = 28，无符号右移 28 位，剩下高 4 位，其余补 0。
    // segmentMask = 15，二进制低 4 位全部是 1 ，所以 j 相当于 hash 右移后的低 4 位。
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
    	// 该 segment 为 null，则初始化并返回
        s = ensureSegment(j);

    // 将新节点插入segment中
    return s.put(key, hash, value, false);
}

```

在 put 的时候会一直获取该 segment 的 lock，然后拿到锁之后会计算 index，就是通过 hash 按位与 table.lenth，然后就是常规操作，遍历这个 table 槽位的 hashEntry 链表，如果不存在的话，就会添加一个节点，然后会判断这个 table 要不要扩容，扩容也很简单，segment 永远不会扩容，只对该 segment 的 table 扩容，是扩大二倍，然后再挨个重新计算 hash 放到新的 table 里

```java
/**
 * segment#put
 */
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 尝试获取锁
    HashEntry<K,V> node = tryLock() ? null :
    	// 失败会 scanAndLockForPut （该方法会先尝试 自旋 + tryLock，如果自旋到一定次数会调用 lock() 进行阻塞）
        scanAndLockForPut(key, hash, value); 
    
    V oldValue;
    try {
       HashEntry<K,V>[] tab = table;
        // 计算在 table 中的索引
       int index = (tab.length - 1) & hash;
        
       // 获取到bucket位置的第一个节点
       HashEntry<K,V> first = entryAt(tab, index);
        
       for (HashEntry<K,V> e = first;;) {
           // hash冲突
           if (e != null) {
               K k;
               // key相等则覆盖
               if ((k = e.key) == key ||
                   (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                   if (!onlyIfAbsent) {
                       e.value = value;
                        ++modCount;
                    }
                    break;
                }
                // 不相等则遍历链表
                 e = e.next;
              }
           else {
                 if (node != null)
                    // 将新节点插入链表作为表头
                    node.setNext(first);
                else
                    // 创建新节点并插入表头
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                 // 判断元素个数是否超过了阈值或者segment中数组的长度超过了MAXIMUM_CAPACITY，如果满足条件则rehash扩容！
                 if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                     // 扩容（Segment 不扩容，扩容下面的table数组，每次都是将数组翻倍）
                     rehash(node);
                 else
                     setEntryAt(tab, index, node);
                 ++modCount;
                 count = c;
                 oldValue = null;
                 break;
             }
       }
    } finally {
       // 解锁
       unlock();
    }
    return oldValue;
}
```



**get 方法，其和 put 找该 key 槽位的步骤一样，都是把 hashcode 再 hash 一次，然后右移 segmentShift 位得到高位 和 segmentMask 取模，然后就可以取到 segment，再用 hash 和 table.lenth 按位与就得到具体的槽位了。get 方法是不会获取锁，因为线性检索是不会有并发问题，只是一种弱一致性，如果其他线程可能对链表结构做了调整，那么就可能读到旧值**

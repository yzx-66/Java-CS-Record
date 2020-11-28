# HashMap

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200929164353968.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



## 数据结构

```java
// 底层存储的数组
transient Node<K,V>[] table;

// 调用 `#entrySet()` 方法后的缓存
transient Set<Map.Entry<K,V>> entrySet;

//  key-value 的键值对（映射）数量
transient int size;

//  HashMap 的修改次数
transient int modCount;

/**
 * 阀值
 * 
 * 如果 table 已经初始化过，当size 如果超过 threshold 则将扩容，threshold = newCap * loadFactor
 *
 * 如果 table == null，则 threshold 的值将成为 newCap 去初始化table
 *   1、threshold 要是大于等于 initialCapacity 的最小 2 的 N 次方，那么 table 的长度就会一直保证
 *		是 2 的 N 次方，因为扩容也是把 table 扩大一倍，所以仍是 2 的 N 次方
 *	 2、为什么 threshold 要是大于等于 initialCapacity 的最小 2 的 N 次方：因为在 put 方法中，
 *	    计算 table 数组对应的位置，逻辑是 (n - 1) & hash ，此时(n - 1) 的二进制每一位都会是 1，
 *	    那么在和 hash 按位与的时候就会更分散，如果存在(n - 1) 的二进制位存在0 那么和 hash 按位与
 *	    的时候该位会一直是 0.
 */
int threshold;

// 负载因子
final float loadFactor;
```

- Node

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200929165421227.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  ```java
  static class Node<K,V> implements Map.Entry<K,V> {
  
      final int hash;
  
      final K key;
  
      V value;
  
      /**
       * Node及子类都会使用该属性，因为 table 的每个槽位要提供一个可以获取到该槽位所有元素的方法，
       * 并且不受子类实现的影响，所以 TreeNode 即使是树结构也会用 next 把所有节点也串联起来，
       * 比如在writeObject()、readObject() 中都是通过 next 属性来获取该槽位所有键值对。
       */
      Node<K,V> next;
  
      // ... 省略实现方法
  
  }
  ```

  - TreeNode ：定义在 HashMap 中，红黑树节点，可以实现 `table` 数组的每一个位置可以形成红黑树。

	![在这里插入图片描述](https://img-blog.csdnimg.cn/20200929164435290.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




## 构造方法

在该构造方法上，不会对 `table` 数组的初始化。它是**延迟**初始化，在我们开始往 HashMap 中添加 key-value 键值对时，在 `#resize()` 方法中才真正初始化。

- 默认

```java
// 默认的初始化容量
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 默认加载因子为 0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;

public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```



- 指定初始容量

  注意：该 initialCapacity 只是用来计算 threshold 的，并没有一个 capacity 属性来保存该值

```java
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```



- 指定初始容量和负载因子

```java
// 最大的容量为 2^30 。
static final int MAXIMUM_CAPACITY = 1 << 30;

public HashMap(int initialCapacity, float loadFactor) {
    // 校验 initialCapacity 参数
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    // 避免 initialCapacity 超过 MAXIMUM_CAPACITY
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    // 校验 loadFactor 参数
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    // 设置 loadFactor 属性
    this.loadFactor = loadFactor;
    // 计算 threshold 阀值
    this.threshold = tableSizeFor(initialCapacity);
}

/**
 * 返回大于 cap 的最小 2 的 N 次方。
 * 例如：cap = 10 时返回 16 ，cap = 32 时返回 32
 */
static final int tableSizeFor(int cap) {
    /**
     * -1 = 1111 1111 1111 1111 1111 1111 1111 1111B
     * 例如 cap = 8
     *	Integer.numberOfLeadingZeros(8 - 1) = Integer.numberOfLeadingZeros(111B) = 29
     *  -1 >>> 29 = 0000 0000 0000 0000 0000 0000 0000 0111B 
     * 
     * 即：把 cap-1 的每一个二进制位都变成了 1 
     */
     // - 1 是为了防止 cap 刚好已经是 2 的 N 次方，那会导致再扩大一倍
    int n = -1 >>> Integer.numberOfLeadingZeros(cap - 1);
    
    // n + 1 之后刚好是 2 的 N 次方 ，因为 n 的二进制位都是 1 （11111.....)
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```



- 指定一个Map

```java
public HashMap(Map<? extends K, ? extends V> m) {
    // 设置加载因子
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    // 批量添加到 table 中
    putMapEntries(m, false);
}

final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    // <1>
    if (s > 0) {
        // 如果 table 为空，说明还没初始化，适合在构造方法的情况
        if (table == null) { // pre-size
            // 根据 s 的大小 + loadFactor 负载因子，计算需要最小的 tables 大小
            // + 1.0F 的目的，是因为下面 (int) 直接取整，避免不够。
            float ft = ((float)s / loadFactor) + 1.0F; 
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            
            // 如果 t 大于阀值，则使用该 t ，并变成大于 t 的最小 2 的 N 次方赋给 threshold 
            // 然后将在后面的 putVal() 的 resize() 扩容中使用其初始化 table
            if (t > threshold)
                threshold = tableSizeFor(t);
            
        // 如果 table 非空，说明已经初始化，需要不断扩容到阀值超过 s 的数量，避免 putVal 时再扩容
        } else {
            // 要放在 while 中是为了防止扩容一次还不够
            while (s > threshold && table.length < MAXIMUM_CAPACITY)
                resize(); // 扩容
        }

        // <2> 遍历 m 集合，逐个添加到 HashMap 中。
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```



## 添加元素

- 添加单个元素

```java
public V put(K key, V value) {
    // hash(key) 计算哈希值
    return putVal(hash(key), key, value, false, true);
}

/**
 * 哈希函数
 */
static final int hash(Object key) {
    int h;
    // h = key.hashCode() 计算哈希值
    // ^ (h >>> 16) 高 16 位与自身进行异或计算，保证计算出来的 hash 更加离散
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// 每个位置链表树化成红黑树，需要的链表最小长度
static final int TREEIFY_THRESHOLD = 8;

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; // tables 数组
    Node<K,V> p; // 对应位置的 Node 节点
    int n; // 数组大小
    int i; // 对应的 table 的位置
    
    // <1> 如果 table 未初始化，或者容量为 0 ，则进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize() /*扩容，该处的 newCap 是 threshold */ ).length;
    
    // <2.1> 如果对应位置的 Node 节点为空，则直接创建 Node 节点即可。
    if ((p = tab[i = (n - 1) & hash] /*获得对应位置的 Node 节点*/) == null)
        tab[i] = newNode(hash, key, value, null);
    // <2.2> 如果对应位置的 Node 节点非空，则可能存在哈希冲突
    else {
        Node<K,V> e; // key 在 HashMap 对应的老节点
        K k;
        // <2.2.1> 如果找到的 p 节点，就是要找的，则则直接使用即可
        if (p.hash == hash && // 判断 hash 值相等
            ((k = p.key) == key || (key != null && key.equals(k)))) // 判断 key 真正相等
            e = p;
        // <2.2.2> 如果找到的 p 节点，是红黑树 Node 节点，则直接添加到树中
        else if (p instanceof TreeNode)
            // 红黑树的所有操作都封装在 TreeNode 中
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // <2.2.3> 如果找到的 p 是 Node 节点，则说明是链表，需要遍历查找
        else {
            // 顺序遍历链表
            for (int binCount = 0; ; ++binCount) {
                // e = p.next：e 指向下一个节点，因为上面我们已经判断了最开始的 p 节点。
                // 如果已经遍历到链表的尾巴，则说明 key 在 HashMap 中不存在，则需要创建
                if ((e = p.next) == null) {
                    // 创建新的 Node 节点
                    p.next = newNode(hash, key, value, null);
                    // 链表的长度如果数量达到 TREEIFY_THRESHOLD（8）时，则进行树化。
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break; // 结束
                }
                // 如果遍历的 e 节点，就是要找的，则则直接使用即可
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break; // 结束
                // p 指向下一个节点
                p = e;
            }
        }
        // <3> 如果找到了对应的节点
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // 修改节点的 value ，如果允许修改
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 节点被访问的回调，HsahMap 为空实现
            afterNodeAccess(e);
            // 返回老的值
            return oldValue;
        }
    }
    // <4>
    // 增加修改次数，如果该 Key 存在且允许修改，则不会增加 
    ++modCount;
    // 如果超过阀值，则进行扩容
    if (++size > threshold)
        resize();
    // 添加节点后的回调
    afterNodeInsertion(evict);
    // 返回 null
    return null;
}

// HashMap 允许树化最小 table 的长度
static final int MIN_TREEIFY_CAPACITY = 64;

final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // <1.1> 如果 table 长度小于 MIN_TREEIFY_CAPACITY(64) ，则选择先扩容不会树化
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    // <1.2> 将 hash 对应位置进行树化
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        // 顺序遍历链表，逐个转换成 TreeNode 节点
        TreeNode<K,V> hd = null, tl = null;
        do {
            // return new TreeNode<>(p.hash, p.key, p.value, next);
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                // 一样会用 next 连接该槽的所有节点
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        // 树化
        if ((tab[index] = hd) != null)
            hd.treeify(tab); // 树化部分介绍
    }
}
```

红黑树部分：TreeNoede#putTreeval

```java
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                       int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
    TreeNode<K,V> root = (parent != null) ? root() : this;
    
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        if ((ph = p.hash) > h)
            dir = -1;
        else if (ph < h)
            dir = 1;
        // hash 值相等 且 key 相等 ，则返回该节点
        else if ((pk = p.key) == k || (k != null && k.equals(pk))) 
            return p;
        /**
         * hash 值相等 但是 key 不相等，即 hash 碰撞了
         */
        // k 是否实现了Compareable
        		 // 没实现，进入 if，让 tieBreakOrder 决定方向dir
        else if ((kc == null && (kc = comparableClassFor(k)) == null) || 
                 // 实现了，如果 compare 后不相等则不进入 if，其返回的正负就是下一步遍历的方向，
                 // 如果 compare 相等，则在该节点的子树查找是否已经存在 k
                 (dir = compareComparables(kc, k, pk)) == 0) {	
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                if (((ch = p.left) != null &&
                     (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                     (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            
            // 在 p 的子树没有找到，决定下一步要遍历的方向
            dir = tieBreakOrder(k, pk);
        }

        TreeNode<K,V> xp = p;
        // 根据 dir 获取到下一次遍历的节点
        // 如果获取后等于 null，那就说明找到了插入的地方，则进行插入
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            Node<K,V> xpn = xp.next;
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
            xp.next = x;
            x.parent = x.prev = xp;
            if (xpn != null)
                ((TreeNode<K,V>)xpn).prev = x;
            // balanceInsertion(root, x) 插入后按照红黑树规则调整树
            // 调整 tab 里该 hash 对应槽位的第一个元素位红黑树调整后的 root
            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}

// 如果对象x的类是C，如果C实现了Comparable<C>接口，那么返回C，否则返回null
static Class<?> comparableClassFor(Object x) {
    if (x instanceof Comparable) {
        Class<?> c; Type[] ts, as; Type t; ParameterizedType p;
        if ((c = x.getClass()) == String.class) // 如果x是个字符串对象
            return c; // 返回String.class，因为String  实现了 Comparable 接口
 
        // 如果 c 不是字符串类，获取c直接实现的接口（如果是泛型接口则附带泛型信息）    
        if ((ts = c.getGenericInterfaces()) != null) {
            for (int i = 0; i < ts.length; ++i) { // 遍历接口数组
                // 如果当前接口t是个泛型接口 
                // 如果该泛型接口t的原始类型p 是 Comparable 接口
                // 如果该Comparable接口p只定义了一个泛型参数
                // 如果这一个泛型参数的类型就是c，那么返回c
                if (((t = ts[i]) instanceof ParameterizedType) &&
                    ((p = (ParameterizedType)t).getRawType() ==
                        Comparable.class) &&
                    (as = p.getActualTypeArguments()) != null &&
                    as.length == 1 && as[0] == c) // type arg is c
                    return c;
            }
            // 上面for循环的目的就是为了看看x的class是否 implements  Comparable<x的class>
        }
    }
    return null; // 如果c并没有实现 Comparable<c> 那么返回空
}

// k 既然实现了 Comparable 接口，该方法中进行比较
@SuppressWarnings({"rawtypes","unchecked"}) // for cast to Comparable
static int compareComparables(Class<?> kc, Object k, Object x) {
           // 如果x为空，或者其所属的类不是kc，返回0，在tieBreakOrder 中决定方向 dir
    return (x == null || x.getClass() != kc ? 0 : 
           // 如果x所属的类是kc，返回k.compareTo(x)的比较结果，如果非零就直接决定 dir 且不进入if
            ((Comparable)k).compareTo(x)); 
}


// 用这个方法来比较两个对象，返回值要么大于0，要么小于0，不会为0，不然就无法继续满足二叉树结构了
static int tieBreakOrder(Object a, Object b) {
    int d;
    if (a == null || b == null ||
        // 比较两个对象的类名，类名是字符串对象，就按字符串的比较规则
        (d = a.getClass().getName().compareTo(b.getClass().getName())) == 0)
        // 如果两个对象是同一个类型，调用本地方法为两个对象生成hashCode值，再进行比较（相等返回-1）
        d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                -1 : 1);
    return d;
}

// 如果 root 不是 tab 对应 hash 槽位的第一个，则把 root 放到 hash 的第一个位置，把原先的第一个连到 root 后面
static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
    int n;
    if (root != null && tab != null && (n = tab.length) > 0) {
        int index = (n - 1) & root.hash;
        TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
        // root 不是第一个
        if (root != first) {
            Node<K,V> rn;
            // 把当前 root 设为第一个
            tab[index] = root;
            TreeNode<K,V> rp = root.prev;
            if ((rn = root.next) != null)
                ((TreeNode<K,V>)rn).prev = rp;
            if (rp != null)
                rp.next = rn;
            if (first != null)
                // 把 first 连到 root 后面
                first.prev = root;
            root.next = first;
            root.prev = null;
        }
        assert checkInvariants(root);
    }
}
```



- 添加多个元素

```java
public void putAll(Map<? extends K, ? extends V> m) {
    putMapEntries(m, true);
}
```



## 扩容

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // <1> 开始：
    // <1.1> oldCap 大于 0 ，说明 table 非空
    if (oldCap > 0) {
        // <1.1.1> 超过最大容量，则直接设置 threshold 阀值为 Integer.MAX_VALUE ，不再允许扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        
        // <1.1.2> newCap = oldCap << 1 ，目的是两倍扩容
        // 如果 oldCap >= DEFAULT_INITIAL_CAPACITY ，说明当前容量大于默认值（16），则 2 倍阀值。
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // <1.2.1>【非默认构造方法】oldThr 大于 0 ，则使用 oldThr（即threshold） 作为新的容量
    else if (oldThr > 0)
        newCap = oldThr;
    // <1.2.2>【默认构造方法】oldThr 等于 0 ，则使用 DEFAULT_INITIAL_CAPACITY 作为新的容量，
    // 使用 DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY 作为新的容量
    else {               
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    // 1.3 如果上述的逻辑，未计算新的阀值，则使用 newCap * loadFactor 作为新的阀值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    
	//----------到此 已经计算出 newCap、newThr，下面会对 table 扩容和重分配-------------
    
    // <2> 开始：
    // 将 newThr 赋值给 threshold 属性
    threshold = newThr;
    // 创建新的 Node 数组，赋值给 table 属性
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    
    // 如果老的 table 数组非空，则需要进行一波搬运
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            // 获得老的 table 数组第 j 位置的 Node 节点 e
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                // 置空老的 table 数组第 j 位置
                oldTab[j] = null;
                // <2.1> 如果 e 节点只有一个元素，直接赋值给新的 table 即可
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // <2.2> 如果 e 节点是红黑树节点，则通过红黑树分裂处理
                else if (e instanceof TreeNode)
                    // 如下代码截图
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // <2.3> 如果 e 节点是链表
                else { // preserve order
                    // HashMap 是成倍扩容，原来位置的链表的节点会被分散到新的 table 的两个位置中去
                    // 通过 e.hash & oldCap 计算，根据结果分到高位、和低位的位置中。  
                    Node<K,V> loHead = null, loTail = null; // 低位的 head 节点
                    Node<K,V> hiHead = null, hiTail = null; // 高位的 head 节点
                    Node<K,V> next;
                    // 这里 do while 的原因是，e 已经非空，所以减少一次判断。细节~
                    do {
                        // next 指向下一个节点
                        next = e.next;
                        /**
                         * 实际在 oldcap << 1 之后的变化就是(n - 1) 最高位又多了个 1 ，那么只要让
                         * 最高位这个 1 失效就可以保证 hash在对扩容后的 (n - 1) 按位与运算后值不变
                         *
                         * 例如 oldcap = 4 , 有 hash 为 11 和 15 的两个元素
                         *    11 & (4 - 1) = {1011 & 011} = {11} = 3 
                         *    15 & (4 - 1) = {1111 & 011} = {11}= 3 
                         *    相当于 11 的 1[0]11 的 [0] 决定该位置始终为 0 
                         *  扩容之后
                         *    11 & (7 - 1) = {1011 & 0111} = {11} = 3
                         *    15 & (8 - 1) = {1111 & 0111} = {111}= 7 
                         *    相当于 11 的 1[0]11 的 [0] 决定该位置始终为 0 ，让 0 被保留了。
                         *
                         * 那么怎么判断 hash 值对应扩容后 (n - 1) 的二进制最高位是不是 [0] 
                         * 用 hash & oldCap 是否等于 0 即可得到。
                         * 如果该位置是 [1] 的话，那么其位置就会 j + oldCap ，因为 [1] 的价值
                         * 就是 + oldCap，因为要再加（ 2 的 n-1 次方）即 oldCap 的值 
                         */
                        if ((e.hash & oldCap) == 0) { // 如果结果为 0 时，则放置到低位
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        } else { // 如果结果非 1 时，则放置到高位
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    
                    // 设置低位到新的 newTab 的 j 位置上
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 设置高位到新的 newTab 的 j + oldCap 位置上
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}

```

红黑树部分：TreeNode#spilt()
<img src = 'https://img-blog.csdnimg.cn/20200929164455734.png' />
<img src = 'https://img-blog.csdnimg.cn/20200929164513201.png' />




## 移除元素

- 移除单个元素

```java
public V remove(Object key) {
    Node<K,V> e;
    // hash(key) 求哈希值
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; // table 数组
    Node<K,V> p; // hash 对应 table 位置的 p 节点
    int n, index;
    // <1> 查找 hash 对应 table 位置的 p 节点
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, // 如果找到 key 对应的节点，则赋值给 node
                e;
        K k; V v;
        // <1.1> 如果找到的 p 节点，就是要找的，则则直接使用即可
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            // <1.2> 如果找到的 p 节点，是红黑树 Node 节点，则直接在红黑树中查找
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            // <1.3> 如果找到的 p 是 Node 节点，则说明是链表，需要遍历查找
            else {
                do {
                    // 如果遍历的 e 节点，就是要找的，则则直接使用即可
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break; // 结束
                    }
                    p = e; // 注意，这里 p 会保存找到节点的前一个节点
                } while ((e = e.next) != null);
            }
        }
        
   		//----------------找到要删除的 node 节点 ------------------------
        
        // <2> 如果找到 node 节点，则进行移除
        // 如果有要求匹配 value 的条件，这里会进行一次判断先移除
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            // <2.1> 如果找到的 node 节点，是红黑树 Node 节点，则直接在红黑树中删除
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // <2.2.1> 如果查找到的是链表的头节点，则直接将 table 对应位置指向 node 的下一个节点
            else if (node == p)
                tab[index] = node.next;
            // <2.2.2> 如果查找到的是链表的中间节点，则将 p 指向 node 的下一个节点，实现删除
            else
                p.next = node.next;
            
            // 增加修改次数
            ++modCount;
            // 减少 HashMap 数量
            --size;
            // 移除 Node 后的回调
            afterNodeRemoval(node);
            // 返回 node
            return node;
        }
    }
    // 查找不到，则返回 null
    return null;
}
```

红黑树部分：TreeNode#removeTreeNode

```java
final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                                  boolean movable) {
    int n;
    if (tab == null || (n = tab.length) == 0)
        return;
    
    int index = (n - 1) & hash;
    // 获取到当前 hash 槽的第一个元素 first 作为临时的 root
    TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
    // 获取当前节点的 next 和 prev
    TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
    
    // <1> 先从该 hash 槽的链表中删除
    // <1.1> 先改变 prev 的 next 指针
    // 如果 pred 为空说明该节点是 root，所以只要让该节点的 next 作为 root 并设置给 tab 即可
    if (pred == null)
        tab[index] = first = succ; // 给 hash槽 first 赋值为 next
    // 否则直接让 prev 跳过当前节点直接指向 next
    else
        pred.next = succ;
    		
    // <1.2> 改变 next 的 prev 指针
    if (succ != null)
        succ.prev = pred;
    
    // 在把 succ 赋值给 first 后，first 为空就说明当前 hash 值的链表只有其一个节点
    if (first == null)
        return;
    
    // 如果 root 在红黑树中可能还有父节点，所以让其父节点作为root
    if (root.parent != null)
       root = root.root();
    
    if (root == null
        || (movable
            && (root.right == null
                || (rl = root.left) == null
                || rl.left == null))) {
        tab[index] = first.untreeify(map);  // too small
        return;
    }
    		
    // <2> 从红黑树中删除删除当前节点
    TreeNode<K,V> p = this, pl = left, pr = right, replacement;
    /**
     * 如果左右子节均不为空，不可以使用数经典的删除方法，即找到右子树的最小值，然后进行值交换，
     * 因为前面已经在 hash 值链表里把这个节点删了，所以必须要删掉当前节点，因此只有改变树的连接
     */
    if (pl != null && pr != null) {
        TreeNode<K,V> s = pr, sl;
       // <2.1>在右子树找到最左边的节点（大于当前节点值的最小值），s 会和 p 交换位置
        while ((sl = s.left) != null) // find successor
            s = sl;
        // 进行颜色交换，值不用换，因为会替换 p 节点
        boolean c = s.red; s.red = p.red; p.red = c; // swap colors
                
        // 记录
        TreeNode<K,V> sr = s.right;
        TreeNode<K,V> pp = p.parent;
                
        // <2.2> 把 p 放到 s 的位置（（即交换连接，连接 s 的 parent、child）
        // 如果 p 只有一个右子节点（即 s == p.right)，让 p 作为 s 的右子节点
        if (s == pr) { // p was s's direct parent
           p.parent = s;
            s.right = p;
        }
        else { 
            // <2.2.1> p 的 parent 变为 s 的 parent，s 的 parent 的子节点变为 p
            TreeNode<K,V> sp = s.parent;
            // 让 p 的 parent 指向 s 的 parent
            if ((p.parent = sp) != null) {
                // 让 s 的 parent 的子节点也指向 p
                if (s == sp.left) 
                    sp.left = p;
                else
                    sp.right = p;
            }
            // <2.3> 让 s 取代 p（即交换连接，连接 p 的 parent、child）
            // <2.3.1> s 连接 p 的右子节点
            if ((s.right = pr) != null)
                pr.parent = s;
        }
                
        // <2.2.2> p 连接 s 的左子节点（即 null）
        // 因为 p 已经完全放到了 s 的位置，而 s 是最左边，所以 p 的左边也应该为空
        p.left = null;
                
        // <2.2.3> p 连接 s 的右子节点
        if ((p.right = sr) != null)  
            sr.parent = p; // 对应改变父节点
                
        // <2.3.2> s 连接 p 的左子节点
        if ((s.left = pl) != null)
            pl.parent = s; // 对应改变父节点
                
        // <2.3.3> s 的 parent 变为 p 的 parent，p 的 parent 的子节点变为 s
        if ((s.parent = pp) == null)
            root = s;
        else if (p == pp.left)
            pp.left = s;
        else
            pp.right = s;
          
        // <2.4> 选一个节点在下面进行连接或者丢弃
        if (sr != null) // p 已经完全完成了和 s 交换连接，已经在他原先右子树的最左边了， 所以肯定没有左子节点，但可能有右子节点，该种情况同下面的 else if（pr != null）
            replacement = sr;
        else // p 没有子节点，该种情况同下面的 else，会删除 p
            replacement = p; /
    }
  	// 只有左子节点
    else if (pl != null)
        replacement = pl;
    // 只有右子节点
    else if (pr != null)
        replacement = pr;
    // 无子节点
    else 
        replacement = p;
    
    // 如果当前节点有子节点，或者 s 和 p 交换连接后 p 有右子节点，让 p 的新父节点指向 replacement
    if (replacement != p) {
        TreeNode<K,V> pp = replacement.parent = p.parent;
        if (pp == null)
             (root = replacement).red = false;
        else if (p == pp.left)
            pp.left = replacement;
        else
            pp.right = replacement;
        p.left = p.right = p.parent = null;
    }
			
    // 如果当前节点是红色，那么在删除后不会有影响，如果是黑色可能要调整
    TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);

    // 当前节点无子节点，或者 s 和 p 交换连接后没子节点，直接让 p 的新父节点指向该节点的指针变为 null
    if (replacement == p) {  // detach
        TreeNode<K,V> pp = p.parent;
        p.parent = null;
        if (pp != null) {
            if (p == pp.left)
                pp.left = null;
             else if (p == pp.right)
                pp.right = null;
         }
    }
    
    // 入参为 true
    if (movable)
        // 改变该槽位的 first 为 当前 root
        moveRootToFront(tab, r);
}
```






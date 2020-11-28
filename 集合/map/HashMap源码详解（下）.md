## 树化

TreeNode#treeify （在 HashMap 和 TreeNode 中均有调用，无返回值）

```java
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    // 从当前 hash 值相等的链表第一个节点开始建树
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        // 就让链表第一个元素成为 root
        if (root == null) {
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            // 逻辑同 putTreeVal 
            for (TreeNode<K,V> p = root;;) {
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

                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    // 插入后按照红黑树规则调整树
                    root = balanceInsertion(root, x);
                    break;
                }
           }
        }
    }
    // 调整 tab 当前hash 槽第一个元素
    moveRootToFront(tab, root);
}
```

- HashMap 在 put 元素时若同时满足下面两个条件时触发

  ```java
  // 每个位置链表树化成红黑树，需要的链表最小长度
  static final int TREEIFY_THRESHOLD = 8;
  
  // HashMap 允许树化最小 table 的长度
  static final int MIN_TREEIFY_CAPACITY = 64;
  ```

  

- 为什么是 8

  - 首先，参考 [泊松概率函数(Poisson distribution)](http://en.wikipedia.org/wiki/Poisson_distribution) ，当链表长度到达 8 的概率是 0.00000006 ，不到千万分之一。所以绝大多数情况下，在 hash 算法正常的时，不太会出现链表转红黑树的情况。
  - 其次，TreeNode 相比普通的 Node 来说，会有**两倍**的空间占用。并且在长度比较小的情况下，红黑树的查找性能和链表是差别不大的。例如说，红黑树的 `O(logN) = log8 = 3` 和链表的 `O(N) = 8` 只相差 5 
  - 毕竟 HashMap 是 JDK 提供的基础数据结构，必须在空间和时间做抉择。所以，选择链表是空间复杂度优先，选择红黑树是时间复杂度优化。在绝大多数情况下，不会出现需要红黑树的情况。



TreeNode#untreeify （该方法只在 TreeNode 内部调用 ，比如 spilt、removeTreeNode，有返回值）

```java
final Node<K,V> untreeify(HashMap<K,V> map) {
    Node<K,V> hd = null, tl = null;
    for (Node<K,V> q = this; q != null; q = q.next) {
        // return new Node<>(p.hash, p.key, p.value, next);
        Node<K,V> p = map.replacementNode(q, null);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    // 直接替换成 Node 后返回链表即可
    return hd
}
```

- 当 HashMap 因为移除 key 时，导致对应 `table` 位置的红黑树的内部节点数小于等于 `UNTREEIFY_THRESHOLD = 6` 时，则将红黑树退化成链表。

  ```java
  /**
   * The bin count threshold for untreeifying a (split) bin during a
   * resize operation. Should be less than TREEIFY_THRESHOLD, and at
   * most 6 to mesh with shrinkage detection under removal.
   */
  static final int UNTREEIFY_THRESHOLD = 6;
  ```

- 使用 6 作为取消树化的阀值，可能是为了避免后续移除 key 时，红黑树如果内部节点数小于 7 就退化成链表，这样可能导致过于频繁的树化和取消树化。



## 红黑树调整
（不做讲解）
- 共用方法 TreeNode#rotateLeft、TreeNode#rotateRight

	```JAVA
	// TODO
	/* ------------------------------------------------------------ */
        // Red-black tree methods, all adapted from CLR

        static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                              TreeNode<K,V> p) {
            TreeNode<K,V> r, pp, rl;
            if (p != null && (r = p.right) != null) {
                if ((rl = p.right = r.left) != null)
                    rl.parent = p;
                if ((pp = r.parent = p.parent) == null)
                    (root = r).red = false;
                else if (pp.left == p)
                    pp.left = r;
                else
                    pp.right = r;
                r.left = p;
                p.parent = r;
            }
            return root;
        }

        static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                               TreeNode<K,V> p) {
            TreeNode<K,V> l, pp, lr;
            if (p != null && (l = p.left) != null) {
                if ((lr = p.left = l.right) != null)
                    lr.parent = p;
                if ((pp = l.parent = p.parent) == null)
                    (root = l).red = false;
                else if (pp.right == p)
                    pp.right = l;
                else
                    pp.left = l;
                l.right = p;
                p.parent = l;
            }
            return root;
        }
	```



- 添加后调整 TreeNode#balanceInsertion 

	```java
		// TODO
		static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                    TreeNode<K,V> x) {
            x.red = true;
            for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
                if ((xp = x.parent) == null) {
                    x.red = false;
                    return x;
                }
                else if (!xp.red || (xpp = xp.parent) == null)
                    return root;
                if (xp == (xppl = xpp.left)) {
                    if ((xppr = xpp.right) != null && xppr.red) {
                        xppr.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.right) {
                            root = rotateLeft(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateRight(root, xpp);
                            }
                        }
                    }
                }
                else {
                    if (xppl != null && xppl.red) {
                        xppl.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.left) {
                            root = rotateRight(root, x = xp);
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateLeft(root, xpp);
                            }
                        }
                    }
                }
            }
		}
	```



- 删除后调整 TreeNode#balanceDeletion 

	```java
		// TODO
		static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root,
                                                   TreeNode<K,V> x) {
            for (TreeNode<K,V> xp, xpl, xpr;;) {
                if (x == null || x == root)
                    return root;
                else if ((xp = x.parent) == null) {
                    x.red = false;
                    return x;
                }
                else if (x.red) {
                    x.red = false;
                    return root;
                }
                else if ((xpl = xp.left) == x) {
                    if ((xpr = xp.right) != null && xpr.red) {
                        xpr.red = false;
                        xp.red = true;
                        root = rotateLeft(root, xp);
                        xpr = (xp = x.parent) == null ? null : xp.right;
                    }
                    if (xpr == null)
                        x = xp;
                    else {
                        TreeNode<K,V> sl = xpr.left, sr = xpr.right;
                        if ((sr == null || !sr.red) &&
                            (sl == null || !sl.red)) {
                            xpr.red = true;
                            x = xp;
                        }
                        else {
                            if (sr == null || !sr.red) {
                                if (sl != null)
                                    sl.red = false;
                                xpr.red = true;
                                root = rotateRight(root, xpr);
                                xpr = (xp = x.parent) == null ?
                                    null : xp.right;
                            }
                            if (xpr != null) {
                                xpr.red = (xp == null) ? false : xp.red;
                                if ((sr = xpr.right) != null)
                                    sr.red = false;
                            }
                            if (xp != null) {
                                xp.red = false;
                                root = rotateLeft(root, xp);
                            }
                            x = root;
                        }
                    }
                }
                else { // symmetric
                    if (xpl != null && xpl.red) {
                        xpl.red = false;
                        xp.red = true;
                        root = rotateRight(root, xp);
                        xpl = (xp = x.parent) == null ? null : xp.left;
                    }
                    if (xpl == null)
                        x = xp;
                    else {
                        TreeNode<K,V> sl = xpl.left, sr = xpl.right;
                        if ((sl == null || !sl.red) &&
                            (sr == null || !sr.red)) {
                            xpl.red = true;
                            x = xp;
                        }
                        else {
                            if (sl == null || !sl.red) {
                                if (sr != null)
                                    sr.red = false;
                                xpl.red = true;
                                root = rotateLeft(root, xpl);
                                xpl = (xp = x.parent) == null ?
                                    null : xp.left;
                            }
                            if (xpl != null) {
                                xpl.red = (xp == null) ? false : xp.red;
                                if ((sl = xpl.left) != null)
                                    sl.red = false;
                            }
                            if (xp != null) {
                                xp.red = false;
                                root = rotateRight(root, xp);
                            }
                            x = root;
                        }
                    }
                }
            }
		}
	```

## 查找元素

- 查找单个元素

```java
public V get(Object key) {
    Node<K,V> e;
    // hash(key) 哈希值
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 查找 hash 对应 table 位置的 p 节点
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        
        // 如果找到的 first 节点，就是要找的，则则直接使用即可
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        
        if ((e = first.next) != null) {
            // 如果找到的 first 节点，是红黑树 Node 节点，则直接在红黑树中查找
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
        
            // 如果找到的 e 是 Node 节点，则说明是链表，需要遍历查找
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}

```

红黑树部分：

```java
final TreeNode<K,V> getTreeNode(int h, Object k) {
    return ((parent != null) ? root() : this).find(h, k, null);
}


final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
    TreeNode<K,V> p = this;
    do {
        int ph, dir; K pk;
        TreeNode<K,V> pl = p.left, pr = p.right, q;
        if ((ph = p.hash) > h)
            p = pl;
        else if (ph < h)
            p = pr;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        /**
         * 从下面开始是 hash 冲突的情况，要解决的是hash值相同的时候查找左子树还是右子树
         */
        else if (pl == null) // hash 冲突，但左子树为空
            p = pr;
        else if (pr == null) // hash 冲突，但右子树为空
            p = pl;
        else if ((kc != null ||
                  (kc = comparableClassFor(k)) != null) &&
                 (dir = compareComparables(kc, k, pk)) != 0)
            p = (dir < 0) ? pl : pr; // hash 冲突，且左右子树均不为空，但实现了 Compareble 
        else if ((q = pr.find(h, k, kc)) != null)
            return q; // 上面都不满足，那只有先把右子树找一遍，如果找到就返回
        else
            p = pl; // 把右子树遍历完没找到，所以下一次遍历左子树
    } while (p != null);
    return null;
}
```



## 清空

```java
public void clear() {
    Node<K,V>[] tab;
    // 增加修改次数
    modCount++;
    if ((tab = table) != null && size > 0) {
        // 设置大小为 0
        size = 0;
        // 设置每个位置为 null
        for (int i = 0; i < tab.length; ++i)
            tab[i] = null;
    }
}
```

## 迭代器

1、Map 未实现迭代器接口，需要转换成集合

* keySet
```java
transient Set<K> keySet;

public Set<K> keySet() {
    // 获得 keySet 缓存
    Set<K> ks = keySet;
    // 如果不存在，则进行创建
    if (ks == null) {
        ks = new KeySet();
        keySet = ks;
    }
    return ks;
}


final class KeySet extends AbstractSet<K> { 
         
    	public final Iterator<K> iterator()     { return new KeyIterator(); }
    	
    
    	// 其余方法略，都是直接调用 HashMap 的方法。
}

final class KeySet extends AbstractSet<K> {
        // 迭代器返回 KeyIterator
    	public final Iterator<K> iterator()     { return new KeyIterator(); }
       
        // 遍历是通过遍历 table 数组 
        public final void forEach(Consumer<? super K> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                // 嵌套循环
                for (Node<K,V> e : tab) {
                    for (; e != null; e = e.next)
                        action.accept(e.key);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
    
    	//-----------下面都是直接调用 HashMap 的方法---------------------
    	public final int size()                 { return size; }
        public final void clear()               { HashMap.this.clear(); }
    	public final boolean contains(Object o) { return containsKey(o); }
        public final boolean remove(Object key) {
            return removeNode(hash(key), key, null, false, true) != null;
        }
        public final Spliterator<K> spliterator() {
            return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
        }
    }

// 迭代器最终实现是 HashIterator
final class KeyIterator extends HashIterator
        implements Iterator<K> {
        public final K next() { return nextNode().key; }
}

```


* values
```java
// AbstractMap.java
transient Collection<V> values

// HashMap.java
public Collection<V> values() {
    // 获得 vs 缓存
    Collection<V> vs = values;
    // 如果不存在，则进行创建
    if (vs == null) {
        vs = new Values();
        values = vs;
    }
    return vs;
}


final class Values extends AbstractCollection<V> {
        // 迭代器返回 ValueIterator
        public final Iterator<V> iterator()     { return new ValueIterator(); }
        
        // 同 KeySet，其余方法略...
}

// 迭代器最终实现是 HashIterator
final class ValueIterator extends HashIterator
        implements Iterator<V> {
        public final V next() { return nextNode().value; }
}
```


* entrySet
```java
// AbstractMap.java
transient Set<Map.Entry<K,V>> entrySet;

// HashMap.java
public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    // 获得 entrySet 缓存
    // 如果不存在，则进行创建
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}

final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    	// 迭代器返回的 EntryIterator
        public final Iterator<Map.Entry<K,V>> iterator() {
            return new EntryIterator();
        }
        
        // 同 KeySet，其余方法略...
}

// 迭代器最终实现是 HashIterator
final class EntryIterator extends HashIterator
        implements Iterator<Map.Entry<K,V>> {
        public final Map.Entry<K,V> next() { return nextNode(); }
}
```



2、HashIterator（next() 方法未实现）

```java
abstract class HashIterator {
        Node<K,V> next;        // next entry to return
        Node<K,V> current;     // current entry
        int expectedModCount;  // for fast-fail
        int index;             // current slot

        HashIterator() {
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
            if (t != null && size > 0) { 
                // 找到 table 第一个不为空的位置，即也是第一个 entry
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            // 如果 next 为空，且 table 不为空则继续遍历 table
            if ((next = (current = e).next) == null && (t = table) != null) {
                do {} while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }

        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            // 直接调用 removeNode()
            removeNode(p.hash, p.key, null, false, false);
            // 修改 expectedModCount
            expectedModCount = modCount;
        }
}
```



## 序列化

- 序列化

```java
@java.io.Serial
private void writeObject(java.io.ObjectOutputStream s)
    throws IOException {
    // 获得 HashMap table 数组大小
    int buckets = capacity();
    // Write out the threshold, loadfactor, and any hidden stuff
    // 写入非静态属性、非 transient 属性
    s.defaultWriteObject();
    // 写入 table 数组大小
    s.writeInt(buckets);
    // 写入 key-value 键值对数量
    s.writeInt(size);
    // 写入具体的 key-value 键值对
    internalWriteEntries(s);
}

// table 数组大小。封装方法的原因，需要考虑 table 未初始化的情况。
final int capacity() { 
    return (table != null) ? table.length :
        (threshold > 0) ? threshold :
        DEFAULT_INITIAL_CAPACITY;
}

// Called only from writeObject, to ensure compatible ordering.
void internalWriteEntries(java.io.ObjectOutputStream s) throws IOException {
    Node<K,V>[] tab;
    if (size > 0 && (tab = table) != null) {
        // 遍历 table 数组
        for (Node<K,V> e : tab) {
            // 遍历链表或红黑树
            for (; e != null; e = e.next) {
                // 写入 key
                s.writeObject(e.key);
                // 写入 value
                s.writeObject(e.value);
            }
        }
    }
}
```



- 反序列化

```java
@java.io.Serial
private void readObject(java.io.ObjectInputStream s)
    throws IOException, ClassNotFoundException {
    // 读取非静态属性、非 transient 属性
    s.defaultReadObject();
    // 重新初始化
    reinitialize();
    
    // 校验 loadFactor 参数
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new InvalidObjectException("Illegal load factor: " +
                                         loadFactor);
    // 读取 HashMap table 数组大小
    s.readInt();                // Read and ignore number of buckets
    // 读取 key-value 键值对数量 size
    int mappings = s.readInt(); // Read number of mappings (size)
    // 校验 size 参数
    if (mappings < 0)
        throw new InvalidObjectException("Illegal mappings count: " +
                                         mappings);
    else if (mappings > 0) { // (if zero, use defaults)
        // Size the table using given load factor only if within
        // range of 0.25...4.0
        float lf = Math.min(Math.max(0.25f, loadFactor), 4.0f);
        float fc = (float)mappings / lf + 1.0f;
        // 计算容量
        int cap = ((fc < DEFAULT_INITIAL_CAPACITY) ?
                   DEFAULT_INITIAL_CAPACITY :
                   (fc >= MAXIMUM_CAPACITY) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor((int)fc));
        // 计算 threshold 阀值
        float ft = (float)cap * lf;
        threshold = ((cap < MAXIMUM_CAPACITY && ft < MAXIMUM_CAPACITY) ?
                     (int)ft : Integer.MAX_VALUE);

        // Check Map.Entry[].class since it's the nearest public type to
        // what we're actually creating.
        SharedSecrets.getJavaObjectInputStreamAccess().checkArray(s, Map.Entry[].class, cap); 
        // 创建 table 数组
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] tab = (Node<K,V>[])new Node[cap];
        table = tab;

      
        // 遍历读取 key-value 键值对
        for (int i = 0; i < mappings; i++) {
            // 读取 key
            @SuppressWarnings("unchecked")
            K key = (K) s.readObject();
            // 读取 value
            @SuppressWarnings("unchecked")
            V value = (V) s.readObject();
            // 添加 key-value 键值对
            putVal(hash(key), key, value, false, false);
        }
    }
}

/**
 * Reset to initial default state.  Called by clone and readObject.
 */
void reinitialize() {
    table = null;
    entrySet = null;
    keySet = null;
    values = null;
    modCount = 0;
    threshold = 0;
    size = 0;
}
```



## 小结

- HashMap 是一种散列表的数据结构，底层采用数组 + 链表 + 红黑树来实现存储。

  > Redis Hash 数据结构，采用数组 + 链表实现。
  >
  > Redis Zset 数据结构，采用跳表实现。
  >
  > 因为红黑树实现起来相对复杂，我们自己在实现 HashMap 可以考虑采用数组 + 链表 + 跳表来实现存储。

- HashMap 默认容量为 16(`1 << 4`)，每次超过阀值时，按照两倍大小进行自动扩容，所以容量总是 2^N 次方。并且，底层的 `table` 数组是延迟初始化，在首次添加 key-value 键值对才进行初始化。

- HashMap 默认加载因子是 0.75 ，如果我们已知 HashMap 的大小，需要正确设置容量和加载因子。

- HashMap 每个槽位在满足如下两个条件时，可以进行树化成红黑树，避免槽位是链表数据结构时，链表过长，导致查找性能过慢。

  - 树化：
    - 条件一，HashMap 的 `table` 数组大于等于 64 。
    - 条件二，槽位链表长度大于等于 8 时。选择 8 作为阀值的原因是，参考 [泊松概率函数(Poisson distribution)](http://en.wikipedia.org/wiki/Poisson_distribution) ，概率不足千万分之一。
  - 退为非树化：
    - 在槽位的红黑树的节点数量小于等于 6 时，会退化回链表。

- HashMap 的查找和添加 key-value 键值对的平均时间复杂度为 O(1) 。

  - 对于槽位是链表的节点，**平均**时间复杂度为 O(k) 。其中 k 为链表长度。
  - 对于槽位是红黑树的节点，**平均**时间复杂度为 O(logk) 。其中 k 为红黑树节点数量。
  

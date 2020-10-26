#### TreeMap

![](https://img-blog.csdnimg.cn/20200929170440927.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


- NaviableMap 定义了一些有方向操作

	![在这里插入图片描述](https://img-blog.csdnimg.cn/20200929170509671.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



红黑树

1. 每个节点要么是红色，要么是黑色。
2. 根节点必须是黑色
3. 红色节点不能连续（也即是，红色节点的孩子和父亲都不能是红色）。
4. 对于每个节点，从该点至`null`（树尾端）的任何路径，都含有相同个数的黑色节点。

##### 数据结构

```java
// key 排序器
private final Comparator<? super K> comparator;

// 红黑树的根节点
private transient Entry<K,V> root;

// key-value 键值对数量
private transient int size = 0;

// 修改次数
private transient int modCount = 0;
```



- TreeMap#Entry

  ```java
  // 颜色 - 红色
  private static final boolean RED   = false;
  // 颜色 - 黑色
  private static final boolean BLACK = true;
  
  static final class Entry<K,V> implements Map.Entry<K,V> {
  
      K key;
      
      V value;
      
      Entry<K,V> left;
      
      Entry<K,V> right;
      
      Entry<K,V> parent;
      
      // 颜色
      boolean color = BLACK;
      
      // ... 省略一些
      
  }
  ```

  

##### 构造方法

- 默认构造方法

```java
public TreeMap() {
    comparator = null;
}
```



- 传入 `comparator` 参数

```java
public TreeMap(Comparator<? super K> comparator) {
    this.comparator = comparator;
}
```

- 传入 SortedMap

```java
public TreeMap(SortedMap<K, ? extends V> m) {
    // <1> 设置 comparator 属性
    comparator = m.comparator();
    try {
        // <2> 使用 m 构造红黑树
        buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
    } catch (java.io.IOException | ClassNotFoundException cannotHappen) {
    }
}

private void buildFromSorted(int size, Iterator<?> it,
                             java.io.ObjectInputStream str,
                             V defaultVal)
    throws  java.io.IOException, ClassNotFoundException {
    // <1> 设置 key-value 键值对的数量
    this.size = size;
    // <2> computeRedLevel(size) 方法，计算红黑树的高度
    // <3> 使用 m 构造红黑树，返回根节点
    root = buildFromSorted(0, 0, size - 1, computeRedLevel(size),
                           it, str, defaultVal);
}

private static int computeRedLevel(int size) {
    // 因为构建的平衡二叉树从第零0层开始一次有 1 2 4 8 16 .... 个元素，所以高度就是二进制位的个数
    return 31 - Integer.numberOfLeadingZeros(size + 1);
}


/**
 * 使用 m 构造红黑树。因为 m 是 SortedMap 类型，所以天然有序，所以可以基于 m 的中间为红黑树的根节点，
 * m 的左边为左子树，m 的右边为右子树。
 * 因为元素有序，通过递归构建一颗完全平衡的二叉树，并且除了最高层的叶子节点为红色，其余节点均为黑色。
 *
 * level:从哪一层开始
 * lo：有序序列的低位索引（并无实际作用，只是为了和 hi 比较是否递归结束）
 * hi：有序序列的高位索引（并无实际作用，只是为了和 hi 比较是否递归结束）
 * redLevel：要将所有最高层的节点都设置成 red
 * it：通过迭代器获取元素（和 str 二选一）
 * str：通过流获取元素（和 it 二选一）
 * defaultVal：获取完元素的键 在获取值时 如果 defaultVal 不为空则直接使用defaultVal 
 * 			   否则接着从迭代器或者流中获取
 */
private final Entry<K,V> buildFromSorted(int level, int lo, int hi,
                                         int redLevel,
                                         Iterator<?> it,
                                         java.io.ObjectInputStream str,
                                         V defaultVal)
    throws  java.io.IOException, ClassNotFoundException {
    
    // <1.1> 递归结束
    if (hi < lo) return null;

    // <1.2> 计算中间值
    int mid = (lo + hi) >>> 1;

    // <2.1> 创建左子树
    Entry<K,V> left  = null;
    if (lo < mid)
        // <2.2> 递归左子树
        left = buildFromSorted(level + 1, lo, mid - 1, redLevel,
                               it, str, defaultVal);
 
    // <3.1> 获得 key-value 键值对
    K key;
    V value;
    if (it != null) { // 使用 it 迭代器，获得下一个值
        if (defaultVal == null) {
            Map.Entry<?,?> entry = (Map.Entry<?,?>)it.next(); // 从 it 获得下一个 Entry 节点
            key = (K) entry.getKey(); // 读取 key
            value = (V) entry.getValue(); // 读取 value
        } else {
            key = (K)it.next();  // 读取 key
            value = defaultVal; // 设置 default 为 value
        }
    } else { // use stream 处理 str 流的情况
        key = (K) str.readObject(); //  从 str 读取 key 值
        value = (defaultVal != null ? defaultVal : (V) str.readObject()); // 从 str 读取 value 值
    }

    
    // <3.2> 创建中间节点
    Entry<K,V> middle =  new Entry<>(key, value, null);

    // color nodes in non-full bottommost level red
    // <3.3> 如果到树的最大高度，则设置为红节点
    if (level == redLevel)
        middle.color = RED;

    // <3.4> 如果左子树非空，则进行设置
    if (left != null) {
        middle.left = left; // 当前节点，设置左子树
        left.parent = middle; // 左子树，设置父节点为当前节点
    }

    // <4.1> 创建右子树
    if (mid < hi) {
        // <4.2> 递归右子树
        Entry<K,V> right = buildFromSorted(level + 1, mid + 1, hi, redLevel,
                                           it, str, defaultVal);
        // <4.3> 当前节点，设置右子树
        middle.right = right;
        // <4.3> 右子树，设置父节点为当前节点
        right.parent = middle;
    }

    // 返回当前节点
    return middle;
}

```



- 传入  Map 类型

```java
public TreeMap(Map<? extends K, ? extends V> m) {
    comparator = null;
    // 添加所有元素
    putAll(m);
}

public void putAll(Map<? extends K, ? extends V> map) {
    // <1> 路径一，满足如下条件，调用 buildFromSorted 方法来优化处理
    int mapSize = map.size();
    if (size == 0 // 如果 TreeMap 的大小为 0
            && mapSize != 0 // map 的大小非 0
            && map instanceof SortedMap) { // 如果是 map 是 SortedMap 类型
        if (Objects.equals(comparator, ((SortedMap<?,?>)map).comparator())) {//排序规则相同
            // 增加修改次数
            ++modCount;
            // 基于 SortedMap 顺序迭代插入即可
            try {
                buildFromSorted(mapSize, map.entrySet().iterator(),
                                null, null);
            } catch (java.io.IOException | ClassNotFoundException cannotHappen) {
            }
            return;
        }
    }
    // <2> 路径二，直接遍历 map 来添加
    super.putAll(map);
}
```





##### 添加元素

- 添加单个元素

```java
public V put(K key, V value) {
    // 记录当前根节点
    Entry<K,V> t = root;
    // <1> 如果无根节点，则直接使用 key-value 键值对，创建根节点
    if (t == null) {
        // <1.1> 校验 key 类型。
        compare(key, key); // type (and possibly null) check

        // <1.2> 创建 Entry 节点
        root = new Entry<>(key, value, null);
        // <1.3> 设置 key-value 键值对的数量
        size = 1;
        // <1.4> 增加修改次数
        modCount++;
        return null;
    }
    // <2> 遍历红黑树
    int cmp; // key 比父节点小还是大
    Entry<K,V> parent; // 父节点
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    if (cpr != null) { // 如果有自定义 comparator ，则使用它来比较
        do {
            // <2.1> 记录新的父节点
            parent = t;
            // <2.2> 比较 key
            cmp = cpr.compare(key, t.key);
            // <2.3> 比 key 小，说明要遍历左子树
            if (cmp < 0)
                t = t.left;
            // <2.4> 比 key 大，说明要遍历右子树
            else if (cmp > 0)
                t = t.right;
            // <2.5> 说明，相等，说明要找到的 t 就是 key 对应的节点，直接设置 value 即可。
            else
                return t.setValue(value);
        } while (t != null);  // <2.6>
    } else { // 如果没有自定义 comparator ，则使用 key 自身比较器来比较
        if (key == null) // 如果 key 为空，则抛出异常
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            // <2.1> 记录新的父节点
            parent = t;
            // <2.2> 比较 key
            cmp = k.compareTo(t.key);
            // <2.3> 比 key 小，说明要遍历左子树
            if (cmp < 0)
                t = t.left;
            // <2.4> 比 key 大，说明要遍历右子树
            else if (cmp > 0)
                t = t.right;
            // <2.5> 说明，相等，说明要找到的 t 就是 key 对应的节点，直接设置 value 即可。
            else
                return t.setValue(value);
        } while (t != null); // <2.6>
    }
    // <3> 创建 key-value 的 Entry 节点（默认为黑色）
    Entry<K,V> e = new Entry<>(key, value, parent);
    // 设置左右子树
    if (cmp < 0) // <3.1>
        parent.left = e;
    else // <3.2>
        parent.right = e;
    // <3.3> 插入后，进行自平衡
    fixAfterInsertion(e);
    // <3.4> 设置 key-value 键值对的数量
    size++;
    // <3.5> 增加修改次数
    modCount++;
    return null;
}

final int compare(Object k1, Object k2) {
    return comparator == null ?
        	// 如果没有自定义 comparator 比较器，则使用 key 自身比较
            ((Comparable<? super K>)k1).compareTo((K)k2) 
            : comparator.compare((K)k1, (K)k2); 
}
```



插入后调整以保持红黑树特性

```java
//TODO 红黑树调整函数（总共六种情况）
private void fixAfterInsertion(Entry<K,V> x) {
    x.color = RED;
    while (x != null && x != root && x.parent.color == RED) {
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {//如果y为null，则视为BLACK
                setColor(parentOf(x), BLACK);              // 情况1
                setColor(y, BLACK);                        // 情况1
                setColor(parentOf(parentOf(x)), RED);      // 情况1
                x = parentOf(parentOf(x));                 // 情况1
            } else {
                if (x == rightOf(parentOf(x))) {
                    x = parentOf(x);                       // 情况2
                    rotateLeft(x);                         // 情况2
                }
                setColor(parentOf(x), BLACK);              // 情况3
                setColor(parentOf(parentOf(x)), RED);      // 情况3
                rotateRight(parentOf(parentOf(x)));        // 情况3
            }
        } else {
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);              // 情况4
                setColor(y, BLACK);                        // 情况4
                setColor(parentOf(parentOf(x)), RED);      // 情况4
                x = parentOf(parentOf(x));                 // 情况4
            } else {
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);                       // 情况5
                    rotateRight(x);                        // 情况5
                }
                setColor(parentOf(x), BLACK);              // 情况6
                setColor(parentOf(parentOf(x)), RED);      // 情况6
                rotateLeft(parentOf(parentOf(x)));         // 情况6
            }
        }
    }
    root.color = BLACK;
}
```



##### 扩容

不需要扩容，直接增加节点即可，只有数组会涉及扩容。



##### 删除元素

- 二叉树如何删除元素
  - 情况一，无子节点。
    直接删除父节点对其的指向即可。
  - 情况二，**只**有左子节点。
    将删除节点的父节点，指向删除节点的左子节点。
  - 情况三，**只**有右子节点。
    和情况二的处理方式一致。将删除节点的父节点，指向删除节点的右子节点。
    - 情况四，有左子节点 + 右子节点。
      因为无法使用子节点替换掉删除的节点。所以先查找该节点右子树的**最小值**，然后将该节点的键值都进行替换，然后删除该节点右子树的**最小值**节点即可
- 删除单个元素

```java
public V remove(Object key) {
    // <1> 获得 key 对应的 Entry 节点
    Entry<K,V> p = getEntry(key);
    // <2> 如果不存在，则返回 null ，无需删除
    if (p == null)
        return null;

    V oldValue = p.value;
    // <3> 删除节点
    deleteEntry(p);
    return oldValue;
}

private void deleteEntry(Entry<K,V> p) {
    // 增加修改次数
    modCount++;
    // 减少 key-value 键值对数
    size--;

    // If strictly internal, copy successor's element to p and then make p
    // point to successor.
    // <1> 如果删除的节点 p 既有左子节点，又有右子节点，
    if (p.left != null && p.right != null) {
        // <1.1> 获得右子树的最小值
        Entry<K,V> s = successor(p);
        // <1.2> 修改 p 的 key-value 为 s 的 key-value 键值对
        p.key = s.key;
        p.value = s.value;
        // <1.3> 设置 p 指向 s 。此时，就变成删除 s 节点了。
        p = s;
    } // p has 2 children

    // Start fixup at replacement node, if it exists.
    // <2> 获得替换节点
    Entry<K,V> replacement = (p.left != null ? p.left : p.right);
    // <3> 有子节点的情况
    if (replacement != null) {
        // Link replacement to parent
        // <3.1> 替换节点的父节点，指向 p 的父节点
        replacement.parent = p.parent;
        // <3.2.1> 如果 p 的父节点为空，则说明 p 是根节点，直接 root 设置为替换节点
        if (p.parent == null)
            root = replacement;
        // <3.2.2> 如果 p 是父节点的左子节点，则 p 的父子节的左子节指向替换节点
        else if (p == p.parent.left)
            p.parent.left  = replacement;
        // <3.2.3> 如果 p 是父节点的右子节点，则 p 的父子节的右子节指向替换节点
        else
            p.parent.right = replacement;

        // Null out links so they are OK to use by fixAfterDeletion.
        // <3.3> 置空 p 的所有指向
        p.left = p.right = p.parent = null;

        // Fix replacement
        // <3.4> 如果 p 的颜色是黑色，则执行自平衡
        if (p.color == BLACK)
            fixAfterDeletion(replacement);
    // <4> 如果 p 没有父节点，说明删除的是根节点，直接置空 root 即可
    } else if (p.parent == null) { // return if we are the only node.
        root = null;
    // <5> 如果删除的没有左子树，又没有右子树
    } else { //  No children. Use self as phantom replacement and unlink.
        // <5.1> 如果 p 的颜色是黑色，则执行自平衡
        if (p.color == BLACK)
            fixAfterDeletion(p);

        // <5.2> 删除 p 和其父节点的相互指向
        if (p.parent != null) {
            // 如果 p 是父节点的左子节点，则置空父节点的左子节点
            if (p == p.parent.left)
                p.parent.left = null;
            // 如果 p 是父节点的右子节点，则置空父节点的右子节点
            else if (p == p.parent.right)
                p.parent.right = null;
            // 置空 p 对父节点的指向
            p.parent = null;
        }
    }
}


static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    // <1> 如果 t 为空，则返回 null
    if (t == null)
        return null;
    // <2> 如果 t 的右子树非空，则取右子树的最小值
    else if (t.right != null) {
        // 先取右子树的根节点
        Entry<K,V> p = t.right;
        // 再取该根节点的做子树的最小值，即不断遍历左节点
        while (p.left != null)
            p = p.left;
        // 返回
        return p;
    // <3> 如果 t 的右子树为空
    } else {
        // 先获得 t 的父节点
        Entry<K,V> p = t.parent;
        // 不断向上遍历父节点，直到子节点 ch 不是父节点 p 的右子节点
        Entry<K,V> ch = t;
        while (p != null // 还有父节点
                && ch == p.right) { // 继续遍历的条件，必须是子节点 ch 是父节点 p 的右子节点
            ch = p;
            p = p.parent;
        }
        return p;
    }
}
```



删除后调整以保持红黑树特性

```java
// TODO
private void fixAfterDeletion(Entry<K,V> x) {
    while (x != root && colorOf(x) == BLACK) {
        if (x == leftOf(parentOf(x))) {
            Entry<K,V> sib = rightOf(parentOf(x));
            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);                   // 情况1
                setColor(parentOf(x), RED);             // 情况1
                rotateLeft(parentOf(x));                // 情况1
                sib = rightOf(parentOf(x));             // 情况1
            }
            if (colorOf(leftOf(sib))  == BLACK &&
                colorOf(rightOf(sib)) == BLACK) {
                setColor(sib, RED);                     // 情况2
                x = parentOf(x);                        // 情况2
            } else {
                if (colorOf(rightOf(sib)) == BLACK) {
                    setColor(leftOf(sib), BLACK);       // 情况3
                    setColor(sib, RED);                 // 情况3
                    rotateRight(sib);                   // 情况3
                    sib = rightOf(parentOf(x));         // 情况3
                }
                setColor(sib, colorOf(parentOf(x)));    // 情况4
                setColor(parentOf(x), BLACK);           // 情况4
                setColor(rightOf(sib), BLACK);          // 情况4
                rotateLeft(parentOf(x));                // 情况4
                x = root;                               // 情况4
            }
        } else { // 跟前四种情况对称
            Entry<K,V> sib = leftOf(parentOf(x));
            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);                   // 情况5
                setColor(parentOf(x), RED);             // 情况5
                rotateRight(parentOf(x));               // 情况5
                sib = leftOf(parentOf(x));              // 情况5
            }
            if (colorOf(rightOf(sib)) == BLACK &&
                colorOf(leftOf(sib)) == BLACK) {
                setColor(sib, RED);                     // 情况6
                x = parentOf(x);                        // 情况6
            } else {
                if (colorOf(leftOf(sib)) == BLACK) {
                    setColor(rightOf(sib), BLACK);      // 情况7
                    setColor(sib, RED);                 // 情况7
                    rotateLeft(sib);                    // 情况7
                    sib = leftOf(parentOf(x));          // 情况7
                }
                setColor(sib, colorOf(parentOf(x)));    // 情况8
                setColor(parentOf(x), BLACK);           // 情况8
                setColor(leftOf(sib), BLACK);           // 情况8
                rotateRight(parentOf(x));               // 情况8
                x = root;                               // 情况8
            }
        }
    }
    setColor(x, BLACK);
}
```

##### 查找元素

- 单个元素

```java
public V get(Object key) {
    // 获得 key 对应的 Entry 节点
    Entry<K,V> p = getEntry(key);
    // 返回 value 值
    return (p == null ? null : p.value);
}

final Entry<K,V> getEntry(Object key) { // 不使用 comparator 查找
    // Offload comparator-based version for sake of performance
    // 如果自定义了 comparator 比较器，则基于 comparator 比较来查找
    if (comparator != null)
        return getEntryUsingComparator(key);
    // 如果 key 为空，抛出异常
    if (key == null)
        throw new NullPointerException();
    @SuppressWarnings("unchecked")
    Comparable<? super K> k = (Comparable<? super K>) key;
    // 遍历红黑树
    Entry<K,V> p = root;
    while (p != null) {
        // 比较值
        int cmp = k.compareTo(p.key);
        // 如果 key 小于当前节点，则遍历左子树
        if (cmp < 0)
            p = p.left;
        // 如果 key 大于当前节点，则遍历右子树
        else if (cmp > 0)
            p = p.right;
        // 如果 key 相等，则返回该节点
        else
            return p;
    }
    // 查找不到，返回 null
    return null;
}

final Entry<K,V> getEntryUsingComparator(Object key) { // 使用 comparator 查找
    @SuppressWarnings("unchecked")
    K k = (K) key;
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        // 遍历红黑树
        Entry<K,V> p = root;
        while (p != null) {
            // 比较值
            int cmp = cpr.compare(k, p.key);
            // 如果 key 小于当前节点，则遍历左子树
            if (cmp < 0)
                p = p.left;
            // 如果 key 大于当前节点，则遍历右子树
            else if (cmp > 0)
                p = p.right;
            // 如果 key 相等，则返回该节点
            else
                return p;
        }
    }
    // 查找不到，返回 null
    return null;
}
```

- 查找接近的元素（在 NavigableMap 中，定义了四个查找接近元素的方法）
  - `#ceilingEntry(K key)` 方法，大于等于 `key` 的节点
  - `#higherEntry(K key)` 方法，大于 `key` 的节点
  - `#floorEntry(K key)` 方法，小于等于 `key` 的节点
  - `#lowerEntry(K key)` 方法，小于 `key` 的节点

大于等于

```java
public Map.Entry<K,V> ceilingEntry(K key) {
    // <1> 调用 #getCeilingEntry(K key) 方法，查找满足大于等于 key 的 Entry 节点。
    // <2> 调用 #exportEntry(TreeMap.Entry<K,V> e) 方法，创建不可变的 SimpleImmutableEntry 节点。这样，避免使用者直接修改节点，例如说修改 key 导致破坏红黑树。
    return exportEntry(getCeilingEntry(key));
}

static <K,V> Map.Entry<K,V> exportEntry(TreeMap.Entry<K,V> e) {
    return (e == null) ? null :
        new AbstractMap.SimpleImmutableEntry<>(e);
}

final Entry<K,V> getCeilingEntry(K key) {
    Entry<K,V> p = root;
    // <3> 循环二叉查找遍历红黑树
    while (p != null) {
        // <3.1> 比较 key
        int cmp = compare(key, p.key);
        // <3.2> 当前节点比 key 大，则遍历左子树，这样缩小节点的值
        if (cmp < 0) {
            // <3.2.1> 如果有左子树，则遍历左子树
            if (p.left != null)
                p = p.left;
            // <3.2.2.> 如果没有，则直接返回该节点
            else
                return p;
        // <3.3> 当前节点比 key 小，则遍历右子树，这样放大节点的值
        } else if (cmp > 0) {
            // <3.3.1> 如果有右子树，则遍历右子树
            if (p.right != null) {
                p = p.right;
            } else {
                // <3.3.2> 找到当前的后继节点
                Entry<K,V> parent = p.parent;
                Entry<K,V> ch = p;
                // 找到往 root 路径上的第一个右子节点
                while (parent != null && ch == parent.right) {
                    ch = parent;
                    parent = parent.parent;
                }
                return parent;
            }
        // <3.4> 如果相等，则返回该节点即可
        } else
            return p;
    }
    // <3.5> 
    return null;
}
```

大于

```java
public Map.Entry<K,V> higherEntry(K key) {
    return exportEntry(getHigherEntry(key));
}

final Entry<K,V> getHigherEntry(K key) {
    Entry<K,V> p = root;
    // 循环二叉查找遍历红黑树
    while (p != null) {
        // 比较 key
        int cmp = compare(key, p.key);
        // 当前节点比 key 大，则遍历左子树，这样缩小节点的值
        if (cmp < 0) {
            // 如果有左子树，则遍历左子树
            if (p.left != null)
                p = p.left;
            // 如果没有，则直接返回该节点
            else
                return p;
        // 当前节点比 key 小，则遍历右子树，这样放大节点的值
        } else {
            // 如果有右子树，则遍历右子树
            if (p.right != null) {
                p = p.right;
            } else {
                // 找到当前的后继节点
                Entry<K,V> parent = p.parent;
                Entry<K,V> ch = p;
                while (parent != null && ch == parent.right) {
                    ch = parent;
                    parent = parent.parent;
                }
                return parent;
            }
        }
        // <X> 此处，相等的情况下，不返回
    }
    // 查找不到，返回 null
    return null;
}
```

小于等于

```java
public Map.Entry<K,V> floorEntry(K key) {
    return exportEntry(getFloorEntry(key));
}

final Entry<K,V> getFloorEntry(K key) {
    Entry<K,V> p = root;
    // 循环二叉查找遍历红黑树
    while (p != null) {
        // 比较 key
        int cmp = compare(key, p.key);
        if (cmp > 0) {
            // 如果有右子树，则遍历右子树
            if (p.right != null)
                p = p.right;
            // 如果没有，则直接返回该节点
            else
                return p;
        // 当前节点比 key 小，则遍历右子树，这样放大节点的值
        } else if (cmp < 0) {
            // 如果有左子树，则遍历左子树
            if (p.left != null) {
                p = p.left;
            } else {
                // 找到当前节点的前继节点
                Entry<K,V> parent = p.parent;
                Entry<K,V> ch = p;
                // 找到往 root 路径上的第一个左子节点
                while (parent != null && ch == parent.left) {
                    ch = parent;
                    parent = parent.parent;
                }
                return parent;
            }
        // 如果相等，则返回该节点即可
        } else
            return p;

    }
    // 查找不到，返回 null
    return null;
}
```

小于

```java
public Map.Entry<K,V> lowerEntry(K key) {
    return exportEntry(getLowerEntry(key));
}

final Entry<K,V> getLowerEntry(K key) {
    Entry<K,V> p = root;
    // 循环二叉查找遍历红黑树
    while (p != null) {
        // 比较 key
        int cmp = compare(key, p.key);
        // 当前节点比 key 小，则遍历右子树，这样放大节点的值
        if (cmp > 0) {
            // 如果有右子树，则遍历右子树
            if (p.right != null)
                p = p.right;
            // 如果没有，则直接返回该节点
            else
                return p;
        // 当前节点比 key 大，则遍历左子树，这样缩小节点的值
        } else {
            // 如果有左子树，则遍历左子树
            if (p.left != null) {
                p = p.left;
            } else {
                // 找到当前节点的前继节点
                Entry<K,V> parent = p.parent;
                Entry<K,V> ch = p;
                while (parent != null && ch == parent.left) {
                    ch = parent;
                    parent = parent.parent;
                }
                return parent;
            }
        }
        // 此处，相等的情况下，不返回
    }
    // 查找不到，返回 null
    return null;
}
```

- 不返回 Entry 节点，只返回符合条件的 key 

```java
public K lowerKey(K key) {
    return keyOrNull(getLowerEntry(key));
}

public K floorKey(K key) {
    return keyOrNull(getFloorEntry(key));
}

public K ceilingKey(K key) {
    return keyOrNull(getCeilingEntry(key));
}

public K higherKey(K key) {
    return keyOrNull(getHigherEntry(key));
}

static <K,V> K keyOrNull(TreeMap.Entry<K,V> e) {
    return (e == null) ? null : e.key;
}
```

- 获得首尾的元素

首（最左边节点）

```java
public Map.Entry<K,V> firstEntry() {
    return exportEntry(getFirstEntry());
}

final Entry<K,V> getFirstEntry() {
    Entry<K,V> p = root;
    if (p != null)
        // 循环，不断遍历到左子节点，直到没有左子节点
        while (p.left != null)
            p = p.left;
    return p;
}


```

衍生方法

```java
public Map.Entry<K,V> pollFirstEntry() { // 获得并移除首个 Entry 节点
    // 获得首个 Entry 节点
    Entry<K,V> p = getFirstEntry();
    Map.Entry<K,V> result = exportEntry(p);
    // 如果存在，则进行删除。
    if (p != null)
        deleteEntry(p);
    return result;
}

public K firstKey() {
    return key(getFirstEntry());
}
static <K> K key(Entry<K,?> e) {
    if (e == null) // 如果不存在 e 元素，则抛出 NoSuchElementException 异常
        throw new NoSuchElementException();
    return e.key;
}
```



尾（最右边节点）

```java
public Map.Entry<K,V> lastEntry() {
    return exportEntry(getLastEntry());
}

final Entry<K,V> getLastEntry() {
    Entry<K,V> p = root;
    if (p != null)
        // 循环，不断遍历到右子节点，直到没有右子节点
        while (p.right != null)
            p = p.right;
    return p;
}
```

衍生方法

```java
public Map.Entry<K,V> pollLastEntry() { // 获得并移除尾部 Entry 节点
    // 获得尾部 Entry 节点
    Entry<K,V> p = getLastEntry();
    Map.Entry<K,V> result = exportEntry(p);
    // 如果存在，则进行删除。
    if (p != null)
        deleteEntry(p);
    return result;
}

public K lastKey() {
    return key(getLastEntry());
}
```



##### 清空

```java
public void clear() {
    // 增加修改次数
    modCount++;
    // key-value 数量置为 0
    size = 0;
    // 设置根节点为 null
    root = null;
}
```



##### 迭代器

- 抽象父类：PrivateEntryIterator

```java
abstract class PrivateEntryIterator<T> implements Iterator<T> {

    // 下一个节点
    Entry<K,V> next;
    
    // 最后返回的节点
    Entry<K,V> lastReturned;
    
    // 当前的修改次数
    int expectedModCount;

    PrivateEntryIterator(Entry<K,V> first) {
        expectedModCount = modCount;
        lastReturned = null;
        next = first;
    }

    public final boolean hasNext() {
        return next != null;
    }

    final Entry<K,V> nextEntry() { // 获得下一个 Entry 节点
        // 记录当前节点
        Entry<K,V> e = next;
        // 如果没有下一个，抛出 NoSuchElementException 异常
        if (e == null)
            throw new NoSuchElementException();
        // 如果发生了修改，抛出 ConcurrentModificationException 异常
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        // 获得 e 的后继节点，赋值给 next
        next = successor(e);
        // 记录最后返回的节点
        lastReturned = e;
        // 返回当前节点
        return e;
    }

    final Entry<K,V> prevEntry() { // 获得前一个 Entry 节点
        // 记录当前节点
        Entry<K,V> e = next;
        // 如果没有下一个，抛出 NoSuchElementException 异常
        if (e == null)
            throw new NoSuchElementException();
        // 如果发生了修改，抛出 ConcurrentModificationException 异常
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        // 获得 e 的前继节点，赋值给 next
        next = predecessor(e);
        // 记录最后返回的节点
        lastReturned = e;
        // 返回当前节点
        return e;
    }

    public void remove() { // 删除节点
        // 如果当前返回的节点不存在，则抛出 IllegalStateException 异常
        if (lastReturned == null)
            throw new IllegalStateException();
        // 如果发生了修改，抛出 ConcurrentModificationException 异常
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        // deleted entries are replaced by their successors
        // 在 lastReturned 左右节点都存在的时候，实际在 deleteEntry 方法中，是将后继节点替换到 lastReturned 中
        // 因此，next 需要指向 lastReturned
        if (lastReturned.left != null && lastReturned.right != null)
            next = lastReturned;
        // 删除节点
        deleteEntry(lastReturned);
        // 记录新的修改次数
        expectedModCount = modCount;
        // 置空 lastReturned
        lastReturned = null;
    }

}

// 和 secessor 方法相对应（在 TreeMap 中定义的）
static <K,V> Entry<K,V> predecessor(Entry<K,V> t) {
    // 如果 t 为空，则返回 null
    if (t == null)
        return null;
    // 如果 t 的左子树非空，则取左子树的最大值
    else if (t.left != null) {
        Entry<K,V> p = t.left;
        while (p.right != null)
            p = p.right;
        return p;
    // 如果 t 的左子树为空
    } else {
        // 先获得 t 的父节点
        Entry<K,V> p = t.parent;
        // 不断向上遍历父节点，直到子节点 ch 不是父节点 p 的左子节点
        Entry<K,V> ch = t;
        while (p != null // 还有父节点
                && ch == p.left) { // 继续遍历的条件，必须是子节点 ch 是父节点 p 的左子节点
            ch = p;
            p = p.parent;
        }
        return p;
    }
}
```



- key 的**正序**迭代器

```java
Iterator<K> keyIterator() {
    return new KeyIterator(getFirstEntry()); // 获得的是首个元素
}

final class KeyIterator extends PrivateEntryIterator<K> {

    KeyIterator(Entry<K,V> first) {
        super(first);
    }

    // 实现 next 方法，实现正序
    public K next() {
        return nextEntry().key;
    }

}
```



- key 的**倒序**迭代器

```java
Iterator<K> descendingKeyIterator() {
    return new DescendingKeyIterator(getLastEntry()); // 获得的是尾部元素
}

final class DescendingKeyIterator extends PrivateEntryIterator<K> {

    DescendingKeyIterator(Entry<K,V> first) {
        super(first);
    }

    // 实现 next 方法，实现倒序
    public K next() {
        return prevEntry().key;
    }

    // 重写 remove 方法，因为在 deleteEntry 方法中，在 lastReturned 左右节点都存在的时候，是将后继节点替换到 lastReturned 中。
    // 而这个逻辑，对于倒序遍历，没有影响，因为倒序时被删除节点的右子树已经被遍历完了。
    public void remove() {
        // 如果当前返回的节点不存在，则抛出 IllegalStateException 异常
        if (lastReturned == null)
            throw new IllegalStateException();
        // 如果发生了修改，抛出 ConcurrentModificationException 异常
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        // 删除节点
        deleteEntry(lastReturned);
        // 置空 lastReturned
        lastReturned = null;
        // 记录新的修改次数
        expectedModCount = modCount;
    }

}
```



- value 的**正序**迭代器

```java
final class ValueIterator extends PrivateEntryIterator<V> {

    ValueIterator(Entry<K,V> first) {
        super(first);
    }

    // 实现 next 方法，实现正序
    public V next() {
        return nextEntry().value;
    }

}
```

Entry 的**正序**迭代器

```java
final class EntryIterator extends PrivateEntryIterator<Map.Entry<K,V>> {

    EntryIterator(Entry<K,V> first) {
        super(first);
    }

    // 实现 next 方法，实现正序
    public Map.Entry<K,V> next() {
        return nextEntry();
    }

}
```



##### 转为集合

- keySet（**正序**的 key Set）

```java
/**
 * 正序的 KeySet 缓存对象
 */
private transient KeySet<K> navigableKeySet;

public Set<K> keySet() {
    return navigableKeySet();
}

public NavigableSet<K> navigableKeySet() {
    KeySet<K> nks = navigableKeySet;
    return (nks != null) ? nks : (navigableKeySet = new KeySet<>(this));
}

static final class KeySet<E> extends AbstractSet<E> implements NavigableSet<E> {
    private final NavigableMap<E, ?> m;
    KeySet(NavigableMap<E,?> map) { m = map; }

    public Iterator<E> iterator() {
        if (m instanceof TreeMap)
            return ((TreeMap<E,?>)m).keyIterator();
        else
            return ((TreeMap.NavigableSubMap<E,?>)m).keyIterator();
    }

    public Iterator<E> descendingIterator() {
        if (m instanceof TreeMap)
            return ((TreeMap<E,?>)m).descendingKeyIterator();
        else
            return ((TreeMap.NavigableSubMap<E,?>)m).descendingKeyIterator();
    }

    public int size() { return m.size(); }
    public boolean isEmpty() { return m.isEmpty(); }
    public boolean contains(Object o) { return m.containsKey(o); }
    public void clear() { m.clear(); }
    public E lower(E e) { return m.lowerKey(e); }
    public E floor(E e) { return m.floorKey(e); }
    public E ceiling(E e) { return m.ceilingKey(e); }
    public E higher(E e) { return m.higherKey(e); }
    public E first() { return m.firstKey(); }
    public E last() { return m.lastKey(); }
    public Comparator<? super E> comparator() { return m.comparator(); }
    public E pollFirst() {
        Map.Entry<E,?> e = m.pollFirstEntry();
        return (e == null) ? null : e.getKey();
    }
    public E pollLast() {
        Map.Entry<E,?> e = m.pollLastEntry();
        return (e == null) ? null : e.getKey();
    }
    public boolean remove(Object o) {
        int oldSize = size();
        m.remove(o);
        return size() != oldSize;
    }
    public NavigableSet<E> subSet(E fromElement, boolean fromInclusive,
                                  E toElement,   boolean toInclusive) {
        return new KeySet<>(m.subMap(fromElement, fromInclusive,
                                      toElement,   toInclusive));
    }
    public NavigableSet<E> headSet(E toElement, boolean inclusive) {
        return new KeySet<>(m.headMap(toElement, inclusive));
    }
    public NavigableSet<E> tailSet(E fromElement, boolean inclusive) {
        return new KeySet<>(m.tailMap(fromElement, inclusive));
    }
    public SortedSet<E> subSet(E fromElement, E toElement) {
        return subSet(fromElement, true, toElement, false);
    }
    public SortedSet<E> headSet(E toElement) {
        return headSet(toElement, false);
    }
    public SortedSet<E> tailSet(E fromElement) {
        return tailSet(fromElement, true);
    }
    public NavigableSet<E> descendingSet() {
        return new KeySet<>(m.descendingMap());
    }

    public Spliterator<E> spliterator() {
        return keySpliteratorFor(m);
    }
}
```



- descendingKeySet（**倒序**的 key Set）

```java
/**
 * 倒序的 NavigableMap 缓存对象
 */
private transient NavigableMap<K,V> descendingMap;

public NavigableSet<K> descendingKeySet() {
    return descendingMap().navigableKeySet();
}

public NavigableMap<K, V> descendingMap() {
    NavigableMap<K, V> km = descendingMap;
    return (km != null) ? km :
    	// 后面介绍
        (descendingMap = new DescendingSubMap<>(this, 
                                                true, null, true,
                                                true, null, true));
}

```

- values

```java
public Collection<V> values() {
    Collection<V> vs = values;
    if (vs == null) {
        vs = new Values();
        values = vs; // values 缓存，来自 AbstractMap 的属性
    }
    return vs;
}

class Values extends AbstractCollection<V> {
        public Iterator<V> iterator() {
            return new ValueIterator(getFirstEntry());
        }

        public int size() { return TreeMap.this.size(); }

        public boolean contains(Object o) { return TreeMap.this.containsValue(o); }

        public boolean remove(Object o) {
            for (Entry<K,V> e = getFirstEntry(); e != null; e = successor(e)) {
                if (valEquals(e.getValue(), o)) {
                    deleteEntry(e);
                    return true;
                }
            }
            return false;
        }

        public void clear() { TreeMap.this.clear(); }

        public Spliterator<V> spliterator() {
            return new ValueSpliterator<>(TreeMap.this, null, null, 0, -1, 0);
        }
    }
```

- entrySet

```java
/**
 * Entry 缓存集合
 */
private transient EntrySet entrySet;

public Set<Map.Entry<K,V>> entrySet() {
    EntrySet es = entrySet;
    return (es != null) ? es : (entrySet = new EntrySet());
}

class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        public Iterator<Map.Entry<K,V>> iterator() {
            return new EntryIterator(getFirstEntry());
        }

        public boolean contains(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> entry = (Map.Entry<?,?>) o;
            Object value = entry.getValue();
            Entry<K,V> p = getEntry(entry.getKey());
            return p != null && valEquals(p.getValue(), value);
        }

        public boolean remove(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> entry = (Map.Entry<?,?>) o;
            Object value = entry.getValue();
            Entry<K,V> p = getEntry(entry.getKey());
            if (p != null && valEquals(p.getValue(), value)) {
                deleteEntry(p);
                return true;
            }
            return false;
        }

        public int size() { return TreeMap.this.size(); }

        public void clear() { TreeMap.this.clear(); }

        public Spliterator<Map.Entry<K,V>> spliterator() {
            return new EntrySpliterator<>(TreeMap.this, null, null, 0, -1, 0);
        }
    }
```



##### 序列化

- 序列化

```java
@java.io.Serial
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException {
    // Write out the Comparator and any hidden stuff
    // 写入非静态属性、非 transient 属性
    s.defaultWriteObject();

    // Write out size (number of Mappings)
    // 写入 key-value 键值对数量
    s.writeInt(size);

    // Write out keys and values (alternating)
    // 写入具体的 key-value 键值对
    for (Map.Entry<K, V> e : entrySet()) {
        s.writeObject(e.getKey());
        s.writeObject(e.getValue());
    }
}

```



- 反序列化

```java
@java.io.Serial
private void readObject(final java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    // Read in the Comparator and any hidden stuff
    // 读取非静态属性、非 transient 属性
    s.defaultReadObject();

    // Read in size
    // 读取 key-value 键值对数量 size
    int size = s.readInt();

    // 使用输入流，构建红黑树。
    // 因为序列化时，已经是顺序的，所以输入流也是顺序的
    buildFromSorted(size, null, s, null); // 注意，此时传入的是 s 参数，输入流
}
```



##### 范围查找

在 SortedMap 接口中，定义了按照 key 查找范围，返回子 SortedMap 结果的方法

- `#subMap(K fromKey, K toKey)`
- `#headMap(K toKey)`
- `#tailMap(K fromKey)`



在 NavigableMap 中，定义了按照 key 查找范围，返回子 NavigableMap 结果的方法：

- `#subMap(K fromKey, K toKey)`
- `#subMap(K fromKey, boolean fromInclusive, K toKey, boolean toInclusive)`
- `#headMap(K toKey)`
- `#headMap(K toKey, boolean inclusive)`
- `#tailMap(K fromKey)`
- `#tailMap(K fromKey, boolean inclusive)`



TreeMap 对上述接口，实现如下方法：

```java
// subMap 组
public SortedMap<K,V> subMap(K fromKey, K toKey) {
    return subMap(fromKey, true, toKey, false);
}
public NavigableMap<K,V> subMap(K fromKey, boolean fromInclusive,
                                K toKey,   boolean toInclusive) {
    return new AscendingSubMap<>(this,
                                 false, fromKey, fromInclusive,
                                 false, toKey,   toInclusive);
}

// headMap 组
public SortedMap<K,V> headMap(K toKey) {
    return headMap(toKey, false);
}
public NavigableMap<K,V> headMap(K toKey, boolean inclusive) {
    return new AscendingSubMap<>(this,
                                 true,  null,  true,
                                 false, toKey, inclusive);
}

// tailMap 组
public SortedMap<K,V> tailMap(K fromKey) {
    return tailMap(fromKey, true);
}
public NavigableMap<K,V> tailMap(K fromKey, boolean inclusive) {
    return new AscendingSubMap<>(this,
                                 false, fromKey, inclusive,
                                 true,  null,    true);
}
```



###### NavigableSubMap



![在这里插入图片描述](https://img-blog.csdnimg.cn/20200929170526946.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


属性

- 一句话总结：都是调用的内部 TreeMap 对象的方法，只是调用前判断范围或者改变顺序

```java
final TreeMap<K,V> m;

/**
 * lo - 开始位置
 * hi - 结束位置
 */
final K lo, hi;
/**
 * fromStart - 是否从 TreeMap 开头开始。如果是的话，{@link #lo} 可以不传
 * toEnd - 是否从 TreeMap 结尾结束。如果是的话，{@link #hi} 可以不传
 */
final boolean fromStart, toEnd;
/**
 * loInclusive - 是否包含 key 为 {@link #lo}  的元素
 * hiInclusive - 是否包含 key 为 {@link #hi} 的元素
 */
final boolean loInclusive, hiInclusive;

NavigableSubMap(TreeMap<K,V> m,
                boolean fromStart, K lo, boolean loInclusive,
                boolean toEnd,     K hi, boolean hiInclusive) {
    // 如果既不从开头开始，又不从结尾结束，那么就要校验 lo 小于 hi ，否则抛出 IllegalArgumentException 异常
    if (!fromStart && !toEnd) {
        if (m.compare(lo, hi) > 0)
            throw new IllegalArgumentException("fromKey > toKey");
    } else {
        // 如果不从开头开始，则进行 lo 的类型校验
        if (!fromStart) // type check
            m.compare(lo, lo);
        // 如果不从结尾结束，则进行 hi 的类型校验
        if (!toEnd)
            m.compare(hi, hi);
    }

    // 赋值属性
    this.m = m;
    this.fromStart = fromStart;
    this.lo = lo;
    this.loInclusive = loInclusive;
    this.toEnd = toEnd;
    this.hi = hi;
    this.hiInclusive = hiInclusive;
}
```

范围校验

```java
// 因为 NavigableSubMap 是 TreeMap 的子 NavigableMap，所以其所有的操作，不能超过 TreeMap 范围
final boolean inRange(Object key) {
    return !tooLow(key) 
            && !tooHigh(key);
}


final boolean tooLow(Object key) {
    if (!fromStart) {
        // 比较 key
        int c = m.compare(key, lo);
        if (c < 0 // 如果小于，则肯定过小
                || (c == 0 && !loInclusive)) // 如果相等，则进一步判断是否 !loInclusive ，不包含 lo 的情况
            return true;
    }
    return false;
}


final boolean tooHigh(Object key) {
    if (!toEnd) {
        // 比较 key
        int c = m.compare(key, hi);
        if (c > 0 // 如果大于，则肯定过大
                || (c == 0 && !hiInclusive)) // 如果相等，则进一步判断是否 !hiInclusive ，不包含 high 的情况
            return true;
    }
    return false;
}
```

​	

添加单个元素

```java
public final V put(K key, V value) {
    // 校验 key 的范围，如果不在，则抛出 IllegalArgumentException 异常
    if (!inRange(key))
        throw new IllegalArgumentException("key out of range");
    // 执行添加单个元素
    return m.put(key, value);
}
```



删除单个元素

```java
public final V remove(Object key) {
    return !inRange(key) // 校验 key 的范围
            ? null : // 如果不在，则返回 null
            m.remove(key); // 执行移除单个元素
}
```



获得单个元素

```java
public final V get(Object key) {
    return !inRange(key) // 校验 key 的范围
            ? null : // 如果不在，则返回 null
            m.get(key); // 执行获得单个元素
}
```



模板方法

- 因为子类的**排序规则不同**，所以 NavigableSubMap 定义了如下抽象方法，交给子类实现

  ```java
  abstract TreeMap.Entry<K,V> subLowest();
  abstract TreeMap.Entry<K,V> subHighest();
  abstract TreeMap.Entry<K,V> subCeiling(K key);
  abstract TreeMap.Entry<K,V> subHigher(K key);
  abstract TreeMap.Entry<K,V> subFloor(K key);
  abstract TreeMap.Entry<K,V> subLower(K key);
  ```

  

- NavigableSubMap 为了子类实现更方便，提供了如下方法

  ```java
  // 获得 NavigableSubMap 开始位置的 Entry 节点
  final TreeMap.Entry<K,V> absLowest() { 
      TreeMap.Entry<K,V> e =
          (fromStart ?  m.getFirstEntry() : // 如果从 TreeMap 开始，则获得 TreeMap 的首个 Entry 节点
           (loInclusive ? m.getCeilingEntry(lo) : // 如果 key 从 lo 开始（包含），则获得 TreeMap 从 lo 开始（>=）最接近的 Entry 节点
                          m.getHigherEntry(lo))); // 如果 key 从 lo 开始（不包含），则获得 TreeMap 从 lo 开始(>)最接近的 Entry 节点
      return (e == null || tooHigh(e.key)) /** 超过 key 过大 **/ ? null : e;
  }
  
  // 获得 NavigableSubMap 结束位置的 Entry 节点
  final TreeMap.Entry<K,V> absHighest() { 
      TreeMap.Entry<K,V> e =
          (toEnd ?  m.getLastEntry() : // 如果从 TreeMap 开始，则获得 TreeMap 的尾部 Entry 节点
           (hiInclusive ?  m.getFloorEntry(hi) : // 如果 key 从 hi 开始（包含），则获得 TreeMap 从 hi 开始（<=）最接近的 Entry 节点
                           m.getLowerEntry(hi))); // 如果 key 从 lo 开始（不包含），则获得 TreeMap 从 lo 开始(<)最接近的 Entry 节点
      return (e == null || tooLow(e.key)) /** 超过 key 过小 **/ ? null : e;
  }
  
  // 获得 NavigableSubMap >= key 最接近的 Entry 节点
  final TreeMap.Entry<K,V> absCeiling(K key) { 
      // 如果 key 过小，则只能通过 `#absLowest()` 方法，获得 NavigableSubMap 开始位置的 Entry 节点
      if (tooLow(key))
          return absLowest();
      // 获得 NavigableSubMap >= key 最接近的 Entry 节点
      TreeMap.Entry<K,V> e = m.getCeilingEntry(key);
      return (e == null || tooHigh(e.key)) /** 超过 key 过大 **/ ? null : e;
  }
  
  // 获得 NavigableSubMap > key 最接近的 Entry 节点
  final TreeMap.Entry<K,V> absHigher(K key) { 
      // 如果 key 过小，则只能通过 `#absLowest()` 方法，获得 NavigableSubMap 开始位置的 Entry 节点
      if (tooLow(key))
          return absLowest();
      // 获得 NavigableSubMap > key 最接近的 Entry 节点
      TreeMap.Entry<K,V> e = m.getHigherEntry(key);
      return (e == null || tooHigh(e.key)) /** 超过 key 过大 **/ ? null : e;
  }
  
  // 获得 NavigableSubMap <= key 最接近的 Entry 节点
  final TreeMap.Entry<K,V> absFloor(K key) { 
      // 如果 key 过大，则只能通过 `#absHighest()` 方法，获得 NavigableSubMap 结束位置的 Entry 节点
      if (tooHigh(key))
          return absHighest();
      // 获得 NavigableSubMap <= key 最接近的 Entry 节点
      TreeMap.Entry<K,V> e = m.getFloorEntry(key);
      return (e == null || tooLow(e.key)) /** 超过 key 过小 **/ ? null : e;
  }
  
  // 获得 NavigableSubMap < key 最接近的 Entry 节点
  final TreeMap.Entry<K,V> absLower(K key) { 
      // 如果 key 过大，则只能通过 `#absHighest()` 方法，获得 NavigableSubMap 结束位置的 Entry 节点
      if (tooHigh(key))
          return absHighest();
      // 获得 NavigableSubMap < key 最接近的 Entry 节点
      TreeMap.Entry<K,V> e = m.getLowerEntry(key);
      return (e == null || tooLow(e.key)) /** 超过 key 过小 **/ ? null : e;
  }
  
  // 获得 TreeMap 最大 key 的 Entry 节点，用于升序遍历的时候。注意，是 TreeMap 。
  final TreeMap.Entry<K,V> absHighFence() { 
      // toEnd 为真时，意味着无限大，所以返回 null
      return (toEnd ? null : (hiInclusive ?
                              m.getHigherEntry(hi) : // 获得 TreeMap > hi 最接近的 Entry 节点。
                              m.getCeilingEntry(hi))); // 获得 TreeMap => hi 最接近的 Entry 节点。
  }
  
  // 获得 TreeMap 最小 key 的 Entry 节点，用于降序遍历的时候。注意，是 TreeMap 。
  final TreeMap.Entry<K,V> absLowFence() { 
      return (fromStart ? null : (loInclusive ?
                                  m.getLowerEntry(lo) :  // 获得 TreeMap < lo 最接近的 Entry 节点。
                                  m.getFloorEntry(lo))); // 获得 TreeMap <= lo 最接近的 Entry 节点。
  }
  ```

  

查找接近的元素

```java
public final Map.Entry<K,V> ceilingEntry(K key) {
    return exportEntry(subCeiling(key));
}
public final K ceilingKey(K key) {
    return keyOrNull(subCeiling(key));
}

public final Map.Entry<K,V> higherEntry(K key) {
    return exportEntry(subHigher(key));
}
public final K higherKey(K key) {
    return keyOrNull(subHigher(key));
}

public final Map.Entry<K,V> floorEntry(K key) {
    return exportEntry(subFloor(key));
}
public final K floorKey(K key) {
    return keyOrNull(subFloor(key));
}

public final Map.Entry<K,V> lowerEntry(K key) {
    return exportEntry(subLower(key));
}
public final K lowerKey(K key) {
    return keyOrNull(subLower(key));
}
```



获得首尾的元素

```java
/**
 * 首节点
 */
public final Map.Entry<K,V> firstEntry() {
    return exportEntry(subLowest());
}

public final K firstKey() {
    return key(subLowest());
}

public final Map.Entry<K,V> pollFirstEntry() {
    // 获得 NavigableSubMap 的首个 Entry 节点
    TreeMap.Entry<K,V> e = subLowest();
    Map.Entry<K,V> result = exportEntry(e);
    // 如果存在，则进行删除。
    if (e != null)
        m.deleteEntry(e);
    return result;
}
```



```java
/**
 * 尾节点
 */
public final Map.Entry<K,V> lastEntry() {
    return exportEntry(subHighest());
}

public final K lastKey() {
    return key(subHighest());
}

public final Map.Entry<K,V> pollLastEntry() {
    // 获得 NavigableSubMap 的尾部 Entry 节点
    TreeMap.Entry<K,V> e = subHighest();
    Map.Entry<K,V> result = exportEntry(e);
    // 如果存在，则进行删除。
    if (e != null)
        m.deleteEntry(e);
    return result;
}
```



迭代器

```java
abstract class SubMapIterator<T> implements Iterator<T> {

    TreeMap.Entry<K,V> lastReturned;
  
    TreeMap.Entry<K,V> next;
    
    int expectedModCount;
    
    /**
     * 遍历的上限 key 。
     *
     * 如果遍历到该 key ，说明已经超过范围了
     */
    final Object fenceKey;
   
    SubMapIterator(TreeMap.Entry<K,V> first,
                   TreeMap.Entry<K,V> fence) {
        expectedModCount = m.modCount;
        lastReturned = null;
        next = first;
        fenceKey = fence == null ? UNBOUNDED /** 无界限 **/ : fence.key;
    }

    public final boolean hasNext() { // 是否还有下一个节点
        return next != null && next.key != fenceKey;
    }

    final TreeMap.Entry<K,V> nextEntry() { // 获得下一个 Entry 节点
        // 记录当前节点
        TreeMap.Entry<K,V> e = next;
        // 如果没有下一个，抛出 NoSuchElementException 异常
        if (e == null || e.key == fenceKey)
            throw new NoSuchElementException();
        // 如果发生了修改，抛出 ConcurrentModificationException 异常
        if (m.modCount != expectedModCount)
            throw new ConcurrentModificationException();
        // 获得 e 的后继节点，赋值给 next
        next = successor(e);
        // 记录最后返回的节点
        lastReturned = e;
        // 返回当前节点
        return e;
    }

    final TreeMap.Entry<K,V> prevEntry() { // 获得前一个 Entry 节点
        // 记录当前节点
        TreeMap.Entry<K,V> e = next;
        // 如果没有下一个，抛出 NoSuchElementException 异常
        if (e == null || e.key == fenceKey)
            throw new NoSuchElementException();
        // 如果发生了修改，抛出 ConcurrentModificationException 异常
        if (m.modCount != expectedModCount)
            throw new ConcurrentModificationException();
        // 获得 e 的前继节点，赋值给 next
        next = predecessor(e);
        // 记录最后返回的节点
        lastReturned = e;
        // 返回当前节点
        return e;
    }

    final void removeAscending() { // 删除节点（顺序遍历的情况下）
        // 如果当前返回的节点不存在，则抛出 IllegalStateException 异常
        if (lastReturned == null)
            throw new IllegalStateException();
        // 如果发生了修改，抛出 ConcurrentModificationException 异常
        if (m.modCount != expectedModCount)
            throw new ConcurrentModificationException();
        // deleted entries are replaced by their successors
        // 在 lastReturned 左右节点都存在的时候，实际在 deleteEntry 方法中，是将后继节点替换到 lastReturned 中
        // 因此，next 需要指向 lastReturned
        if (lastReturned.left != null && lastReturned.right != null)
            next = lastReturned;
        // 删除节点
        m.deleteEntry(lastReturned);
        // 置空 lastReturned
        lastReturned = null;
        // 记录新的修改次数
        expectedModCount = m.modCount;
    }

    final void removeDescending() { // 删除节点倒序遍历的情况下）
        // 如果当前返回的节点不存在，则抛出 IllegalStateException 异常
        if (lastReturned == null)
            throw new IllegalStateException();
        // 如果发生了修改，抛出 ConcurrentModificationException 异常
        if (m.modCount != expectedModCount)
            throw new ConcurrentModificationException();
        // 删除节点
        m.deleteEntry(lastReturned);
        // 置空 lastReturned
        lastReturned = null;
        // 记录新的修改次数
        expectedModCount = m.modCount;
    }

}
```

```java
/**
 * key 的正序迭代器
 */
final class SubMapKeyIterator extends SubMapIterator<K>
    implements Spliterator<K> {

    SubMapKeyIterator(TreeMap.Entry<K,V> first,
                      TreeMap.Entry<K,V> fence) {
        super(first, fence);
    }

    // 实现 next 方法，实现正序
    public K next() {
        return nextEntry().key;
    }

    // 实现 remove 方法，实现正序的移除方法
    public void remove() {
        removeAscending();
    }

    // 其余方法略
}
```

```java
/**
 * key 的倒序迭代器
 */
final class DescendingSubMapKeyIterator extends SubMapIterator<K>
    implements Spliterator<K> {

    DescendingSubMapKeyIterator(TreeMap.Entry<K,V> last,
                                TreeMap.Entry<K,V> fence) {
        super(last, fence);
    }

    // 实现 next 方法，实现倒序
    public K next() {
        return prevEntry().key;
    }

    // 实现 remove 方法，实现倒序的移除方法
    public void remove() {
        removeDescending();
    }

    // 其余方法略
}
```

```java
/**
 * Entry 的正序迭代器
 */
final class SubMapEntryIterator extends SubMapIterator<Map.Entry<K,V>> {

    SubMapEntryIterator(TreeMap.Entry<K,V> first,
                        TreeMap.Entry<K,V> fence) {
        super(first, fence);
    }

    // 实现 next 方法，实现正序
    public Map.Entry<K,V> next() {
        return nextEntry();
    }

    // 实现 remove 方法，实现正序的移除方法
    public void remove() {
        removeAscending();
    }

}
```

```java
/**
 * Entry 的倒序迭代器
 */
final class DescendingSubMapEntryIterator extends SubMapIterator<Map.Entry<K,V>> {

    DescendingSubMapEntryIterator(TreeMap.Entry<K,V> last,
                                  TreeMap.Entry<K,V> fence) {
        super(last, fence);
    }

    // 实现 next 方法，实现倒序
    public Map.Entry<K,V> next() {
        return prevEntry();
    }

    // 实现 remove 方法，实现倒序的移除方法
    public void remove() {
        removeDescending();
    }
}
```

转为 Set

```java
/**
 * 正序的 KeySet 缓存对象
 */
transient KeySet<K> navigableKeySetView;
        
public final Set<K> keySet() {
    return navigableKeySet();
}

public final NavigableSet<K> navigableKeySet() {
    KeySet<K> nksv = navigableKeySetView;
    return (nksv != null) ? nksv :
    	// 传入 this 后 TreeMap.KeySet 每一个方法都会调用到 NavigableSubMap 子类的对象身上
        (navigableKeySetView = new TreeMap.KeySet<>(this));
}
```

###### AscendingSubMap 

（正序）

查找接近的元素

```java
TreeMap.Entry<K,V> subLowest()       { return absLowest(); }
TreeMap.Entry<K,V> subHighest()      { return absHighest(); }
TreeMap.Entry<K,V> subCeiling(K key) { return absCeiling(key); }
TreeMap.Entry<K,V> subHigher(K key)  { return absHigher(key); }
TreeMap.Entry<K,V> subFloor(K key)   { return absFloor(key); }
TreeMap.Entry<K,V> subLower(K key)   { return absLower(key); }
```

获得迭代器

```java
Iterator<K> keyIterator() {
    return new SubMapKeyIterator(absLowest(), absHighFence());
}

Iterator<K> descendingKeyIterator() {
    return new DescendingSubMapKeyIterator(absHighest(), absLowFence());
}
```

查找范围的元素

```java
public NavigableMap<K,V> subMap(K fromKey, boolean fromInclusive,
                                K toKey,   boolean toInclusive) {
    // 如果不在范围，抛出 IllegalArgumentException 异常
    if (!inRange(fromKey, fromInclusive))
        throw new IllegalArgumentException("fromKey out of range");
    // 如果不在范围，抛出 IllegalArgumentException 异常
    if (!inRange(toKey, toInclusive))
        throw new IllegalArgumentException("toKey out of range");
    // 创建 AscendingSubMap 对象
    return new AscendingSubMap<>(m,
                                 false, fromKey, fromInclusive,
                                 false, toKey,   toInclusive);
}

public NavigableMap<K,V> headMap(K toKey, boolean inclusive) {
    // 如果不在范围，抛出 IllegalArgumentException 异常
    if (!inRange(toKey, inclusive))
        throw new IllegalArgumentException("toKey out of range");
    // 创建 AscendingSubMap 对象
    return new AscendingSubMap<>(m,
                                 fromStart, lo,    loInclusive,
                                 false,     toKey, inclusive);
}

public NavigableMap<K,V> tailMap(K fromKey, boolean inclusive) {
    // 如果不在范围，抛出 IllegalArgumentException 异常
    if (!inRange(fromKey, inclusive))
        throw new IllegalArgumentException("fromKey out of range");
    // 创建 AscendingSubMap 对象
    return new AscendingSubMap<>(m,
                                 false, fromKey, inclusive,
                                 toEnd, hi,      hiInclusive);
}
```

###### DescendingSubMap

（倒序）

查找接近的元素

```java
TreeMap.Entry<K,V> subLowest()       { return absHighest(); }
TreeMap.Entry<K,V> subHighest()      { return absLowest(); }
TreeMap.Entry<K,V> subCeiling(K key) { return absFloor(key); }
TreeMap.Entry<K,V> subHigher(K key)  { return absLower(key); }
TreeMap.Entry<K,V> subFloor(K key)   { return absCeiling(key); }
TreeMap.Entry<K,V> subLower(K key)   { return absHigher(key); }
```

获得迭代器

```java
Iterator<K> keyIterator() {
    return new DescendingSubMapKeyIterator(absHighest(), absLowFence());
}

Iterator<K> descendingKeyIterator() {
    return new SubMapKeyIterator(absLowest(), absHighFence());
}
```

查找范围的元素

```java
public NavigableMap<K,V> subMap(K fromKey, boolean fromInclusive,
                                K toKey,   boolean toInclusive) {
    // 如果不在范围，抛出 IllegalArgumentException 异常
    if (!inRange(fromKey, fromInclusive))
        throw new IllegalArgumentException("fromKey out of range");
    // 如果不在范围，抛出 IllegalArgumentException 异常
    if (!inRange(toKey, toInclusive))
        throw new IllegalArgumentException("toKey out of range");
    // 创建 DescendingSubMap 对象
    return new DescendingSubMap<>(m,
                                  false, toKey,   toInclusive,
                                  false, fromKey, fromInclusive);
}

public NavigableMap<K,V> headMap(K toKey, boolean inclusive) {
    // 如果不在范围，抛出 IllegalArgumentException 异常
    if (!inRange(toKey, inclusive))
        throw new IllegalArgumentException("toKey out of range");
    // 创建 DescendingSubMap 对象
    return new DescendingSubMap<>(m,
                                  false, toKey, inclusive,
                                  toEnd, hi,    hiInclusive);
}

public NavigableMap<K,V> tailMap(K fromKey, boolean inclusive) {
    // 如果不在范围，抛出 IllegalArgumentException 异常
    if (!inRange(fromKey, inclusive))
        throw new IllegalArgumentException("fromKey out of range");
    // 创建 DescendingSubMap 对象
    return new DescendingSubMap<>(m,
                                  fromStart, lo, loInclusive,
                                  false, fromKey, inclusive);
}
```



##### 小结

- TreeMap 按照 key 的**顺序**的 Map 实现类，底层采用**红黑树**来实现存储。
- TreeMap 因为采用树结构，所以无需初始考虑像 HashMap 考虑**容量**问题，也不存在扩容问题。
- TreeMap 的 **key** 不允许为空( `null` )，可能是因为红黑树是一颗二叉查找树，需要对 key 进行排序。
- TreeMap 的查找、添加、删除 key-value 键值对的**平均**时间复杂度为 `O(logN)` 。原因是，TreeMap 采用红黑树，操作都需要经过二分查找，而二分查找的时间复杂度是 `O(logN)` 。
- 相比 HashMap 来说，TreeMap 不仅仅支持指定 key 的查找，也支持 key **范围**的查找。当然，这也得益于 TreeMap 数据结构能够提供的有序特性。

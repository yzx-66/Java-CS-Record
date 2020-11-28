# TreeMap

![](https://img-blog.csdnimg.cn/20200929170440927.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


- NaviableMap 定义了一些有方向操作

	![在这里插入图片描述](https://img-blog.csdnimg.cn/20200929170509671.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



红黑树

1. 每个节点要么是红色，要么是黑色。
2. 根节点必须是黑色
3. 红色节点不能连续（也即是，红色节点的孩子和父亲都不能是红色）。
4. 对于每个节点，从该点至`null`（树尾端）的任何路径，都含有相同个数的黑色节点。

## 数据结构

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

  

## 构造方法

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





## 添加元素

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



## 扩容

不需要扩容，直接增加节点即可，只有数组会涉及扩容。



## 删除元素

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

## 清空

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

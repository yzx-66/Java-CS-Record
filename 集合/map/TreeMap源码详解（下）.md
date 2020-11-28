## 查找元素

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



## 迭代器

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



## 转为集合

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



## 序列化

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

## 范围查找

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



### NavigableSubMap



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

### AscendingSubMap 

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

### DescendingSubMap

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



## 小结

- TreeMap 按照 key 的**顺序**的 Map 实现类，底层采用**红黑树**来实现存储。
- TreeMap 因为采用树结构，所以无需初始考虑像 HashMap 考虑**容量**问题，也不存在扩容问题。
- TreeMap 的 **key** 不允许为空( `null` )，可能是因为红黑树是一颗二叉查找树，需要对 key 进行排序。
- TreeMap 的查找、添加、删除 key-value 键值对的**平均**时间复杂度为 `O(logN)` 。原因是，TreeMap 采用红黑树，操作都需要经过二分查找，而二分查找的时间复杂度是 `O(logN)` 。
- 相比 HashMap 来说，TreeMap 不仅仅支持指定 key 的查找，也支持 key **范围**的查找。当然，这也得益于 TreeMap 数据结构能够提供的有序特性。

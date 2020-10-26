#### LinkedHashMap

![](https://img-blog.csdnimg.cn/2020092917015981.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


- LinkedHashMap 继承自 HashMap 类，基本上没有重写 HashMap 的方法，只是把 HashMap 空实现的预留拓展点实现了。



##### 数据结构

- Map 拓展属性

```java
/**
 * 头节点。
 *
 * 越老的节点，放在越前面。所以头节点，指向链表的开头
 */
transient LinkedHashMap.Entry<K,V> head;

/**
 * 尾节点
 *
 * 越新的节点，放在越后面。所以尾节点，指向链表的结尾
 */
transient LinkedHashMap.Entry<K,V> tail;

/**
 * 是否按照访问的顺序。
 *
 * true ：按照 key-value 的插入和访问顺序进行排序（即放置到链表的结尾，被 tail 指向）。
 * false ：按照 key-value 的插入顺序进行排序，访问该节点不会给该节点重新排序，
 *		   如果插入的 key 对应的 Entry 节点已经存在，也会被放到结尾。
 */
final boolean accessOrder;
```



- 数据节点 Entry (拓展了前后指针)

```java
static class Entry<K,V> extends HashMap.Node<K,V> {

    Entry<K,V> before, // 前一个节点
            after; // 后一个节点

    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```



##### 构造方法

LinkedHashMap 一共有 5 个构造方法，其中四个和 HashMap 相同，只是多初始化 `accessOrder = false` 。所以，默认使用**插入**顺序进行访问。

- 允许自定义 `accessOrder` 属性。

```java
public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```



##### 创建节点

和 HashMap 的 newNode() 不同点是会让 tail 指向该节点，并且 Entry 是双向链表

```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    // <1> 创建 Entry 节点
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<>(hash, key, value, e);
    // <2> 添加到结尾
    linkNodeLast(p);
    // 返回
    return p;
}

// 添加到结尾,被 tail 指向
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    // 记录原尾节点到 last 中
    LinkedHashMap.Entry<K,V> last = tail;
    // 设置 tail 指向 p ，变更新的尾节点
    tail = p;
    // 如果原尾节点 last 为空，说明 head 也为空，所以 head 也指向 p
    if (last == null)
        head = p;
    // last <=> p ，相互指向
    else {
        p.before = last;
        last.after = p;
    }
}
```



##### 后置拓展点

- afterNodeAccess()

在 `accessOrder` 属性为 `true` 时，当 Entry 节点被访问时，放置到链表的结尾，被 `tail` 指向。

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    // accessOrder 判断必须是满足按访问顺序。
    // (last = tail) != e 将 tail 赋值给 last,并且判断是否 e 已经是队尾。如果是队尾，就不处理了。
    if (accessOrder && (last = tail) != e) {
        
        // 将 e 赋值给 p 【因为要 Node 类型转换成 Entry 类型】
        // 同时 b、a 分别是 e 的前后节点
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        // 第一步，将 p 从链表中移除
        p.after = null;
        
        // 处理 b 的下一个节点指向 a
        if (b == null)
            head = a;
        else
            b.after = a;
        
        // 处理 a 的前一个节点指向 b       
        if (a != null)
            a.before = b;
        else
            last = b;
        
        // 第二步，将 p 添加到链表的尾巴。实际这里的代码，和 linkNodeLast 是一致的。        
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        
        // tail 指向 p ，实际就是 e 。
        tail = p;
        // 增加修改次数
        ++modCount;
    }
}
```

补充：

因为 HashMap 提供的 `#get(Object key)` 和 `#getOrDefault(Object key, V defaultValue)` 方法，并未调用 `#afterNodeAccess(Node<K,V> e)` 方法，这在按照**读取**顺序访问显然不行，所以 LinkedHashMap 重写这两方法的代码

```java
public V get(Object key) {
    // 获得 key 对应的 Node
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    // 如果访问到，回调节点被访问
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}

public V getOrDefault(Object key, V defaultValue) {
    // 获得 key 对应的 Node
    Node<K,V> e;
   if ((e = getNode(hash(key), key)) == null)
       return defaultValue;
    // 如果访问到，回调节点被访问
    if (accessOrder)
       afterNodeAccess(e);
   return e.value;
}
```



- afterNodeInsertion()

当 Entry 节点被添加时，放置到链表的结尾，被 `tail` 指向。

```java
// evict 翻译为驱逐，表示是否允许移除元素
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    // first = head 记录当前头节点。因为移除从头开始，最老
    // <1> removeEldestEntry(first) 判断是否满足移除最老节点
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        // <2> 移除指定节点
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    // 默认情况下，都不移除最老的节点
    return false;
}
```



拓展：

如何基于 LinkedHashMap 实现 LRU 算法的缓存。

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {

    private final int CACHE_SIZE;

    /**
     * 传递进来最多能缓存多少数据
     *
     * @param cacheSize 缓存大小
     */
    public LRUCache(int cacheSize) {
        // true 表示让 LinkedHashMap 按照访问顺序来进行排序，最近访问的放在头部，最老访问的放在尾部。
        super((int) Math.ceil(cacheSize / 0.75) + 1, 0.75f, true);
        CACHE_SIZE = cacheSize;
    }

    /**
     * 添加元素的时候会调用这个拓展点，如果超过了 CACHE_SIZE，就会删除最老的节点，从而实现 LRU。
     */
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // 当 map 中的数据量大于指定的缓存个数的时候，就自动删除最老的数据。
        return size() > CACHE_SIZE;
    }
}
```



- afterNodeRemoval()

当 Entry 节点被添加时，从链表中删除该节点，让 tail 指向倒数第二个节点

```java
void afterNodeRemoval(Node<K,V> e) { // unlink
    // 将 e 赋值给 p 【因为要 Node 类型转换成 Entry 类型】
    // 同时 b、a 分别是 e 的前后节点
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    // 将 p 从链表中移除
    p.before = p.after = null;
    // 处理 b 的下一个节点指向 a
    if (b == null)
        head = a;
    else
        b.after = a;
    // 处理 a 的前一个节点指向 b
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```



##### 清空

```java
public void clear() {
    // 清空
    super.clear();
    // 标记 head 和 tail 为 null
    head = tail = null;
}
```



##### 迭代器

- 转换成 Set/Collection

keySet

```java
public Set<K> keySet() {
    // 获得 keySet 缓存
    Set<K> ks = keySet;
    // 如果不存在，则进行创建
    if (ks == null) {
        ks = new LinkedKeySet(); // LinkedKeySet 是 LinkedHashMap 自定义的
        keySet = ks;
    }
    return ks;
}


final class LinkedKeySet extends AbstractSet<K> {
    
    public final Iterator<K> iterator() {
        return new LinkedKeyIterator(); 
    }
    
    // 通过head + tail 构成的双向链表进行遍历，而不是遍历 table 数组。
    public final void forEach(Consumer<? super K> action) {
        if (action == null)
            throw new NullPointerException();
        int mc = modCount;
        // 链表遍历
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after)
            action.accept(e.key);
        if (modCount != mc)
            throw new ConcurrentModificationException();
    }

    //------------------直接调用 LinkedHashMap 的方法 ------------------------
    public final int size()                 { return size; }
    public final void clear()               { LinkedHashMap.this.clear(); }
    
    public final boolean contains(Object o) { return containsKey(o); }
    public final boolean remove(Object key) {
        return removeNode(hash(key), key, null, false, true) != null;
    }
    public final Spliterator<K> spliterator()  {
        return Spliterators.spliterator(this, Spliterator.SIZED |
                                        Spliterator.ORDERED |
                                        Spliterator.DISTINCT);
    }
    public Object[] toArray() { return keysToArray(new Object[size]); }
    public <T> T[] toArray(T[] a) { return keysToArray(prepareArray(a)); }
    
}


final class LinkedKeyIterator extends LinkedHashIterator
    implements Iterator<K> {

    // key
    public final K next() { return nextNode().getKey(); }
}
```

values

```java
public Collection<V> values() {
    // 获得 values 缓存
    Collection<V> vs = values;
    // 如果不存在，则进行创建
    if (vs == null) {
        vs = new LinkedValues(); // LinkedValues 是 LinkedHashMap 自定义的
        values = vs;
    }
    return vs;
}

final class LinkedValues extends AbstractCollection<V> {
    public final Iterator<V> iterator() {
        return new LinkedValueIterator(); // <X>
    }
    
    // 同 LinkedKeySet ，其余方法略
}

final class LinkedValueIterator extends LinkedHashIterator
    implements Iterator<V> {

    // value
    public final V next() { return nextNode().value; }
}
```

entrySet

```java
public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    // LinkedEntrySet 是 LinkedHashMap 自定义的
    return (es = entrySet) == null ? (entrySet = new LinkedEntrySet()) : es;
}

final class LinkedEntrySet extends AbstractSet<Map.Entry<K,V>> {
    public final Iterator<Map.Entry<K,V>> iterator() {
        return new LinkedEntryIterator(); // <X>
    }
    
    // 同 LinkedKeySet ，其余方法略
}


final class LinkedEntryIterator extends LinkedHashIterator
    implements Iterator<Map.Entry<K,V>> {

    // Entry
    public final Map.Entry<K,V> next() { return nextNode(); }
}
```



- LinkedHashIterator 

```java
abstract class LinkedHashIterator {

    // 下一个节点
    LinkedHashMap.Entry<K,V> next;
    // 当前节点
    LinkedHashMap.Entry<K,V> current;
    // 修改次数
    int expectedModCount;

    LinkedHashIterator() {
        next = head;
        expectedModCount = modCount;
        current = null;
    }

    public final boolean hasNext() {
        return next != null;
    }

    final LinkedHashMap.Entry<K,V> nextNode() {
        LinkedHashMap.Entry<K,V> e = next;
        // 如果发生了修改，抛出 ConcurrentModificationException 异常
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        // 如果 e 为空，说明没有下一个节点，则抛出 NoSuchElementException 异常
        if (e == null)
            throw new NoSuchElementException();
        // 遍历到下一个节点
        current = e;
        // 通过链表的方式遍历
        next = e.after;
        return e;
    }

    public final void remove() {
        // 移除当前节点
        Node<K,V> p = current;
        if (p == null)
            throw new IllegalStateException();
        // 如果发生了修改，抛出 ConcurrentModificationException 异常
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        // 标记 current 为空，因为被移除了
        current = null;
        // 移除节点
        removeNode(p.hash, p.key, null, false, false);
        // 修改 expectedModCount 次数
        expectedModCount = modCount;
    }

}
```





##### 小结

- LinkedHashMap 是 HashMap 的子类，增加了顺序访问的特性。
  - 【默认】当 `accessOrder = false` 时，按照 key-value 的**插入**顺序进行访问。
  - 当 `accessOrder = true` 时，按照 key-value 的**插入**和**读取**顺序进行访问。
- LinkedHashMap 的**顺序**特性，通过内部的双向链表实现，所以我们把它看成是 LinkedList + LinkedHashMap 的组合。
- LinkedHashMap 通过重写 HashMap 提供的回调方法，从而实现其对**顺序**的特性的处理。同时，因为 LinkedHashMap 的**顺序**特性，需要重写 `#keysToArray(T[] a)` 等**遍历**相关的方法。
- LinkedHashMap 可以方便实现 LRU 算法的缓存

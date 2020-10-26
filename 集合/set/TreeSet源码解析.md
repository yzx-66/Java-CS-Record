#### TreeSet

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200929171453582.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


- NavigableSet 定义了一些有方向的操作

	![在这里插入图片描述](https://img-blog.csdnimg.cn/2020092917151295.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



##### 数据结构

```java
private transient NavigableMap<E,Object> m;

// Dummy value to associate with an Object in the backing Map
private static final Object PRESENT = new Object();
```



##### 构造方法

```java
TreeSet(NavigableMap<E,Object> m) {
    this.m = m;
}

public TreeSet() {
    this(new TreeMap<>());
}

public TreeSet(Comparator<? super E> comparator) {
    this(new TreeMap<>(comparator));
}

public TreeSet(Collection<? extends E> c) {
    this();
    // 批量添加
    addAll(c);
}

public TreeSet(SortedSet<E> s) {
    this(s.comparator());
    // 批量添加
    addAll(s);
}

```



##### 添加元素

- 单个元素

```java
public boolean add(E e) {
    return m.put(e, PRESENT)==null;
}
```

- 多个元素

```java
public  boolean addAll(Collection<? extends E> c) {
    // Use linear-time version if applicable
    // 情况一
    if (m.size()==0 && c.size() > 0 &&
        c instanceof SortedSet &&
        m instanceof TreeMap) {
        SortedSet<? extends E> set = (SortedSet<? extends E>) c;
        TreeMap<E,Object> map = (TreeMap<E, Object>) m;
        if (Objects.equals(set.comparator(), map.comparator())) {
            map.addAllForTreeSet(set, PRESENT);
            return true;
        }
    }
    // 情况二
    return super.addAll(c);
}
```



##### 删除元素

- 单个元素

```java
public boolean remove(Object o) {
    return m.remove(o)==PRESENT;
}
```



##### 查找元素

- 是否包含

```java
public boolean contains(Object o) {
    return m.containsKey(o);
}
```

- 接近的元素

```java
public E lower(E e) {
    return m.lowerKey(e);
}

public E floor(E e) {
    return m.floorKey(e);
}

public E ceiling(E e) {
    return m.ceilingKey(e);
}

public E higher(E e) {
    return m.higherKey(e);
}

```

- 首尾的元素

```java
public E first() {
    return m.firstKey();
}

public E last() {
    return m.lastKey();
}

/**
 * 拓展方法
 */
public E pollFirst() {
    Map.Entry<E,?> e = m.pollFirstEntry();
    return (e == null) ? null : e.getKey();
}

public E pollLast() {
    Map.Entry<E,?> e = m.pollLastEntry();
    return (e == null) ? null : e.getKey();
}
```



##### 清空

```java
public void clear() {
    m.clear();
}
```



##### 迭代器

```java
public Iterator<E> iterator() { // 正序 Iterator 迭代器
    return m.navigableKeySet().iterator();
}

public Iterator<E> descendingIterator() { // 倒序 Iterator 迭代器
    return m.descendingKeySet().iterator();
}
```



##### 序列化

- 序列化

```java
@java.io.Serial
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException {
    // Write out any hidden stuff
    // 写入非静态属性、非 transient 属性
    s.defaultWriteObject();

    // Write out Comparator
    // 写入比较器
    s.writeObject(m.comparator());

    // Write out size
    // 写入 key-value 键值对数量
    s.writeInt(m.size());

    // Write out all elements in the proper order.
    // 写入具体的 key-value 键值对
    for (E e : m.keySet())
        s.writeObject(e);
}
```

- 反序列化

```java
@java.io.Serial
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    // Read in any hidden stuff
    // 读取非静态属性、非 transient 属性
    s.defaultReadObject();

    // Read in Comparator
    // 读取比较器
    @SuppressWarnings("unchecked")
    Comparator<? super E> c = (Comparator<? super E>) s.readObject();

    // Create backing TreeMap
    // 创建 TreeMap 对象
    TreeMap<E,Object> tm = new TreeMap<>(c);
    m = tm;

    // Read in size
    // 读取 key-value 键值对数量
    int size = s.readInt();

    // 读取具体的 key-value 键值对
    tm.readTreeSet(size, s, PRESENT);
}
```

```java
// TreeMap.java
void readTreeSet(int size, java.io.ObjectInputStream s, V defaultVal)
    throws java.io.IOException, ClassNotFoundException {
    buildFromSorted(size, null, s, defaultVal);
}
```



##### 查找范围

（对 NavigableSet 的实现）

```java
// subSet 组
public NavigableSet<E> subSet(E fromElement, boolean fromInclusive,
                              E toElement,   boolean toInclusive) {
    return new TreeSet<>(m.subMap(fromElement, fromInclusive,
                                   toElement,   toInclusive));
}
public SortedSet<E> subSet(E fromElement, E toElement) {
    return subSet(fromElement, true, toElement, false);
}

// headSet 组
public NavigableSet<E> headSet(E toElement, boolean inclusive) {
    return new TreeSet<>(m.headMap(toElement, inclusive));
}
public SortedSet<E> headSet(E toElement) {
    return headSet(toElement, false);
}

// tailSet 组
public NavigableSet<E> tailSet(E fromElement, boolean inclusive) {
    return new TreeSet<>(m.tailMap(fromElement, inclusive));
}

public SortedSet<E> tailSet(E fromElement) {
    return tailSet(fromElement, true);
}
```



##### 小结

TreeSet 是基于 TreeMap 的 Set 实现类

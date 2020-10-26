#### HashSet

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200929171413541.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


- HashSet ，基于 HashMap 的 Set 实现类。



##### 数据结构

```java
// HashSet 只有一个属性，那就是 map
private transient HashMap<E, Object> map;

// 因为 HashSet 没有 value 的需要，所以使用一个统一的 PRESENT 即可
private static final Object PRESENT = new Object();
```



##### 构造方法

```java
public HashSet() {
    map = new HashMap<>();
}

public HashSet(Collection<? extends E> c) {
    // 最小必须是 16 。
    // (int) (c.size()/.75f) + 1 避免扩容
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    // 批量添加
    addAll(c);
}

public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}

public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}

HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    // 注意，这种情况下的构造方法，创建的是 LinkedHashMap 对象
    map = new LinkedHashMap<>(initialCapacity, loadFactor); 
}
```



##### 添加元素

- 添加一个元素

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

- 添加多个元素

```java
public boolean addAll(Collection<? extends E> c) {
    boolean modified = false;
    // 遍历 c 集合，逐个添加
    for (E e : c)
        if (add(e))
            modified = true;
    return modified;
}
```



##### 删除元素

- 删除单个元素

```java
public boolean remove(Object o) {
    return map.remove(o) == PRESENT;
}
```



##### 查找元素

- 查找单个元素

```java
public boolean contains(Object o) {
    return map.containsKey(o);
}
```

**清空**

```java
public void clear() {
    map.clear();
}

```



##### 迭代器

```java
public Iterator<E> iterator() {
        return map.keySet().iterator();
}
```





##### 序列化

- 序列化

```java
@java.io.Serial
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException {
    // 写入非静态属性、非 transient 属性
    s.defaultWriteObject();

    // 写入 map table 数组大小
    s.writeInt(map.capacity());
    // 写入 map 加载因子
    s.writeFloat(map.loadFactor());

    // Write out size
    // 写入 map 大小
    s.writeInt(map.size());

    // 遍历 map ，逐个 key 序列化
    for (E e : map.keySet())
        s.writeObject(e);
}
```



- 反序列化

```java
@java.io.Serial
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    // 读取非静态属性、非 transient 属性
    s.defaultReadObject();

    // 读取 HashMap table 数组大小
    int capacity = s.readInt();
    // 校验 capacity 参数
    if (capacity < 0) {
        throw new InvalidObjectException("Illegal capacity: " + capacity);
    }

    // 获得加载因子 loadFactor
    float loadFactor = s.readFloat();
    // 校验 loadFactor 参数
    if (loadFactor <= 0 || Float.isNaN(loadFactor)) {
        throw new InvalidObjectException("Illegal load factor: " +
                                         loadFactor);
    }

    // 读取 key-value 键值对数量 size
    int size = s.readInt();
    // 校验 size 参数
    if (size < 0) {
        throw new InvalidObjectException("Illegal size: " +
                                         size);
    }

    /**
     * 计算容量
     */
    capacity = (int) Math.min(size * Math.min(1 / loadFactor, 4.0f),
            HashMap.MAXIMUM_CAPACITY);


    SharedSecrets.getJavaObjectInputStreamAccess()
                 .checkArray(s, Map.Entry[].class, HashMap.tableSizeFor(capacity)); 

    // 创建 LinkedHashMap 或 HashMap 对象
    map = (((HashSet<?>)this) instanceof LinkedHashSet ?
           new LinkedHashMap<>(capacity, loadFactor) :
           new HashMap<>(capacity, loadFactor));

    // 遍历读取 key 键，添加到 map 中
    for (int i=0; i<size; i++) {
        @SuppressWarnings("unchecked")
            E e = (E) s.readObject();
        map.put(e, PRESENT);
    }
}
```



##### 小结

- HashSet 是基于 HashMap 的 Set 实现类

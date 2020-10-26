#### ArrrayList

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200929161501617.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


- java.util.RandomAccess 接口，表示 ArrayList 支持**快速**的随机访问

##### 数据结构

```java
transient Object[] elementData;

// 实际存储数据的多少
private int size;
```



##### 构造方法

- 指定大小

```java
// 初始化长度为0时，给elementData赋值的空数组
private static final Object[] EMPTY_ELEMENTDATA = {};

public ArrayList(int initialCapacity) {
    // 初始化容量大于 0 时，创建 Object 数组
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    // 初始化容量等于 0 时，使用 EMPTY_ELEMENTDATA 对象
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    // 初始化容量小于 0 时，抛出 IllegalArgumentException 异常
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```



- 不指定大小

```java
// 不指定大小时，默认第一次扩容的大小
private static final int DEFAULT_CAPACITY = 10;


// 未指定参数时，给elementData赋值的空数组
// 这里要和 EMPTY_ELEMENTDATA 区分开，因为初始化长度为0和未指定长度，在扩容时策略不同
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

```



##### 添加元素

核心为调用System.arraycopy()

- 不指定索引

```java
@Override
public boolean add(E e) {
    // <1> 增加数组修改次数
    modCount++;
    // 添加元素
    add(e, elementData, size);
    // 返回添加成功
    return true;
}

private void add(E e, Object[] elementData, int s) {
    // <2> 如果容量不够，进行扩容
    if (s == elementData.length)
        elementData = grow();
    // <3> 设置到末尾
    elementData[s] = e;
    // <4> 数量大小加一
    size = s + 1;
}
```

- 指定索引

```java
public void add(int index, E element) {
    // 校验位置是否在数组范围内
    rangeCheckForAdd(index);
    // 增加数组修改次数
    modCount++;
    // 如果数组大小不够，进行扩容
    final int s;
    Object[] elementData;
    if ((s = size) == (elementData = this.elementData).length)
        elementData = grow();
    // 将 index + 1 位置开始的元素，进行往后挪
    System.arraycopy(elementData, index,
                     elementData, index + 1,
                     s - index);
    // 设置到指定位置
    elementData[index] = element;
    // 数组大小加一
    size = s + 1;
}

private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```



- 添加多个元素

```java
public boolean addAll(Collection<? extends E> c) {
    // 转成 a 数组
    Object[] a = c.toArray();
    // 增加修改次数
    modCount++;
    // 如果 a 数组大小为 0 ，返回 ArrayList 数组无变化
    int numNew = a.length;
    if (numNew == 0)
        return false;
    // 如果 elementData 剩余的空间不够，则进行扩容。要求扩容的大小，至于能够装下 a 数组。
    Object[] elementData;
    final int s;
    if (numNew > (elementData = this.elementData).length - (s = size))
        elementData = grow(s + numNew);
    // 将 a 复制到 elementData 从 s 开始位置
    System.arraycopy(a, 0, elementData, s, numNew);
    // 数组大小加 numNew
    size = s + numNew;
    return true;
}
```





##### 数组扩容

```java
private Object[] grow() {
    return grow(size + 1);
}

private Object[] grow(int minCapacity) {
    int oldCapacity = elementData.length;
    /**
     * 如果原容量大于 0 ，或者数组不是 DEFAULTCAPACITY_EMPTY_ELEMENTDATA (即初始化未指定大小）时，计算新的数组大小，并创建扩容
     */
    if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        /**
         * 在 minCapacity - oldCapacity 和 oldCapacity >> 1（即oldCapacity的一半）中取一个大的，然后加到oldCapacity上
         */
        int newCapacity = ArraysSupport.newLength(oldCapacity,
                minCapacity - oldCapacity, /* minimum growth */
                oldCapacity >> 1           /* preferred growth */);
        // 将elementData改变到指定大小
        return elementData = Arrays.copyOf(elementData, newCapacity);
    /**
     * 如果是 DEFAULTCAPACITY_EMPTY_ELEMENTDATA 数组（即初始化未指定大小），直接创建新的数组即可。
     */
    } else {
        return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
    }
}
```



##### 删除元素

- 指定索引

```java
public E remove(int index) {
    // 校验 index 不要超过 size
    Objects.checkIndex(index, size);
    final Object[] es = elementData;

    // 记录该位置的原值
    @SuppressWarnings("unchecked") E oldValue = (E) es[index];
    // 快速移除
    fastRemove(es, index);

    // 返回该位置的原值
    return oldValue;
}

private void fastRemove(Object[] es, int i) {
    // 增加数组修改次数
    modCount++;
    // 如果 i 不是移除最末尾的元素，则将 i + 1 位置的数组往前挪
    final int newSize;
    if ((newSize = size - 1) > i) // -1 的原因是，size 是从 1 开始，而数组下标是从 0 开始。
        System.arraycopy(es, i + 1, es, i, newSize - i);
    // 将新的末尾置为 null ，帮助 GC
    es[size = newSize] = null;
}
```



- 指定元素

```java
public boolean remove(Object o) {
    final Object[] es = elementData;
    final int size = this.size;
    // 寻找首个为 o 的位置
    int i = 0;
    found: {
        if (o == null) { // o 为 null 的情况
            for (; i < size; i++)
                if (es[i] == null)
                    break found;
        } else { // o 非 null 的情况
            for (; i < size; i++)
                if (o.equals(es[i]))
                    break found;
        }
        // 如果没找到，返回 false
        return false;
    }
    // 快速移除
    fastRemove(es, i);
    // 找到了，返回 true
    return true;
}
```



- 删除指定索引范围

```java
protected void removeRange(int fromIndex, int toIndex) {
    // 范围不正确，抛出 IndexOutOfBoundsException 异常
    if (fromIndex > toIndex) {
        throw new IndexOutOfBoundsException(
                outOfBoundsMsg(fromIndex, toIndex));
    }
    // 增加数组修改次数
    modCount++;
    // <X> 移除 [fromIndex, toIndex) 的多个元素
    shiftTailOverGap(elementData, fromIndex, toIndex);
}

private void shiftTailOverGap(Object[] es, int lo, int hi) {
    // 将 es 从 hi 位置开始的元素，移到 lo 位置开始。
    System.arraycopy(es, hi, es, lo, size - hi);
    // 将从 [size - hi + lo, size) 的元素置空，因为已经被挪到前面了。
    for (int to = size, i = (size -= hi - lo); i < to; i++)
        es[i] = null;
}

private static String outOfBoundsMsg(int fromIndex, int toIndex) {
    return "From Index: " + fromIndex + " > To Index: " + toIndex;
}
```



- 删除多个元素

```java
public boolean removeAll(Collection<?> c) {
    return batchRemove(c, false, 0, size);
}

boolean batchRemove(Collection<?> c, boolean complement, final int from, final int end) {
    // 校验 c 非 null 。
    Objects.requireNonNull(c);
    final Object[] es = elementData;
    int r;
    
    // 优化，顺序遍历 elementData 数组，找到第一个不符合 complement ，然后结束遍历。
    for (r = from;; r++) {
        // 遍历到尾，都没不符合条件的，直接返回 false 。
        if (r == end)
            return false;
        //  如果包含结果不符合 complement 时，结束
        if (c.contains(es[r]) != complement)
            break;
    }
    // 设置开始写入 w 为 r ，注意不是 r++ 。
    // r++ 后，用于读取下一个位置的元素。因为通过上的优化循环，我们已经 es[r] 是不符合条件的。
    int w = r++;
    try {
        // 继续遍历 elementData 数组，如何符合条件，则进行移除
        for (Object e; r < end; r++)
            if (c.contains(e = es[r]) == complement) // 判断符合条件
                // 移除的方式，通过将当前值 e 写入到 w 位置，然后 w 跳到下一个位置。
                es[w++] = e; 
    } catch (Throwable ex) {
        // 如果 contains 方法发生异常，则将 es 从 r 位置的数据写入到 es 从 w 开始的位置
        System.arraycopy(es, r, es, w, end - r);
        w += end - r;
        // 继续抛出异常
        throw ex;
    } finally {
        // 增加数组修改次数
        modCount += end - w;
        // 将数组 [w, end) 位置赋值为 null 。
        shiftTailOverGap(es, w, end);
    }
    return true;
}
```



##### 清空

```java
public void clear() {
    // 获得当前的数组修改次数
    modCount++;
    // 遍历数组，倒序设置为 null
    final Object[] es = elementData;
    for (int to = size, i = size = 0; i < to; i++)
        es[i] = null;
}
```



##### 迭代器

- 获取迭代器

```java
public Iterator<E> iterator() {
    return new Itr();
}
```

Itr 一共有 3 个属性，如下：

```java
// 下一个访问元素的位置，从下标 0 开始。
int cursor;   

/**
 * 上一次访问元素的位置。
 *
 * 1. 初始化为 -1 ，表示无上一个访问的元素
 * 2. 遍历到下一个元素时，lastRet 会指向当前元素，而 cursor 会指向下一个元素。这样，如果我们要实现 remove 方法，移除当前元素，就可以实现了。
 * 3. 移除元素时，设置为 -1 ，表示最后访问的元素不存在了，都被移除咧。
 */
int lastRet = -1; // index of last element returned; -1 if no such

/**
 * 创建迭代器时，数组修改次数。
 *
 * 在迭代过程中，如果数组发生了变化，会抛出 ConcurrentModificationException 异常。
 */
int expectedModCount = modCount;

// prevent creating a synthetic constructor
Itr() {}
```



- 判断是否还有下一个元素

```java
public boolean hasNext() {
    return cursor != size;
}
```



- 获取下一个元素

```java
public E next() {
    // 校验是否数组发生了变化
    checkForComodification();
    // 判断如果超过 size 范围，抛出 NoSuchElementException 异常
    // <1> i 记录当前 cursor 的位置
    int i = cursor; 
    if (i >= size)
        throw new NoSuchElementException();
    // 判断如果超过 elementData 大小，说明可能被修改了，抛出 ConcurrentModificationException 异常
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    // <2> cursor 指向下一个位置
    cursor = i + 1;
    // <3> 返回当前位置的元素
    return (E) elementData[lastRet = i]; // <4> 此处，会将 lastRet 指向当前位置
}

final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```



- 移除当前元素

```java
public void remove() {
    // 如果 lastRet 小于 0 ，说明没有指向任何元素，抛出 IllegalStateException 异常
    if (lastRet < 0)
        throw new IllegalStateException();
    // 校验是否数组发生了变化
    checkForComodification();

    try {
        // 移除 lastRet 位置的元素
        ArrayList.this.remove(lastRet);
        // cursor 指向 lastRet 位置，因为被移了，所以需要后退下
        cursor = lastRet;
        // lastRet 标记为 -1 ，因为当前元素被移除了
        lastRet = -1;
        // 记录新的数组的修改次数
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```



- 消费剩余未迭代的元素

```java
@Override
public void forEachRemaining(Consumer<? super E> action) {
    // 要求 action 非空
    Objects.requireNonNull(action);
    // 获得当前数组大小
    final int size = ArrayList.this.size;
    // 记录 i 指向 cursor
    int i = cursor;
    if (i < size) {
        // 判断如果超过 elementData 大小，说明可能被修改了，抛出 ConcurrentModificationException 异常
        final Object[] es = elementData;
        if (i >= es.length)
            throw new ConcurrentModificationException();
        // 逐个处理
        for (; i < size && modCount == expectedModCount; i++)
            action.accept(elementAt(es, i));
        // update once at end to reduce heap write traffic
        // 更新 cursor 和 lastRet 的指向
        cursor = i;
        lastRet = i - 1;
        // 校验是否数组发生了变化
        checkForComodification();
    }
}
```



##### 序列化

- 序列化

```java
@java.io.Serial
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException {
    // 获得当前的数组修改次数
    int expectedModCount = modCount;

    // <1> 写入非静态属性、非 transient 属性
    s.defaultWriteObject();

    // <2> 写入 size ，主要为了与 clone 方法的兼容
    s.writeInt(size);

    // <3> 逐个写入 elementData 数组的元素
    for (int i = 0; i < size; i++) {
        s.writeObject(elementData[i]);
    }

    // 如果 other 修改次数发生改变，则抛出 ConcurrentModificationException 异常
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}
```



- 反序列化

```java
@java.io.Serial
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {

    // 读取非静态属性、非 transient 属性
    s.defaultReadObject();

    // 读取 size ，不过忽略不用
    s.readInt(); // ignored

    if (size > 0) {
        // like clone(), allocate array based upon size not capacity
        SharedSecrets.getJavaObjectInputStreamAccess().checkArray(s, Object[].class, size); 
        // 创建 elements 数组
        Object[] elements = new Object[size];
 
        // 逐个读取
        for (int i = 0; i < size; i++) {
            elements[i] = s.readObject();
        }

        // 赋值给 elementData
        elementData = elements;
    } else if (size == 0) {
        // 如果 size 是 0 ，则直接使用空数组
        elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new java.io.InvalidObjectException("Invalid size: " + size);
    }
}
```



##### 时间复杂度

- 查询
  - ArrayList 随机访问时间复杂度是 O(1) ，查找指定元素的**平均**时间复杂度是 O(n) 。
- 删除
  - ArrayList 移除指定位置的元素的最好时间复杂度是 O(1) ，最坏时间复杂度是 O(n) ，平均时间复杂度是 O(n) 。
    - 最好时间复杂度发生在末尾移除的情况。
  - ArrayList 移除指定元素的时间复杂度是 O(n) 。
    - 因为首先需要进行查询，然后在使用移除指定位置的元素，无论怎样，都需要 O(n) 的时间复杂度。
- 添加
  - ArrayList 添加元素的最好时间复杂度是 O(1) ，最坏时间复杂度是 O(n) ，平均时间复杂度是 O(n) 。
    - 最好时间复杂度发生在末尾添加的情况。

#### CopyOnWriteArrayList

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015094549614.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


* CopyOnWriteList 实现的接口和 ArrayList 完全相同，所以 ArrayList 提供的 api ，CopyOnWriteArrayList 也提供

* 其实说白了就是每次要增、删、改的时候，会创建一个新的 array，并把旧的 array 全部复制过来，操作完后再让 array 指向这个新创建的 array，所以也不存在什么扩容问题，因为每次 add 都要扩容

##### 属性

```java
/** The lock protecting all mutators */
final transient ReentrantLock lock = new ReentrantLock();

/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array;


final void setArray(Object[] a) {
    array = a;
}

final Object[] getArray() {
    return array;
}
```



##### add

* 不指定索引

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    //上锁，同一个 CopyOnWriteList 对象的 增、删、改都使用的同一把锁
    lock.lock(); 
    try {
        // 获得当前数组对象
        Object[] elements = getArray(); 
        int len = elements.length;
        // 拷贝到一个新的数组中
        Object[] newElements = Arrays.copyOf(elements, len + 1);
		// 给新数组插入数据元素
        newElements[len] = e;
        // 指向新的数组对象
        setArray(newElements);
        return true;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```

* 指定索引

```java
public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+
                                                ", Size: "+len);
        Object[] newElements;
        // 判断一下是否在数组中间插入的
        int numMoved = len - index;
        if (numMoved == 0)
            // 不是的话，那直接把全部元素拷贝到一个 size 增加 1 的新数组上
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            // 否则，自己 new 一个 size + 1 的数组，然后执行分段拷贝
            newElements = new Object[len + 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index, newElements, index + 1,
                             numMoved);
        }
        // 给新数组添加元素
        newElements[index] = element;
        // 指向新数组
        setArray(newElements);
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```



##### remove

```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        // index 之后的元素个数
        int numMoved = len - index - 1;
        if (numMoved == 0)
            // 是最后一个，那直接新数组长度减小 1 拷贝过去就行了
            // 然后指向新数组
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            // 否则 new 一个长度减小 1 的新数组，然后分段拷贝
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            // 指向新数组
            setArray(newElements);
        }
        return oldValue;
    } finally {
        // 释放锁
        lock.unlock();
    }
 }
```



##### get

```java
public E get(int index) {
    return get(getArray(), index);
}

private E get(Object[] a, int index) {
   return (E) a[index];
}
```



##### 迭代器

```java
public Iterator<E> iterator() {
    return new COWIterator<E>(getArray(), 0);
}

static final class COWIterator<E> implements ListIterator<E> {
    
    // 获取迭代器时 array 的快照
    // 所以如果在迭代过程中，进行了增删，是不会影响迭代的，因为迭代的是 snapshot 快照（也就是修改后要丢弃的老数组）
    private final Object[] snapshot;
    private int cursor;

    private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements; // 快照
    }

    public boolean hasNext() { return cursor < snapshot.length; }
    public boolean hasPrevious() { return cursor > 0; }

    public E next() {
        if (! hasNext())  throw new NoSuchElementException();
        // 从 snapshot 取值
        return (E) snapshot[cursor++];
    }

    public E previous() {
        if (! hasPrevious())  throw new NoSuchElementException();
        // 从 snapshot 取值
        return (E) snapshot[--cursor];
    }
    
    // 其余略。
}
```

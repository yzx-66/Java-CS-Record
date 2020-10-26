### ThreadLocal

* 每个 Thread 都有一个自己的  ThreadLocalMap，其 Key 为 ThreadLocal 对象（在每个线程的 ThreadLocalMap 中只起到一个索引的作用），Value 为 ThreadLocal 的类型参数对象在自己线程的副本
  * 比如 ThreadLocal\<Integer>  i，那么 key 就是 i 作为索引，value 就是自己线程的副本 Integer

* ThreadLocalMap 解决冲突使用的是 线性探测法，不是链地址法
* ThreadLocal 不是用于解决共享变量的问题的，每个线程都有自己的改变量副本，怎么会多个线程一起使用一个变量呢。



##### ThreadLocalMap

###### Entry 

```java
// 节点
static class Entry extends WeakReference<ThreadLocal<?>> {
     /** The value associated with this ThreadLocal. */
     Object value;

    // key 就是 ThreadLocal 对象
     Entry(ThreadLocal<?> k, Object v) {
         // ThreadLocalMap 的 key 在每次垃圾回收时会被回收
         super(k);
         value = v;
     }
 }
```

###### set

```java
private void set(ThreadLocal<?> key, Object value) {

    ThreadLocal.ThreadLocalMap.Entry[] tab = table;
    int len = tab.length;

    // 根据 ThreadLocal 的散列值，查找对应元素在数组中的位置
    // 后面实现中说
    int i = key.threadLocalHashCode & (len-1);

    // 采用“线性探测法”，寻找合适位置
    for (ThreadLocal.ThreadLocalMap.Entry e = tab[i];
        e != null;
        e = tab[i = nextIndex(i, len)]) {

        ThreadLocal<?> k = e.get();

        // key 存在，直接覆盖
        if (k == key) {
            e.value = value;
            return;
        }

        // key == null，但是存在值（因为此处的e != null），说明之前的ThreadLocal对象已经被回收了
        if (k == null) {
            // 用新元素替换陈旧的元素返回即可，防止内存泄漏
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    // ThreadLocal对应的key实例不存在也没有陈旧元素，new 一个
    tab[i] = new ThreadLocal.ThreadLocalMap.Entry(key, value);

    int sz = ++size;

    // cleanSomeSlots 清楚陈旧的Entry（key == null）防止内存泄漏
    // 如果没有清理陈旧的 Entry 并且数组中的元素大于了阈值，则进行 rehash
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        // 扩大两倍，然后挨个重新计算 hash 放到新 Map
        rehash();
}

// 这个槽位冲突，那么找下一个槽位的索引
private static int nextIndex(int i, int len) {
    // 可见是循环的
    return ((i + 1 < len) ? i + 1 : 0);
}
```

###### getEntry

```java
// 没有 get 方法，只有 getEntry 方法
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}


private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            // 删除这个元素，防止内存泄漏
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```



##### 实现

###### hash 值

```java
// ThreadLocal 对象的 hashcode 在其构造的时候初始化
private final int threadLocalHashCode = nextHashCode();

// 静态变量，用于累加
private static AtomicInteger nextHashCode = new AtomicInteger();

// 累加值
private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
    // 每个 ThreadLocal 对象的 hashcode 是累加的，累加 0x61c88647
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```



###### get

```java
public T get() {
    // 获取当前线程
    Thread t = Thread.currentThread();

    // 获取当前线程对应的 threadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 从当前线程的ThreadLocalMap获取相对应的Entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")

            // 获取目标值
            T result = (T)e.value;
            return result;
        }
    }
    // 设置副本初始值到当前线程的 ThreadLocalMap 中
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

private T setInitialValue() {
    // 一般要重写这个 initialValue()，其默认返回的 null
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // 把副本初始值作为 value 设置到进去
        map.set(this, value);
    else
        createMap(t, value);
   return value;
}

// 每个线程保存的副本的初始值，一般要重写
protected T initialValue() {
    return null;
}
```



###### set

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        // 每个线程的 ThreadLocalMap 是懒加载的
        createMap(t, value);
}


void createMap(Thread t, T firstValue) {
    // 用放入一个键值对的构造方法
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```



###### remove

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```



##### 内存泄漏

原因：

* 每个Thread都有一个 ThreadLocal.ThreadLocalMap 的 map，该map的 key 为 ThreadLocal 实例，它为一个弱引用，我们知道弱引用有利于GC回收。当ThreadLocal的key == null时，GC就会回收这部分空间，但是value却不一定能够被回收，因为他还与Current Thread存在一个强引用关系，被 ThreadLocalHashMap 引用。

  ![](https://img-blog.csdnimg.cn/2020101509411090.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




解决：

* 隐式调用
  * ThreadLocalMap#set，当 hash 冲突，在往后逐个寻址的时候如果遇到 key == null ，会替换这个 entry
  * ThreadLocalMap#set，当 添加完毕，如果 size 大于阈值在扩容前，会先删除全部 key == null 的 entry
  * ThreadLocalMap#get，当 hash 冲突，在往后逐个寻找 key 相等的时候，如果遇到 key == null 的，会删除这个 entry
* 显示调用
  * ThreadLocal#remove

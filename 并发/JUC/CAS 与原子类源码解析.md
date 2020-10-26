### CAS

#### UnSafe

##### API

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201014234733212.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


 ```java
  /**
   * 举例：
   *  var 1： 被 cas 对象
   *  var 2： 要被 cas 的属性，在 var 1 的字段偏移，该入参是通过 Unsafe#objectFieldOffset 获得
   *  var 3： 预期值，该入参通过 Unsafe#get*Volatile 获得
   *  var 5： 新值
   */
  public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5)
 ```

  ```java
  /**
   * 举例：
   *  var 1： 被 cas 对象
   *  var 2： 要被 cas 的属性，在 var 1 的字段偏移，该入参是通过 Unsafe#objectFieldOffset 获得
   *  var 4： 增加的值
   */
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            // 获得字段的最新值，然后记录下来
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4)); // 最终还是 自旋 + cas

        // 返回自旋 cas 成功时记录的预期值
        return var5;
    }
  ```

```java
  /**
   * 举例：
   *  var 1： 被 cas 对象
   *  var 2： 要被 cas 的属性，在 var 1 的字段偏移，该入参是通过 Unsafe#objectFieldOffset 获得
   *  var 4： 新值
   */
	public final int getAndSetInt(Object var1, long var2, int var4) {
        int var5;
        do {
            // 获得字段的最新值，然后记录下来
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var4));  // 最终还是 自旋 + cas

        // 返回自旋 cas 成功时记录的预期值
        return var5;
    }
```

##### CPU 原子操作

CAS 可以保证一次的**读-改-写**操作是原子操作，在单处理器上该操作容易实现，但是在多处理器上实现就有点儿复杂了。CPU 提供了**两种方法**来实现多处理器的原子操作：总线加锁 或者 缓存加锁。

- **总线加锁**：总线加锁就是就是使用处理器提供的一个 LOCK# 信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住，那么该处理器可以独占使用共享内存。但是这种处理方式**把CPU和内存之间的通信锁住**了，在锁定期间，其他处理器都不能其他内存地址的数据，开销大。
- **缓存加锁**：利用缓存一致性协议来保证原子性（MESI )。缓存一致性机制可以保证同一个内存区域的数据仅能被一个处理器修改，也就是说当 CPU1 修改缓存行中的 `i` 时使用缓存锁定，那么 CPU2 就不能同时缓存了 `i` 的缓存行。

#### Atomic*

使用 AtomicInteger 举例

* 初始化

```java
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

static {
    try {
        // 获得要 value 属性的偏移
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}

// 使用 volatile 修适变量，那么 get/set 就可以直接使用，不然还要调用 native 方法保证最新
private volatile int value;
```

* 部分方法

```java
public final int getAndAdd(int delta) {
    // 直接返回 cas 成功时的预期值
    return unsafe.getAndAddInt(this, valueOffset, delta);
}

public final int addAndGet(int delta) {
    // 返回 cas 成功的预期值 + 改变量（可正可负）
    return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
}

public final int get() {
    // volatile 修饰的 value，直接赋值即可
    return value;
}

public final void set(int newValue) {
    // volatile 修饰的 value，直接读取即可
    value = newValue;
}
```



####  CAS 缺陷

##### 循环时间太长

如果CAS一直不成功呢？这种情况绝对有可能发生，如果自旋 CAS 长时间地不成功，则会给 CPU 带来非常大的开销。

解决：

* 在 J.U.C 中，有些地方就限制了 CAS 自旋的次数，例如： BlockingQueue 的 SynchronousQueue 。



##### 只保证一个变量原子操作

看了 CAS 的实现就知道这只能针对一个共享变量

解决：

* 如果是多个共享变量就只能使用锁了
* 当然如果你有办法把多个变量整成一个变量，利用 CAS 也不错。例如读写锁中 `state` 的高低位。



##### ABA问题

CAS 需要检查操作值有没有发生改变，如果没有发生改变则更新。但是存在这样一种情况：如果一个值原来是 A，变成了 B，然后又变成了 A，那么在 CAS 检查的时候会发现没有改变，但是实质上它已经发生了改变，这就是所谓的ABA问题。



解决：

* 加上版本号，即在每个变量都加上一个版本号，每次改变时加 1 ，即 `A —> B —> A` ，变成`1A —> 2B —> 3A` 。



实现：AtomicStampedReference

* 包装 `[E,Integer]` 的元组，来对对象**标记**版本戳 `stamp` ，从而避免 ABA 问题。

```java
private static class Pair<T> {
    final T reference; // 对象引用
    final int stamp; // 版本戳
    private Pair(T reference, int stamp) {
        this.reference = reference;
        this.stamp = stamp;
    }
    static <T> Pair<T> of(T reference, int stamp) {
        return new Pair<T>(reference, stamp);
    }
}

private volatile Pair<V> pair;

// 赋值时会包装成 pair
public void set(V newReference, int newStamp) {
    Pair<V> current = pair;
    if (newReference != current.reference || newStamp != current.stamp)
        this.pair = Pair.of(newReference, newStamp);
}
```

* CAS 还要加 版本戳

```java
public boolean compareAndSet(V   expectedReference,
                             V   newReference,
                             int expectedStamp,
                             int newStamp) {
    Pair<V> current = pair;
    return
        expectedReference == current.reference &&  // 预期的引用 == 当前引用
        expectedStamp == current.stamp &&  // 预期的版本 == 当前版本
        ((newReference == current.reference &&
          newStamp == current.stamp) ||  // 如果更新后的引用和标志和当前的引用和标志相等，则直接返回 true
         casPair(current, Pair.of(newReference, newStamp))); // 通过 Pair#of(T reference, int stamp) 方法，生成一个新的 Pair 对象，与当前 Pair CAS 替换。
}

private boolean casPair(Pair<V> cmp, Pair<V> val) {
    // 调用的 object 的 cas 方法
    return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val);
}
```

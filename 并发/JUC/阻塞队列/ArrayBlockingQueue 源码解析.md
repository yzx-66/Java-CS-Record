#### ArrayBlockingQueue

* 一个由**数组**实现的**有界**阻塞队列，其大小在构造时由构造函数来决定，确认之后就不能再改变了
* 支持对等待的生产者线程和使用者线程进行排序的可选公平策略，但是在默认情况下不保证线程公平的访问，在构造时可以选择公平策略（`fair = true`）,就是说 ReentrantLock 会变成公平锁。公平性通常会降低吞吐量，但是减少了可变性和避免了“不平衡性”。
* 底层实现就是数组，然后用 Lock 加锁，同时维护队头和队尾索引，并且是循环的



##### 属性

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, Serializable {

    private static final long serialVersionUID = -817911632652898426L;
    
    // 定长数组，维护 ArrayBlockingQueue 的元素
    final Object[] items;
    // 队首位置
    int takeIndex;
    // 队尾位置
    int putIndex;
    // 元素个数
    int count;
    
    // 重入锁
    final ReentrantLock lock;
    // notEmpty condition
    private final Condition notEmpty;
    // notFull condition
    private final Condition notFull;
    
    // 迭代器
    transient ArrayBlockingQueue.Itrs itrs;
    
    
}
```



##### 构造方法

````java
public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
}
    
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
````



##### 入队

###### add

```java
// ArrayBlockingQueue.java
@Override
public boolean add(E e) {
    return super.add(e);
}

/**
 * super.add()：AbstractQueue#add，如下
 */
public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}
```



###### offer

* 不可超时

```java
@Override
public boolean offer(E e) {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        if (count == items.length)
            // 如果超过最大长度，返回 false
            return false;
        else {
            // 入队
            enqueue(e);
            return true;
        }
    } finally {
        // 释放锁
        lock.unlock();
    }
}

private void enqueue(E x) {
    // 添加元素
    final Object[] items = this.items;
    // 在 putIndex 的位置添加
    items[putIndex] = x;
    // 到达队尾，回归队头（由此可见是循环的，所以就少了每次出队后要进行 arraycopy 的操作）
    if (++putIndex == items.length)
        putIndex = 0;
    // 总数+1
    count++;
    // 通知阻塞在等待非空的线程
    notEmpty.signal();
}
```

* 可超时

```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    checkNotNull(e);
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    // 获得锁
    lock.lockInterruptibly();
    try {
        // 若队列已满，循环等待被通知，再次检查队列是否非空（也是为了防止意外唤醒）
        while (count == items.length) {
            // 可等待的时间小于等于零，直接返回失败
            if (nanos <= 0)
                return false;
            // 等待，释放锁加入 Condition 等待队列，直到超时
            nanos = notFull.awaitNanos(nanos); // 返回的为剩余可等待时间，相当于每次等待，都会扣除相应已经等待的时间。
        }
        // 入队
        enqueue(e);
        return true;
    } finally {
        // 解锁
        lock.unlock();
    }
}
```



###### put

```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    // 获得锁
    lock.lockInterruptibly();
    try {
        // 若队列已满，循环等待被通知，再次检查队列是否非空
        while (count == items.length)
            // 等待非满的时候被唤醒
            notFull.await();
        // 入队
        enqueue(e);
    } finally {
        // 解锁
        lock.unlock();
    }
}
```



##### 出队

###### poll

* 不可超时

```java
public E poll() {
    // 获得锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 获得头元素
        return (count == 0) ? null : dequeue();
    } finally {
        // 释放锁
        lock.unlock();
    }
}

private E dequeue() {
    final Object[] items = this.items;
    // 去除队首元素
    E x = (E) items[takeIndex];
    items[takeIndex] = null; // 置空
    // 到达队尾，回归队头（循环的）
    if (++takeIndex == items.length)
        takeIndex = 0;
    // 总数 - 1
    count--;
    // 维护下迭代器
    if (itrs != null)
        itrs.elementDequeued();
    // 通知阻塞在入列的线程
    notFull.signal();
    return x;
}
```

* 可超时

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    // 获得锁
    lock.lockInterruptibly();
    try {
        // 若队列已空，循环等待被通知，再次检查队列是否非空
        while (count == 0) {
            // 可等待的时间小于等于零，直接返回 null
            if (nanos <= 0)
                return null;
            // 等待，直到超时
            nanos = notEmpty.awaitNanos(nanos); // 返回的为剩余可等待时间，相当于每次等待，都会扣除相应已经等待的时间。
        }
        // 出队
        return dequeue();
    } finally {
        // 解锁
        lock.unlock();
    }
}
```



###### take

````java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    // 获得锁
    lock.lockInterruptibly();
    try {
        // 若队列已空，循环等待被通知，再次检查队列是否非空
        while (count == 0)
            notEmpty.await();
        // 出列
        return dequeue();
    } finally {
        // 解锁
        lock.unlock();
    }
}
````



##### 删除

```java
public boolean remove(Object o) {
    if (o == null) return false;
    final Object[] items = this.items;
    // 获得锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count > 0) {
            final int putIndex = this.putIndex;
            int i = takeIndex;
            // 循环向下查找，若匹配，则进行移除。
            do {
                if (o.equals(items[i])) {
                    removeAt(i);
                    return true;
                }
                if (++i == items.length)
                    i = 0;
            } while (i != putIndex);
        }
        return false;
    } finally {
        // 释放锁
        lock.unlock();
    }
}

void removeAt(final int removeIndex) {
    final Object[] items = this.items;
    // 移除的为队头，直接移除即可
    if (removeIndex == takeIndex) {
        // removing front item; just advance
        items[takeIndex] = null;
        // 更新队头索引
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
    // 移除非队头，移除的同时，需要向前复制，填补这个空缺。
    } else {
        final int putIndex = this.putIndex;
        for (int i = removeIndex;;) {
            int next = i + 1;
            if (next == items.length)
                next = 0;
            if (next != putIndex) {
                items[i] = items[next];
                i = next;
            } else {
                // 找到了要删除的位置
                items[i] = null;
                // 更新队尾索引
                this.putIndex = i;
                break;
            }
        }
        count--;
        if (itrs != null)
            itrs.removedAt(removeIndex);
    }
    // 通知
    notFull.signal();
}
```


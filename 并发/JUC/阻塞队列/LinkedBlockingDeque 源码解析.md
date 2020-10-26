#### LinkedBlockingDeque

* 实现接口 BlockingDeque，而 BlockingDeque 又继承接口 BlockingQueue ，提供了一系列的以First和Last结尾的方法，如addFirst、addLast、peekFirst、peekLast，为双向操作Queue提供了可能。
* 底层实现就是链表，然后用 Lock 加锁，维护了头节点和尾节点
* LinkedBlockingQueue 介绍省略



##### 属性

```java
public class LinkedBlockingDeque<E>
    extends AbstractQueue<E>
    implements BlockingDeque<E>, java.io.Serializable {

    // 双向链表的表头
    transient Node<E> first;

    // 双向链表的表尾
    transient Node<E> last;

    // 大小，双向链表中当前节点个数
    private transient int count;

    // 容量，在创建LinkedBlockingDeque时指定的
    private final int capacity;

    final ReentrantLock lock = new ReentrantLock();

    private final Condition notEmpty = lock.newCondition();

    private final Condition notFull = lock.newCondition();
    
    
    static final class Node<E> {
    	E item;
    	Node<E> prev;
    	Node<E> next;

    	Node(E x) { item = x; }
	}
}
```



##### 基础方法

* LinkedBlockingDeque 的add、put、offer、take、peek、poll系列方法都是通过调用XXXFirst，XXXLast方法。 

* 这里就仅以putFirst、putLast、takeFirst、takeLast分析下。 offer 和 poll 的不超时实际就是比 put 和 take 在失败时不会用 Codition 等待，offer 和 poll 的超时实际就是 Codition 等待加个超时时间而已



###### putFirst

```java
public void putFirst(E e) throws InterruptedException {
    // check null
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    // 获取锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        while (!linkFirst(node))
            // 在notFull条件上等待，直到被唤醒或中断
            notFull.await();
    } finally {
        // 释放锁
        lock.unlock();
    }
}

private boolean linkFirst(Node<E> node) {
    // 超出容量
    if (count >= capacity)
        return false;

    // 首节点
    Node<E> f = first;
    // 新节点的next指向原first
    node.next = f;
    // 设置node为新的first
    first = node;

    // 没有尾节点，设置node为尾节点
    if (last == null)
        last = node;
    // 有尾节点，那就将之前first的pre指向新增node
    else
        f.prev = node;
    ++count;
    // 唤醒notEmpty
    notEmpty.signal();
    return true;
}
```



###### **putLast**

```java
public void putLast(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        while (!linkLast(node))
            notFull.await();
    } finally {
        lock.unlock();
    }
}

private boolean linkLast(Node<E> node) {
    if (count >= capacity)
        return false;
    // 尾节点
    Node<E> l = last;

    // 将Node的前驱指向原本的last
    node.prev = l;

    // 将node设置为last
    last = node;
    // 首节点为null，则设置node为first
    if (first == null)
        first = node;
    else
    //非null，说明之前的last有值，就将之前的last的next指向node
        l.next = node;
    ++count;
    notEmpty.signal();
    return true;
}

```



###### takeFirst

```java
public E takeFirst() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        E x;
        // 尝试出队
        while ( (x = unlinkFirst()) == null)
            // 失败阻塞
            notEmpty.await();
        return x;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
    
private E unlinkFirst() {
    // 首节点
    Node<E> f = first;

    // 空队列，直接返回null
    if (f == null)
        return null;

    // first.next
    Node<E> n = f.next;

    // 节点item
    E item = f.item;

    // 移除掉first ==> first = first.next
    f.item = null;
    f.next = f; // help GC
    first = n;

    // 移除后为空队列，仅有一个节点
    if (n == null)
        last = null;
    else
    // n的pre原来指向之前的first，现在n变为first了，pre指向null
        n.prev = null;
    --count;
    notFull.signal();
    return item;
```





###### takeLast

````java
public E takeLast() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        E x;
        // 尝试出队
        while ( (x = unlinkLast()) == null)
            // 失败阻塞
            notEmpty.await();
        return x;
    } finally {
        // 释放锁
        lock.unlock();
    }
 }

private E unlinkLast() {
    // assert lock.isHeldByCurrentThread();
    Node<E> l = last;
    if (l == null)
        return null;
    Node<E> p = l.prev;
    E item = l.item;
    l.item = null;
    l.prev = l; // help GC
    last = p;
    if (p == null)
        first = null;
    else
        p.next = null;
    --count;
    notFull.signal();
    return item;
}
````

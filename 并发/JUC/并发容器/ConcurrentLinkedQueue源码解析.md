#### ConcurrentLinkedQueue

##### Node

```java
private static class Node<E> {
      /** 节点元素域 */
      volatile E item;
      volatile Node<E> next;

      
      Node(E item) {
          UNSAFE.putObject(this, itemOffset, item);
      }

      boolean casItem(E cmp, E val) {
          return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
      }

      void lazySetNext(Node<E> val) {
          /**
           * putOrderedObject 是 putObjectVolatile 的内存非立即可见版本;lazySet 是使用
  		   * Unsafe.putOrderedObject方法，这个方法在对低延迟代码是很有用的，它能够实现非堵塞的写
		   * 入，这些写入不会被 Java 的 JIT 重新排序指令(instruction reordering)，这样它使用快速的存储-存
		   * 储(store-store) barrier,而不是较慢的存储-加载(store-load) barrier,后者总是用在 volatile 的写操
		   * 作上，这种性能提升是有代价的，虽然便宜，也就是写后结果并不会被其他线程看到，甚至是自己
		   * 的线程，通常是几纳秒后被其他线程看到，这个时间比较短，所以代价可以忍受。
		   *
		   * 通常使用在 volatile 修适的变量上
           */
          UNSAFE.putOrderedObject(this, nextOffset, val);
      }

      boolean casNext(Node<E> cmp, Node<E> val) {
          return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
      }

    
      // Unsafe mechanics
      private static final sun.misc.Unsafe UNSAFE;
      /** 偏移量 */
      private static final long itemOffset;
      /** 下一个元素的偏移量 */
     private static final long nextOffset;

      static {
          try {
              UNSAFE = sun.misc.Unsafe.getUnsafe();
              Class<?> k = Node.class;
              itemOffset = UNSAFE.objectFieldOffset
                      (k.getDeclaredField("item"));
              nextOffset = UNSAFE.objectFieldOffset
                      (k.getDeclaredField("next"));
          } catch (Exception e) {
              throw new Error(e);
          }
      }
}
```



##### 构造方法

````java
public ConcurrentLinkedQueue() {
    // 构造 head 和 tail 节点
    head = tail = new Node<E>(null);
}
    
    
public ConcurrentLinkedQueue(Collection<? extends E> c) {
    Node<E> h = null, t = null;
    for (E e : c) {
        checkNotNull(e);
        Node<E> newNode = new Node<E>(e);
        if (h == null)
            h = t = newNode;
        else {
            // 遍历，然后往后连接就可以了
            t.lazySetNext(newNode);
            t = newNode;
        }
    }
    if (h == null)
        h = t = new Node<E>(null);
    head = h;
    tail = t;
}
````



##### offer

* 保证插入成功，但不保证 tail 更新成功（通过 casNext 连接下一个节点为 null 的节点，在连接时会向后找到 next 是 null 的真正尾节点）

* tail 虽然不保证是真正的头节点，但是会非常接近或者就是正真的头节点，因为某个线程尾节点更新失败就说明其他线程更新成功了，所以会一直靠近真正的尾节点

```java
public boolean offer(E e) {
    //检查节点是否为null
    checkNotNull(e);
    // 创建新节点
    final Node<E> newNode = new Node<E>(e);

    // 死循环 直到成功为止
    // 从 tail 开始向后遍历，因为不保证 tail 即使真正的尾节点
    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        // q == null 表示 p 目前就是队尾，尝试加入到队列尾
        if (q == null) {                                // --- 1
            // casNext：t 节点的 next 指向当前节点
            // 如果设置 next 失败，即 tail 已经改变了，那么会接着循环，必须保证这个节点被成功连接
            if (p.casNext(null, newNode)) {             // --- 2
                // node 加入节点后会导致tail距离最后一个节点相差大于一个，需要更新tail
                if (p != t)                             // --- 3
                    // 更新 tail 不用保证成功，只保证上面的更新 next 成功（即：保证连接成功，但不保证 tail 更新成功）
                    // 因为 tail 就算指向的不是真正的尾节点，插入的前提是找到 p.next == null 才会插入，所以会往后遍历
    				// 更新失败说明有并发，那么肯定是被其他线程更新了，所以 tail 还是会一直不断接近或者等于真正的尾节点
                    casTail(t, newNode);                // --- 4
                return true;
            }
        }
        // 如果 q != null ，则表示其他线程已经修改了 tail 的指向
        // p == q 等于自身
        else if (p == q)                                // --- 5
            // p == q 代表着该节点已经被删除了
            // 由于多线程的原因，我们 offer() 的时候也会 poll()，如果 offer() 的时候正好该节点已经 poll() 了，
            // 那么在 poll() 方法中的 updateHead() 方法会将被丢弃的 head.next = head，所以被丢弃节点的 next 
            // 会指向自己，即：p.next == p，这样的话就必须重新设置p。
            // 可以理解为 t != tail ? tail : head，并不是说直接去等于 tail，因为 poll() 方法中是不会更新 tail 的，
            // 所以 tail 就会依然指向被丢弃的节点，那么如果没有新的节点入队去接着更新 tail ，那就说明队列空了，那就指向 head
            p = (t != (t = tail)) ? t : head;           // --- 6
        // tail并没有指向尾节点
        else
            // tail 已经不是最后一个节点，将 p 指向最后一个节点
            p = (p != t && t != (t = tail)) ? t : q;    // --- 7
    }
}
```



##### poll

* 保证出队成功，但不保证 head 更新成功（通过 cas 设置出队节点的 item 为 null，在出队时会向后找到第一个 item 不对 null 的节点） 

* head 虽然不保证是真正的头节点，但是会非常接近或者就是正真的头节点，因为某个线程头节点更新失败就说明其他线程更新成功了，所以会一直靠近真正的头节点

```java
public E poll() {
    // 如果出现 p 被删除的情况需要从 head 重新开始
    restartFromHead:      
    for (;;) {
        // 从 head 开始向后遍历，因为不保证 head 就是真正的头节点
        for (Node<E> h = head, p = h, q;;) {

            // 节点 item
            E item = p.item;

            // item 不为null，那就可以对这个节点出队，则将item cas 设置为null
            if (item != null && p.casItem(item, null)) {                    // --- 1
                // p != head 则更新head
                if (p != h)                                                 // --- 2
                    // p.next != null，则将head更新为p.next ,否则更新为p
                    // 不用保证成功
                    updateHead(h, ((q = p.next) != null) ? q : p);          // --- 3
                return item;
            }
            // p.next == null 此次获取时队列为空（比 offer 就多了这一种情况）
            else if ((q = p.next) == null) {                                // --- 4
                // 更新头节点为这个 item 是 null 的 p 节点，因为走到这个 else if ，那么 p.item = null
                // 但是有可能是遍历了一些节点才到这个 p，所以更新一下可以提高线程再 offer 时节点的查找效率
                // 可成功可失败
                updateHead(h, p);
                // 返回空
                return null;
            }
            // 当一个线程在poll的时候，另一个线程已经把当前的p从队列中删除（即 p.next = p），
            // p 已经被移除不能继续，需要重新开始
            else if (p == q)                                                // --- 5
                continue restartFromHead;
            else
                p = q;                                                      // --- 6
        }
    }
}

final void updateHead(Node<E> h, Node<E> p) {
    if (h != p && casHead(h, p))
        // 如果 head 改为 p成功，那就让已经出队的 h.next = h，这样做的目的是做个已出队标识
        h.lazySetNext(h);
}
```

* 说明

  ![](https://img-blog.csdnimg.cn/20201015094636191.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  



##### 小结

* 要实现一个线程安全的队列有两种方式：阻塞和非阻塞。阻塞队列无非就是锁的应用，而非阻塞则是CAS算法的应用。CoucurrentLinkedQueue 就是 CAS

* 队列中最后一个元素的 next 为 null，并且入队时必须是连接 next 为 null 的真正尾节点。

* 队列中所有未删除的节点的 item 都不能为 null 且都能从 head 节点遍历到；对于要删除的节点，不是直接将其设置为 null，而是先将其 item 域设置为 null（迭代器会跳过item为null的节点）

* 允许 head 和 tail 更新滞后。即 head、tail 不总是指向第一个元素和最后一个元素。


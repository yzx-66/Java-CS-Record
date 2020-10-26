#### LinkedList

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200929161716432.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


- java.util.Deque 接口，提供**双端**队列的功能，LinkedList 支持快速的在头尾添加元素和读取元素，所以很容易实现该特性。

- 继承了 java.util.AbstractSequentialList 抽象类，它是 AbstractList 的子类，实现了只能**连续**访问“数据存储”，基于迭代器顺序遍历后，从而实现后续的操作。例如 `#get(int index)`、`#add(int index, E element)` 等等**随机**操作的方法。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200929161739504.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)






##### 数据结构

```java
// 链表大小
transient int size = 0;

// 头节点
transient Node<E> first;

// 尾节点
transient Node<E> last;

// 节点
private static class Node<E> {

    // 元素
    E item;
   	
   	// 前一个节点
    Node<E> next;
    
    // 后一个节点
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }

}
```



##### 构造方法

```java
public LinkedList() {
}

public LinkedList(Collection<? extends E> c) {
    this();
    // 添加 c 到链表中
    addAll(c);
}
```



##### 添加元素

- 未指定位置（尾插）

```java
public boolean add(E e) {
    // <X> 添加末尾
    linkLast(e);
    return true;
}

void linkLast(E e) {
    // <1> 记录原 last 节点
    final Node<E> l = last;
    // <2> 创建新节点
    // 第一个参数表示，newNode 的前一个节点为 l 。
    // 第二个参数表示，e 为元素。
    // 第三个参数表示，newNode 的后一个节点为 null 。
    final Node<E> newNode = new Node<>(l, e, null);
    // <3> last 指向新节点
    last = newNode;
    // <4.1> 如果原 last 为 null ，说明 first 也为空，则 first 也指向新节点
    if (l == null)
        first = newNode;
    // <4.2> 如果原 last 非 null ，说明 first 也非空，则原 last 的 next 指向新节点。
    else
        l.next = newNode;
    // <5> 增加链表大小
    size++;
    // <6> 增加数组修改次数
    modCount++;
}

```

- 指定位置

```java
public void add(int index, E element) {
    // 校验不要超过范围
    checkPositionIndex(index);

    // 如果刚好等于链表大小，直接添加到尾部即可
    if (index == size)
        linkLast(element);
    // 添加到第 index 的节点的前面
    else
        linkBefore(element, node(index));
}

Node<E> node(int index) {
    // assert isElementIndex(index);

    // 如果 index 小于 size 的一半，就正序遍历，获得第 index 个节点
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    // 如果 index 大于 size 的一半，就倒序遍历，获得第 index 个节点
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    // 获得 succ 的前一个节点
    final Node<E> pred = succ.prev;
    // 创建新的节点 newNode
    final Node<E> newNode = new Node<>(pred, e, succ);
    // 设置 succ 的前一个节点为新节点
    succ.prev = newNode;
    // 如果 pred 为 null ，说明 first 也为空，则 first 也指向新节点
    if (pred == null)
        first = newNode;
    // 如果 pred 非 null ，说明 first 也为空，则 pred 也指向新节点
    else
        pred.next = newNode;
    // 增加链表大小
    size++;
    // 增加数组修改次数
    modCount++;
}
```



其他方法略

- ```public void addFirst(E e)```
- ```public void addLast(E e)```
- ```public boolean offerFirst(E e)```
- ```public boolean offerLast(E e)```
- ```public void push(E e)```

- ```public boolean addAll(Collection<? extends E> c)```
- ```public boolean addAll(int index, Collection<? extends E> c)```



##### 链表扩容

LinkedList 不存在扩容的需求，因为通过 Node 的前后指向即可。



##### 删除元素

- 指定索引

```java
public E remove(int index) {
    checkElementIndex(index);
    // 获得第 index 的 Node 节点，然后进行移除。
    return unlink(node(index));
}

E unlink(Node<E> x) {
    // assert x != null;
    // <1> 获得 x 的前后节点 prev、next
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    // <2> 将 prev 的 next 指向下一个节点
    if (prev == null) { // <2.1> 如果 prev 为空，说明 first 被移除，则直接将 first 指向 next
        first = next;
    } else { // <2.2> 如果 prev 非空
        prev.next = next; // prev 的 next 指向 next
        x.prev = null; // x 的 pre 指向 null
    }

    // <3> 将 next 的 prev 指向上一个节点
    if (next == null) { // <3.1> 如果 next 为空，说明 last 被移除，则直接将 last 指向 prev
        last = prev;
    } else { // <3.2> 如果 next 非空
        next.prev = prev; // next 的 prev 指向 prev
        x.next = null; // x 的 next 指向 null
    }

    // <4> 将 x 的 item 设置为 null ，帮助 GC
    x.item = null;
    // <5> 减少链表大小
    size--;
    // <6> 增加数组的修改次数
    modCount++;
    return element;
}
```

- 指定元素

```java
public boolean remove(Object o) {
    if (o == null) { // o 为 null 的情况
        // 顺序遍历，找到 null 的元素后，进行移除
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        // 顺序遍历，找到等于 o 的元素后，进行移除
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```



其他方法略

- ```public E remove() ```
- ```public E removeFirst()```
- ```public E removeLast()```
- ```public boolean removeFirstOccurrence(Object o)```
- ```public boolean removeLastOccurrence(Object o)```
- ```public E poll() ```
- ```public E pollFirst()```
- ```public E pollLast()```
- ```public E pop()```
- ```public boolean removeAll(Collection<?> c) ```
- ```public boolean retainAll(Collection<?> c) ```



##### 查找

- 查找指定元素索引

```java
public int indexOf(Object o) {
    int index = 0;
    if (o == null) { // 如果 o 为 null 的情况
        // 顺序遍历，如果 item 为 null 的节点，进行返回
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index; // 找到
            index++;
        }
    } else { // 如果 o 非 null 的情况
        // 顺序遍历，如果 item 为 o 的节点，进行返回
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index; // 找到
            index++;
        }
    }
    // 未找到
    return -1;
}
```



- 查找指定位置的元素

```java
public E get(int index) {
    checkElementIndex(index);
    // 基于 node(int index) 方法实现
    return node(index).item;
}
```



其他省略

- ```public boolean contains(Object o)```
- ```public int lastIndexOf(Object o)```
- ```public E peek()```
- ```public E peekFirst()```
- ```public E peekLast() ```
- ```public E element()```
- ```public E getFirst()```
- ```public E getLast()```



##### 清空

```java
public void clear() {
    // 顺序遍历链表，设置每个节点前后指向为 null
    // 通过这样的方式，帮助 GC
    for (Node<E> x = first; x != null; ) {
        // 获得下一个节点
        Node<E> next = x.next;
        // 设置 x 的 item、next、prev 为空。
        x.item = null;
        x.next = null;
        x.prev = null;
        // 设置 x 为下一个节点
        x = next;
    }
    // 清空 first 和 last 指向
    first = last = null;
    // 设置链表大小为 0
    size = 0;
    // 增加数组修改次数
    modCount++;
}
```



##### 迭代器

```java
public Iterator<E> iterator() {
    return listIterator();
}

private class ListItr implements ListIterator<E> {

    // 最后返回的节点
    private Node<E> lastReturned;
    
    // 下一个节点
    private Node<E> next;
    
    /**
     * 下一个访问元素的位置，从下标 0 开始。
     *
     * 主要用于 {@link #nextIndex()} 中，判断是否遍历结束
     */
    private int nextIndex;
    
    /**
     * 创建迭代器时，数组修改次数。
     *
     * 在迭代过程中，如果数组发生了变化，会抛出 ConcurrentModificationException 异常。
     */
    private int expectedModCount = modCount;

    ListItr(int index) {
        // assert isPositionIndex(index);
        // 获得下一个节点
        next = (index == size) ? null : node(index);
        // 下一个节点的位置
        nextIndex = index;
    }

    public boolean hasNext() {
        return nextIndex < size;
    }

    public E next() {
        // 校验是否数组发生了变化
        checkForComodification();
        // 如果已经遍历到结尾，抛出 NoSuchElementException 异常
        if (!hasNext())
            throw new NoSuchElementException();

        // lastReturned 指向，记录最后访问节点
        lastReturned = next;
        // next 指向，下一个节点
        next = next.next;
        // 下一个节点的位置 + 1
        nextIndex++;
        // 返回 lastReturned
        return lastReturned.item;
    }

    public boolean hasPrevious() {
        return nextIndex > 0;
    }

    public E previous() {
        // 校验是否数组发生了变化
        checkForComodification();
        // 如果已经遍历到结尾，抛出 NoSuchElementException 异常
        if (!hasPrevious())
            throw new NoSuchElementException();

        // 修改 lastReturned 和 next 的指向。此时，lastReturned 和 next 是相等的。
        lastReturned = next = (next == null) ? last : next.prev;
        // 下一个节点的位置 - 1
        nextIndex--;
        // 返回 lastReturned
        return lastReturned.item;
    }

    public int nextIndex() {
        return nextIndex;
    }

    public int previousIndex() {
        return nextIndex - 1;
    }

    public void remove() {
        // 校验是否数组发生了变化
        checkForComodification();
        // 如果 lastReturned 为空，抛出 IllegalStateException 异常，因为无法移除了。
        if (lastReturned == null)
            throw new IllegalStateException();

        // 获得 lastReturned 的下一个
        Node<E> lastNext = lastReturned.next;
        // 移除 lastReturned 节点
        unlink(lastReturned);
        // 此处，会分成两种情况
        if (next == lastReturned) // 说明发生过调用 `#previous()` 方法的情况，next 指向下一个节点，而 nextIndex 是无需更改的
            next = lastNext;
        else
            nextIndex--; // nextIndex 减一。

        // 设置 lastReturned 为空
        lastReturned = null;
        // 增加数组修改次数
        expectedModCount++;
    }

    public void set(E e) {
        // 如果 lastReturned 为空，抛出 IllegalStateException 异常，因为无法修改了。
        if (lastReturned == null)
            throw new IllegalStateException();
        // 校验是否数组发生了变化
        checkForComodification();
        // 修改 lastReturned 的 item 为 e
        lastReturned.item = e;
    }

    public void add(E e) {
        // 校验是否数组发生了变化
        checkForComodification();
        // 设置 lastReturned 为空
        lastReturned = null;
        // 此处，会分成两种情况
        if (next == null) // 如果 next 已经遍历到尾，则 e 作为新的尾节点，进行插入。算是性能优化
            linkLast(e);
        else // 插入到 next 的前面
            linkBefore(e, next);
        // nextIndex 加一。
        nextIndex++;
        // 增加数组修改次数
        expectedModCount++;
    }

    public void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        // 遍历剩余链表
        while (modCount == expectedModCount && nextIndex < size) {
            // 执行 action 逻辑
            action.accept(next.item);
            // lastReturned 指向 next
            lastReturned = next;
            //  next 指向下一个节点
            next = next.next;
            // nextIndex 加一。
            nextIndex++;
        }
        // 校验是否数组发生了变化
        checkForComodification();
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }

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

    // 写入链表大小
    s.writeInt(size);
    
    // 顺序遍历，逐个序列化
    for (Node<E> x = first; x != null; x = x.next)
        s.writeObject(x.item);
}
```



- 反序列化

```java
@java.io.Serial
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    // 读取非静态属性、非 transient 属性
    s.defaultReadObject();

    // 读取 size
    int size = s.readInt();

    // 顺序遍历，逐个反序列化
    for (int i = 0; i < size; i++)
        linkLast((E)s.readObject()); // 添加到链表尾部
}
```



##### 小结

- LinkedList 基于节点实现的**双向**链表的 List ，每个节点都指向前一个和后一个节点从而形成链表。
- LinkedList 提供队列、双端队列、栈的功能。
  - 因为 `first` 节点，所以提供了队列的功能的实现的功能。
  - 因为 `last` 节点，所以提供了栈的功能的实现的功能。
  - 因为同时具有 `first` + `last` 节点，所以提供了双端队列的功能。
- LinkedList 随机访问**平均**时间复杂度是 O(n) ，查找指定元素的**平均**时间复杂度是 O(n) 。
- LinkedList 添加元素的最好时间复杂度是 O(1) ，最坏时间复杂度是 O(n) ，平均时间复杂度是 O(n) 。
  - 最好时间复杂度发生在头部、或尾部添加的情况。
- LinkedList 移除指定位置的元素的最好时间复杂度是 O(1) ，最坏时间复杂度是 O(n) ，平均时间复杂度是 O(n) 。
  - 最好时间复杂度发生在头部、或尾部移除的情况。
- LinkedList 移除指定位置的元素的最好时间复杂度是 O(1) ，最坏时间复杂度是 O(n) ，平均时间复杂度是 O(n) 。
  - 最好时间复杂度发生在头部移除的情况。


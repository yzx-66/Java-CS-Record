#### HashTable

##### 区别 HashMap

###### 线程安全

Hashtable 是线程安全的，HashMap 不是线程安全的。

Hashtable 所有的元素操作如put-get等都是 synchronized 修饰的，而 HashMap 并没有。

```java
public synchronized V put(K key, V value);
public synchronized V get(Object key);
// ...
```



###### 性能优劣

- 既然 Hashtable 是线程安全的，每个方法都要阻塞其他线程，所以 Hashtable 性能较差，HashMap 性能较好，使用更广。
- 如果要线程安全又要保证性能，建议使用 JUC 包下的 ConcurrentHashMap。



###### NULL

Hashtable 是不允许键或值为 null 的，HashMap 的键值则都可以为 null。

- Hashtable put 方法逻辑：

```java
 public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();

        // ...
} 
```

- HashMap hash 方法逻辑：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

即： Hashtable key 为 null 会直接抛出空指针异常，value 为 null 手动抛出空指针异常，而 HashMap 的逻辑对 null 作了特殊处理。



###### 实现方式

- Hashtable 的继承源码：

```java
public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable
```

- HashMap 的继承源码：

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```

即：两者继承的类不一样，Hashtable 继承了 Dictionary（ JDK 1.0 ），而 HashMap 继承的是 AbstractMap（JDK 1.2）



###### 容量扩容

初始容量

- HashMap 的初始容量为：16

  ```java
  /**
   * Constructs a new, empty hashtable with a default initial capacity (11)
   * and load factor (0.75).
   */
  public Hashtable() {
      this(11, 0.75f);
  }
  ```

  

- Hashtable 初始容量为：11，

  ```java
  /**
   * Constructs an empty <tt>HashMap</tt> with the default initial capacity
   * (16) and the default load factor (0.75).
   */
  public HashMap() {
      this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
  }
  ```



负载因子

- 两者默认都是 0.75。



扩容（当现有容量大于总容量 \* 负载因子时）

- HashMap 扩容规则为当前容量翻倍，重分布算法是分高位和低位，低位在 newMap 的索引位置不变，高位在 newMap 的索引为（原索引 + oldCap）

- Hashtable 扩容规则为当前容量翻倍 + 1，并且重分布算法是所有元素都重新计算在 newMap 的位置，然后放到 newMap 里。

  ```java
  protected void rehash() {
          int oldCapacity = table.length;
          Entry<?,?>[] oldMap = table;
  
          // overflow-conscious code
      	// 乘二加一
          int newCapacity = (oldCapacity << 1) + 1;
          if (newCapacity - MAX_ARRAY_SIZE > 0) {
              if (oldCapacity == MAX_ARRAY_SIZE)
                  // Keep running with MAX_ARRAY_SIZE buckets
                  return;
              newCapacity = MAX_ARRAY_SIZE;
          }
          Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];
  
          modCount++;
      	// 重新计算阈值
          threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
          table = newMap;
  
          for (int i = oldCapacity ; i-- > 0 ;) {
              for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
                  Entry<K,V> e = old;
                  old = old.next;
  				
                  // 重新计算 hash 值，然后算出在 newMap 的索引
                  int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                  // 然后放到 newMap 该索引槽位的第一个
                  e.next = (Entry<K,V>)newMap[index];
                  // 连接原先该槽位的一个元素 将其放到第二个
                  newMap[index] = e;
              }
          }
  }
  ```



###### 是否树化

- HashMap ：默认 tab 长度大于64 ，且槽位元素大于 8 ，树化
- Hashtable ：始终不树化



###### 迭代器

- HashMap 中的 Iterator 迭代器是 fail-fast 的，因为只实现了 Iterator
- Hashtable 的 Enumerator 不一定会 fail-fast 的，因为实现了 Enumeration 和 Iterator，如果向上转型为 Enumeration，然后通过 hasMoreElements() 和 nextElement() 进行遍历时并不会判断 expectedModCount 

所以，当其他线程改变了HashMap 的结构，如：增加、删除元素，将会抛出 ConcurrentModificationException 异常，而 Hashtable 则不一定会。

- hashTable

```java
    private class Enumerator<T> implements Enumeration<T>, Iterator<T> {
        final Entry<?,?>[] table = Hashtable.this.table;
        int index = table.length;
        Entry<?,?> entry;
        Entry<?,?> lastReturned;
        final int type;

        /**
         * Indicates whether this Enumerator is serving as an Iterator
         * or an Enumeration.  (true -> Iterator).
         */
        final boolean iterator;

        /**
         * The modCount value that the iterator believes that the backing
         * Hashtable should have.  If this expectation is violated, the iterator
         * has detected concurrent modification.
         */
        protected int expectedModCount = Hashtable.this.modCount;

        Enumerator(int type, boolean iterator) {
            this.type = type;
            this.iterator = iterator;
        }

        // -------------------Enumeration methods--------------------------
        public boolean hasMoreElements() {
            Entry<?,?> e = entry;
            int i = index;
            Entry<?,?>[] t = table;
            /* Use locals for faster loop iteration */
            while (e == null && i > 0) {
                e = t[--i];
            }
            entry = e;
            index = i;
            return e != null;
        }

        @SuppressWarnings("unchecked")
        public T nextElement() {
            Entry<?,?> et = entry;
            int i = index;
            Entry<?,?>[] t = table;
            /* Use locals for faster loop iteration */
            while (et == null && i > 0) {
                et = t[--i];
            }
            entry = et;
            index = i;
            if (et != null) {
                Entry<?,?> e = lastReturned = entry;
                entry = e.next;
                return type == KEYS ? (T)e.key : (type == VALUES ? (T)e.value : (T)e);
            }
            throw new NoSuchElementException("Hashtable Enumerator");
        }

        // -------------------Iterator methods--------------------------
        public boolean hasNext() {
            return hasMoreElements();
        }

        public T next() {
            if (Hashtable.this.modCount != expectedModCount)
                throw new ConcurrentModificationException();
            return nextElement();
        }

        public void remove() {
            if (!iterator)
                throw new UnsupportedOperationException();
            if (lastReturned == null)
                throw new IllegalStateException("Hashtable Enumerator");
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            synchronized(Hashtable.this) {
                Entry<?,?>[] tab = Hashtable.this.table;
                int index = (lastReturned.hash & 0x7FFFFFFF) % tab.length;

                @SuppressWarnings("unchecked")
                Entry<K,V> e = (Entry<K,V>)tab[index];
                for(Entry<K,V> prev = null; e != null; prev = e, e = e.next) {
                    if (e == lastReturned) {
                        if (prev == null)
                            tab[index] = e.next;
                        else
                            prev.next = e.next;
                        expectedModCount++;
                        lastReturned = null;
                        Hashtable.this.modCount++;
                        Hashtable.this.count--;
                        return;
                    }
                }
                throw new ConcurrentModificationException();
            }
        }
    }
```



##### 源码

###### put & rehash

```java
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) {
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    // 遍历看是否已有
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }

    addEntry(hash, key, value, index);
    return null;
}
    
private void addEntry(int hash, K key, V value, int index) {
    Entry<?,?> tab[] = table;
    // 和 jdk 1.7 的 hashMap 一样，先扩容再 put ，不过还不如 jdk 1.7，因为及时要放进去的桶为空也扩容，很大概率扩容之后造成浪费
    if (count >= threshold) {
        // Rehash the table if the threshold is exceeded
        // 扩容
        rehash();
        tab = table;
        hash = key.hashCode();
        // 扩容了，所以重新计算 index
        index = (hash & 0x7FFFFFFF) % tab.length;
    }

    // Creates the new entry.
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>) tab[index];
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
    modCount++;
}

protected void rehash() {
    int oldCapacity = table.length;
    Entry<?,?>[] oldMap = table;

    // overflow-conscious code
    // 没分那么多种情况，为了未初始化导致 oldCap 是 0，所以是 2 倍加一 ，
    int newCapacity = (oldCapacity << 1) + 1;
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        if (oldCapacity == MAX_ARRAY_SIZE)
            // Keep running with MAX_ARRAY_SIZE buckets
            return;
        newCapacity = MAX_ARRAY_SIZE;
    }
    Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

    modCount++;
    // 计算threshold
    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    table = newMap;

    for (int i = oldCapacity ; i-- > 0 ;) {
        for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
            Entry<K,V> e = old;
            old = old.next;

            int index = (e.hash & 0x7FFFFFFF) % newCapacity;
            // 每个元素都立即头插，并发环境可能导致循环链表
            e.next = (Entry<K,V>)newMap[index];
            newMap[index] = e;
        }
    }
}
```


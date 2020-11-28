## 补充说明
**jdk1.7 和1.8之后 hashmap区别**

- jdk7 数组+单链表、 jdk8 数组+(单链表+红黑树) 

- jdk7 链表头插、 jdk8 链表尾插 

  - jdk7 使用头插是为了将热数据放到前面，下面取的时候更快，但是每次 resize 后头插的元素反而会被放到最后面，所以没有头插的必要
  - jdk7 头插: resize 后链表可能倒序，并发环境产生循环链导致该线程死循环

- jdk7 先扩容再put、 jdk8 先put再扩容 

  - jdk 7

    ```java
    void addEntry(int hash, K key, V value, int bucketIndex) {
    		//这里当前数组如果大于等于12（假如）阈值的话，并且当前的数组的Entry数组还不能为空的时候就扩容
        	// 注意这里是 >= 
        　　if ((size >= threshold) && (null != table[bucketIndex])) {
    　　　　　　 //扩容数组，比较耗时
           　　 resize(2 * table.length);
            　　hash = (null != key) ? hash(key) : 0;
            　　bucketIndex = indexFor(hash, table.length);
        　　}
    
        　　createEntry(hash, key, value, bucketIndex);
    　　}
    
     void createEntry(int hash, K key, V value, int bucketIndex) {
        　　Entry<K,V> e = table[bucketIndex];
    　　　　//把新加的放在原先在的前面，原先的是e，现在的是new，next指向e
       　　 table[bucketIndex] = new Entry<>(hash, key, value, e);//假设现在是new
        　　size++;
    　　}
    ```

  - jdk 8

    ```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                       boolean evict) {
            // .....
        
        	// 注意这里是 > 
            if (++size > threshold)
                resize();
        
            afterNodeInsertion(evict);
            return null;
    }
    ```

    

  - jdk 7 在插入新节点时，如果要插入的桶不为空说明存在值就发生了hash冲突，那么就必须得扩容，但是如果不发生Hash冲突的话，说明当前桶是空的（后面并没有挂有链表），那就等到下一次发生Hash冲突的时候在进行扩容。

  - 但是当如果以后都没有发生hash冲突产生，那么就不会进行扩容了，减少了一次无用扩容，也减少了内存的使用。

- jdk7 resize 计算 index 是直接按位与、 jdk8 调整后是(原位置)or(原位置+旧容量)

  - 在JDK1.7的时候是直接用hash值和需要扩容的二进制数进行&（这里就是为什么扩容的时候为啥一定必须是2的多少次幂的原因所在，因为如果只有2的n次幂的情况时最后一位二进制数才一定是1，这样能最大程度减少hash碰撞）（hash值 & length-1）
  - 而在JDK1.8的时候直接用了JDK1.7的时候计算的规律，也就是扩容前的原始位置+扩容的大小值=JDK1.8的计算方式，而不再是JDK1.7的那种异或的方法。但是这种方式就相当于只需要判断Hash值的新增参与运算的位是0还是1就直接迅速计算出了扩容后的储存方式。

**hashmap扩容时的头插法和尾插法的区别，为什么头插导致循环链表？**

```java
void resize(int newCapacity){
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    ......
    //创建一个新的Hash Table
    Entry[] newTable = new Entry[newCapacity];
    //将Old Hash Table上的数据迁移到New Hash Table上
    transfer(newTable);
    table = newTable;
    threshold = (int)(newCapacity * loadFactor);
}

void transfer(Entry[] newTable){
    Entry[] src = table;
    int newCapacity = newTable.length;

    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do { // #
                Entry<K,V> next = e.next; 
                int i = indexFor(e.hash, newCapacity);
                // 头插
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}
```



如果一个两个线程同时执行到该链表遍历循环开始处（# 处），

- 如果一个线程阻塞，一个线程继续执行重分布，
- 并且假设 oldTab 该链表存第一个元素与后面 >=1 个元素（下面假设该链表的所有元素）的 hash 值在与 newTab.lenth 按位与运算后得到的 newIndex 和在 oldTab 中的 oldIndex 相同（即 newIndex = oldIndex）

那么该线程执行完重分布后的结果是：把该链表满足 newIndex = oldIndex 的在 newTable 的 oldIndex 位置进行倒转（比如：a b c 会变成 c b a）

现在阻塞的线程开始执行

- 因为原先 oldTab 对应 oldIndex 的第一个元素已经放到了 newTab 的 oldIndex 槽位链表的最后一个，所以第二个线程操作的就是 newTab 的 oldIndex 槽位链表的最后一个
- 又因为前面假设了该链表所有元素的 hash 和 newTab.lenth 按位与后都 newIndex = oldIndex，那么这个newTable 的最后一个元素重分布后就会放到 newTable 的 oldIndex 位置的第一个，从而构成了循环链表
- 构成循环链表后，该线程的 resize 可以正常返回，但任何线程在下一次 get 或者 put 元素遍历时，就会陷入死循环。



但是从 JDK 8 开始，使用 lohead 和 hihead 在把该链表完全分配完到 lohead 或 hihead 后，再分别插入 lohead 和 hihead ，从而避免了重分布一个元素，就立即将其插入 newTab 可能因为 newIndex = oldIndex 导致的死循环，但这种再把链表全部元素重分配完后再插入的方式就完全不存在该问题。

**为什么到了8就扩容？（为什么是8）**

因为注释有说

- 根据泊松分布的概率，在 loadfactor =0.75 的时候 λ = 0.5，此时链表长度等于 8 的概率只有 0.00000006，而且 TreeNode 所占空间基本是 Node 的二倍，所以没必要过早树化
- 当长度小于 8 时，链表和红黑树的时间复杂度并不会相差很多，即就算 O(log8)  比 O(8) 也只小一点点 





**HashMap的负载因子是什么？默认是多少？为什么默认是这个数？**

0.75

必须在 “冲突的机会” 与 “空间利用率” 之间，寻找一种平衡与折衷。

- 加载因子越大，填满的元素越多，空间利用率越高，但发生冲突的机会变大了；

- 加载因子越小，填满的元素越少，冲突发生的机会减小，但空间浪费了更多了；

并且根据泊松分布 loadfactor =0.75 的时候 λ = 0.5， 和默认的 8 就扩容最匹配



**HashMap 为什么默认初始化大小是 16？**

应该就是个经验值（Experience Value），既然一定要设置一个默认的 2 ^ n 作为初始值，那么就需要在效率和内存使用上做一个权衡。这个值既不能太小，也不能太大。太小了就有可能频繁发生扩容，影响效率。太大了又浪费空间，不划算。所以，16 就作为一个经验值被采用了。



在JDK 8中，默认容量的定义为：`static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16 `，其故意把16写成1<<4，就是提醒开发者，这个地方要是2的幂。







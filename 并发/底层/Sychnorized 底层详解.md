### Sychnorized

#### 对象头

* 32位虚拟机对象头大小= Mark Word（4B）+ kclass(4B) = 8B   
* 64位虚拟机对象头大小= Mark Word（8B）+ kclass(4B) = 12B

* 注：在 32 位虚拟机中，1 个机器码等于 4 字节，也就是 32 bits）

![<img src = ' 开发.assets/164dacca57744160 ' align = 'left' />](https://img-blog.csdnimg.cn/202010150932428.png#pic_center)




对象头主要包括两部分数据：**Mark Word（标记字段）**、**Klass Pointer（类型指针）**。

* Mark Word（1 个机器码） 用于存储对象自身的运行时数据，它是实现重量级锁、轻量级锁和偏向锁的关键。如下图所示它被分成两部分，刚**开始时LockWord为被设置为HashCode**、**最低三位表示 LockWord 所处的状态（是否偏向锁 1 bit  + 锁标志位 2 bit ）**。

* Klass Point（1 个机器码） 是对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。

* 对象是数组类型，则还需要1 个机器码，因为 JVM 虚拟机可以通过 Java 对象的元数据信息确定 Java 对象的大小，无法从数组的元数据来确认数组的大小，所以用一块来记录数组长度。

  

  

  ![](https://img-blog.csdnimg.cn/20201015093348144.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




对象头信息是与对象自身定义的数据无关的额外存储成本，但是考虑到虚拟机的空间效率，Mark Word 被设计成一个**非固定**的数据结构以便在极小的空间内存存储尽量多的数据，它会根据对象的状态复用自己的存储空间，也就是说，**Mark Word 会随着程序的运行发生变化**，变化状态如下：

- 32 位虚拟机：

  ​	![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015093943713.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)

  - 每一行，是一种情况。为什么要有 1 bit 的是否是偏向锁，是因为有 5 种状态，偏向锁和无锁状态的锁标志位都是 01 ，所以用 1 bit 的是否偏向锁来区分

  - 无锁状态 和 偏向锁状态不能共存，即如果开启了”偏向锁模式“，那么对象在初始化的时候，其对象头的 MarkWord 部分就是偏向锁状态，而不是无锁状态，其中 thread id = 0。所以说就没法保存 hashcode，所以：

    - 当一个对象已经计算过 identity hash code，它就无法进入偏向锁状态；
    - 当一个对象当前正处于偏向锁状态，并且需要计算其identity hash code的话，则它的偏向锁会被撤销，并且锁会膨胀为轻量级锁或者重量锁；

    

* 64 位虚拟机：

  ​	![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015093849533.png#pic_center)

  - 对于 32 位无锁状态，有 25 bits 没有使用。





#### 锁升级（底层）

(参考：https://github.com/farmerjohngit/myblog/issues/12)



Java中的`synchronized`有偏向锁、轻量级锁、重量级锁三种形式。会按偏向锁->轻量级锁->重量级锁 的顺序升级，并且升级之后基本不会降级。

* 偏向锁：只被一个线程持有。对于轻量级锁每次加锁解锁通常都有一个CAS操作；对于重量级锁而言，加锁也会有一个或多个CAS操作，所以为提高一个对象在一段很长的时间内都只被一个线程用做锁对象场景下的性能，引入了偏向锁，在第一次获得锁时，会有一个CAS操作，之后该线程再获取锁，只会执行几个简单的比较和设置，而不是开销相对较大的CAS命令。
* 轻量锁：不同线程交替持有锁（即 有不是特别多个线程，但同步代码块执行时间短，或者一次持有锁的时间短）
* 重量锁：多线程竞争锁



补充说明

* Lock Record 为线程私有，存在于当前线程栈中，是一个线程栈帧；其主要属性有两个 obj （指向锁的指针）、header（即锁的对象头 mark word 部分）
  * 锁记录只真正作用于轻量级锁中，因为只有轻量锁时 mark word 指向 Lock Record
  * 在偏向锁和重量级锁中只是起过渡作用，只是一个锁记录
* 锁撤销指锁升级、锁释放指当前线程退出同步代码块。
* 对象头初始化时，其 mark word 要么是 偏向锁的匿名偏向状态，要么是无锁状态（如果是无锁状态说明禁用了偏向锁，那么一开始获取的锁是轻量锁）。
* 一旦升级为重量级锁，那么其 mark word 指向 objectmonitor 将不再改变，改变的是其指向的 objectmonitor 的 owner 属性。



##### 偏向锁

对象创建

* 当JVM启用了偏向锁模式（1.6以上默认开启），当新创建一个对象的时候，如果该对象所属的class没有关闭偏向锁模式（默认所有class的偏向模式都是是开启的），那新创建对象的`mark word`将是可偏向状态，此时`mark word中`的thread id 为 0，表示未偏向任何线程，也叫做匿名偏向(anonymously biased)。



加锁过程

* case 1：当该对象第一次被线程获得锁的时候，发现是匿名偏向状态，则会用CAS指令，将`mark word`中的thread id由0改成当前线程Id。如果成功，则代表获得了偏向锁，继续执行同步块中的代码。否则，将偏向锁撤销，升级为轻量级锁。

* case 2：当被偏向的线程再次进入同步块时，发现锁对象偏向的就是当前线程，在通过检查后，会往当前线程的栈中添加一条`Displaced Mark Word`（即 header 属性）为空的`Lock Record`，然后继续执行同步块的代码，因为操纵的是线程私有的栈，因此不需要用到CAS指令；由此可见偏向锁模式下，当被偏向的线程再次尝试获得锁时，仅仅进行几个简单的操作就可以了，在这种情况下，`synchronized`关键字带来的性能开销基本可以忽略。 

* case 3.当其他线程进入同步块时，发现已经有偏向的线程了，则会进入到**撤销偏向锁**的逻辑里，一般来说，会在`safepoint`中（在这个时间点上没有字节码正在执行）去查看偏向的线程是否还存活。
  * 如果存活且还在同步块中则将锁升级为轻量级锁，原偏向的线程继续拥有锁，当前线程则走入到锁升级的逻辑里
  * 如果偏向的线程已经不存活或者不在同步块中，则将对象头的`mark word`改为无锁状态（unlocked），之后再升级为轻量级锁（即发生了竞争，不符合偏向锁的情况了，以后都是轻量级锁）。

* 由此可见，偏向锁升级的时机为：当锁已经发生偏向后，只要有另一个线程尝试获得偏向锁，则该偏向锁就会升级成轻量级锁。当然这个说法不绝对，因为还有批量重偏向这一机制。



调用锁对象的 hashcode()

* 当一个对象已经计算过identity hash code，它就无法进入偏向锁状态；

* 当一个对象当前正处于偏向锁状态，并且需要计算其identity hash code的话，则它的偏向锁会被撤销，并且锁会膨胀为轻量级锁或者重量锁；
  * 轻量锁时 可以通过 mark word 找到 Lock word，然后用其header 属性保存 hashcode 
  * 重量锁时 可以通过 mark word 找到 ObjectMonitor ，然后用 header 属性保存 hashcode 



释放过程

* 当有其他线程尝试获得锁时，是根据遍历偏向线程的`lock record`来确定该线程是否还在执行同步块中的代码。因此偏向锁的解锁很简单，仅仅将栈中的最近一条`lock record`的`obj`字段设置为null。需要注意的是，偏向锁的解锁步骤中**并不会修改对象头中的thread id。**



批量重偏向与撤销

* 存在如下两种情况：（见官方[论文](https://www.oracle.com/technetwork/java/biasedlocking-oopsla2006-wp-149958.pdf)第4小节）:
  * 1.一个线程创建了大量对象并执行了初始的同步操作，之后在另一个线程中将这些对象作为锁进行之后的操作。这种case下，会导致大量的偏向锁撤销操作。
  * 2.存在明显多线程竞争的场景下使用偏向锁是不合适的，例如生产者/消费者队列。

* 原理：
  * 以 class 为单位，为每个 class 维护一个偏向锁撤销计数器，每一次该 class 的对象发生偏向撤销操作（即膨胀，锁升级）时，该计数器 +1 ，当这个值达到重偏向阈值（默认20）时，JVM 就认为该 class 的偏向锁有问题，因此会进行批量重偏向。
  * 每个class对象会有一个对应的 `epoch` 字段，每个处于偏向锁状态对象的 `mark word 中` 也有该字段，其初始值为创建该对象时，class 中的`epoch`的值。每次**发生批量重偏向时，就将该值 + 1**，同时遍历 JVM 中所有线程的栈，**找到该 class 所有正处于加锁状态的偏向锁，将其`epoch`字段改为新值**。



* 批量重偏向（bulk rebias）机制是为了解决第一种场景。
  * 当另外一个线程尝试获得锁时，发现当前对象的 `epoch` 值和 class 的 `epoch` 不相等，那就算当前已经偏向了其他线程，也不会执行撤销操作，而是直接通过CAS操作将其`mark word`的Thread Id 改成当前线程Id。
    * `epoch` 值和 class 的 `epoch` 不相等说明：该对象比 class 其他对象少回到偏向锁状态，所以即使应该升级为轻量级锁，照样保持偏向锁。
    * 为什么会不相等：因为上面说了，偏向锁膨胀 20 次之后，会把 class epoch 加 1 。
      * 对象的 epoch 自增针对的是 **被当前存活的 thread 持有的偏向锁锁对象**。
      * 但还存在一种锁对象是：**在批量重偏向时，没有被任何thread持有（也就是当前没有thread在执行对应的synchronize代码），但之前被thread持有过。所以，这种锁对象的markword是偏向状态的，但它的epoch与klass的epoch不相等。**在下一次其他thread准备持有它时，不会因为当前thread的threadId和锁对象markword中的threadId不同而升级为轻量级锁，而是直接CAS成偏向当前thread的markWord（因为锁对象的epoch与klass的epoch不同），从而达到批量重偏向的优化效果

* 批量撤销（bulk revoke）则是为了解决第二种场景。
  * 当达到重偏向阈值后，假设该class计数器继续增长，当其达到批量撤销的阈值后（默认40），JVM就认为该class的使用场景存在多线程竞争，会标记该class为不可偏向，之后，对于该class的锁，直接走轻量级锁的逻辑。



#####  轻量锁

线程在执行同步块之前，JVM会先在当前的线程的栈帧中创建一个`Lock Record`，其包括一个用于存储对象头中的 `mark word`（官方称之为`Displaced Mark Word`）以及一个指向对象的指针。下图右边的部分就是一个`Lock Record`。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015093921310.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


加锁过程

* 1.在线程栈中创建一个`Lock Record`，将其`obj`（即上图的Object reference）字段指向锁对象。

* 2.直接通过 CAS 指令将 `Lock Record` 的地址存储在对象头的`mark word`中，如果对象处于无锁状态则修改成功，代表该线程获得了轻量级锁。如果失败，进入到步骤3。

* 3.如果是当前线程已经持有该锁了，代表这是一次锁重入。设置`Lock Record`第一部分（`Displaced Mark Word`）为null，起到了一个重入计数器的作用。然后结束。 否则说明发生了竞争，需要膨胀为重量级锁。



释放过程

* 1.遍历线程栈,找到所有`obj`字段等于当前锁对象的`Lock Record`。

* 2.如果`Lock Record`的`Displaced Mark Word`为 null，代表这是一次重入，将`obj`设置为 null 后 continue。

* 3.如果`Lock Record`的`Displaced Mark Word`不为 null，则利用 CAS 指令将对象头的`mark word`恢复成为`Displaced Mark Word`。如果成功，则continue，否则膨胀为重量级锁（失败是因为锁已经膨胀，mark word 已被替换成其他标志）。





轻量级锁主要有两种

* 自旋锁

  所谓自旋，是指当有另一个线程想获取被其它线程持有的锁的时候，不会进入阻塞状态，而是使用空循环来进行自旋。注意：自旋是会消耗cpu的，所以，轻量级锁适用于那些**同步代码块执行的很快的场景**，这样，线程原地等待很短很短的时间就能够获得锁了。

  * 自旋锁的一些问题
    * 如果同步代码块执行的很慢，需要消耗大量的时间，那么这个时侯，其他线程在空循环，消耗cpu。

    * 本来一个线程把锁释放之后，当前线程是能够获得锁的，但是假如这个时候有好几个线程都在竞争这个锁的话，那么有可能当前线程会获取不到锁，还得原地等待继续空循环消耗cup，甚至有可能一直获取不到锁，此后再升级为重量级锁相比直接就是重量级锁更加浪费低效。
    * 基于这个问题，我们必须给线程空循环设置一个次数，当线程超过了这个次数，我们就认为，继续使用自旋锁就不适合了，此时锁会再次膨胀，升级为重量级锁。默认情况下，自旋的次数为10次，用户可以通过 -XX:PreBlockSpin 来进行更改。

* 自适应自旋锁

  所谓自适应就意味着自旋的次数**不再是固定**的，**它是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定**。它怎么做呢？

  - 线程如果自旋成功了，那么下次自旋的次数会更加多，因为虚拟机认为既然上次成功了，那么此次自旋也很有可能会再次成功，那么它就会允许自旋等待持续的次数更多。
  - 反之，如果对于某个锁，很少有自旋能够成功的，那么在以后要或者这个锁的时候自旋的次数会减少甚至省略掉自旋过程，以免浪费处理器资源。

#####  重量锁

重量级锁是我们常说的传统意义上的锁，其通过对象内部的监视器（Monitor）实现。

* Monitor 的**本质**是 ObjectMonitor，其也有自己的队列，最终阻塞调用的还是 t -> park() 函数，但是 park 依赖于底层操作系统的 mutex Lock、condition 信号量、counter计数器（和 LockSupport 的 park / unpark 相同），由于使用 Mutex Lock 和 cond_wait 都需要将当前线程挂起并从用户态切换到内核态来执行，这种切换的代价是非常昂贵的。 
* 每一个 JAVA 对象都会与一个监视器 monitor 关联，重量级锁的状态下，对象的`mark word`为指向一个堆中 monitor 对象的指针。当一个 monitor 被重量锁对象持有后，该对象将处于锁定状态。



一个 monitor 对象包括这么几个关键字段：cxq（下图中的ContentionList），EntryList ，WaitSet，owner。

```c++
ObjectMonitor() {
    _header       = NULL; // 对象头的 mark word
    _count        = 0; // 记录个数
    _waiters      = 0; // wait 状态线程个数
    _recursions   = 0; // 重入计数器
    _object       = NULL; // 关联的对象（和 _header 的对象相同）
    _owner        = NULL; // 哪个线程占有该 monitor
    _WaitSet      = NULL; // 处于wait状态的线程节点
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ; // 当前线程释放锁后，下一个执行的线程
    _cxq          = NULL ; // 获取锁失败的线程节点
    FreeNext      = NULL ;
    _EntryList    = NULL ; // 等待被唤醒的线程节点
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
 }
```



加锁过程

* case 1：如果当前是无锁状态，即 owner 为 null ，如果能CAS设置成功，则当前线程直接获得锁

* case 2：如果是锁重入，_recursions ++ （重入计数加 1）

* case 3：当前线程是之前持有轻量级锁的线程，重入计数重置为 1，设置owner字段为当前线程（膨胀后 owner 是指向Lock Record的指针）

* case 4：该锁已经被占用，先自旋尝试获得锁（设置 owner 为当前线程），这样做的目的是为了减少执行操作系统同步操作带来的开销

  * 自旋的过程中获得了锁，则直接返回

  * 否则调用系统同步，如下：

    ![](https://img-blog.csdnimg.cn/2020101509345379.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


    * 将该线程封装成一个`ObjectWaiter`对象插入到 cxq（单向链表）的队列的队首
    * 调用`park`函数挂起当前线程。在linux系统上，`park`函数最终起作用的是gclib库的`pthread_cond_wait`，JDK的`ReentrantLock`底层也是用该 park方法挂起线程的（后面 LockSupport 部分讲）。
    * 当被唤醒后再尝试获得锁


* case 5 ：如果线程获得锁后调用`Object#wait`方法，则会将线程加入到WaitSet中，当被`Object#notify`唤醒后，会将线程从 WaitSet（单向链表） 移动到cxq或EntryList中去。
  * 需要注意的是，当调用一个锁对象的`wait`或`notify`方法时，**如当前锁的状态是偏向锁或轻量级锁则会先膨胀成重量级锁（因为轻量锁和偏向锁没有保存 wait 线程的存储结构）**。







释放过程

当线程释放锁时，会从 cxq 或 EntryList 中挑选一个线程唤醒，被选中的线程叫做`Heir presumptive`即假定继承人（应该是这样翻译），就是图中的`Ready Thread`，假定继承人被唤醒后会尝试获得锁，但`synchronized`是非公平的，所以假定继承人不一定能获得锁（这也是它叫"假定"继承人的原因）。

* 如果_owner不是当前线程

  * 当前线程是之前持有轻量级锁的线程。由轻量级锁膨胀后还没调用过 enter 方法，_owner会是指向Lock Record的指针。则改为指向当前线程，然后继续执行后面代码
  * 否则异常情况，即当前不是持有锁的线程，抛出异常。

* 如果重入计数器还不为0，则计数器  -1 后返回

* 设置owner为null，即释放锁，这个时刻其他的线程能获取到锁。这里是一个非公平锁的优化；

* 如果当前没有等待的线程则直接返回就好了，因为不需要唤醒其他线程。或者如果说succ不为null，代表当前已经有个"醒着的"继承人线程，那当前线程不需要唤醒任何线程；

* 根据QMode的不同，会执行不同的唤醒策略（QMode默认为0）

  * QMode = 2且 cxq 非空：取 cxq 队列队首的 ObjectWaiter 对象，调用 ExitEpilog方法（该方法会唤醒ObjectWaiter对象的线程）然后立即返回；

  * QMode = 3且 cxq 非空：把 cxq 队列插入到 EntryList的 尾部； 

  * QMode = 4且 cxq 非空：把 cxq 队列插入到 EntryList 的头部；

  * QMode = 0：暂时什么都不做，继续往下看；

* 只有QMode=2的时候会提前返回，等于0、3、4的时候都会继续往下执行：
  * 如果 EntryList 的 首元素非空，就取出来调用 ExitEpilog 方法，该方法会唤醒 ObjectWaiter 对象的线程，然后立即返回；
  * 如果 EntryList 的首元素为空，就将 cxq 的所有元素放入到 EntryList 中，然后再从 EntryList 中取出来队首元素执行 ExitEpilog 方法，然后立即返回；



释放锁说明 Demo

* 代码

  ```java
  public class SyncDemo {
  
      public static void main(String[] args) throws InterruptedException {
          SyncDemo syncDemo1 = new SyncDemo();
          
          syncDemo1.startThreadA();
          Thread.sleep(100);
         
          syncDemo1.startThreadB();
          Thread.sleep(100);
          
          syncDemo1.startThreadC();      
      }
  
      final Object lockobj = new Object();
  
      public void startThreadA() {
          new Thread(() -> {
              synchronized (lockobj) {
                  System.out.println("A get lock");
                  Thread.sleep(500);
                  System.out.println("A release lock");
              }
          }, "thread-A").start();
      }
  
      public void startThreadB() {
          new Thread(() -> {
              synchronized (lockobj) { System.out.println("B get lock"); }
          }, "thread-B").start();
      }
  
      public void startThreadC() {
          new Thread(() -> {
              synchronized (lockobj) { System.out.println("C get lock"); }
          }, "thread-C").start();
      }
  }
  ```

  

* 结果：默认策略下，在A释放锁后一定是C线程先获得锁。

* 原因：因为在获取锁时，是将当前线程插入到cxq的**头部**。而释放锁时，默认策略是：如果EntryList为空，则将cxq中的元素按原有顺序插入到到EntryList，并唤醒第一个线程。也就是**当EntryList为空时，是后来的线程先获取锁**。

* 这点JDK中的Lock机制是不一样的，Synchronized是非公平锁，ReentrantLock是可以是公平也可以是非公平的。

  * Synchronized在特定的情况下**对于已经在等待的线程**是后来的线程先获得锁
  * 而ReentrantLock对于**已经在等待的线程**一定是先来的线程先获得锁；





##### 转换流程

![](https://img-blog.csdnimg.cn/20201015093547916.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


* 偏向锁是否可用指的是
  * 是否开启偏向锁
  * 是否已经锁升级
* 是否锁定指的是
  * 在是否有线程获得了锁
  * 撤销偏向时的是否锁定
    * 是：指当前线程还持有偏向锁，但另一个线程已经来争抢
    * 否：指获得偏向锁的线程已经退出同步代码块，然后有另一个线程来争抢


#### 锁优化

##### 自旋锁

(轻量锁手段)



由来

* 线程的阻塞和唤醒，**需要 CPU 从用户态转为核心态**。频繁的阻塞和唤醒对 CPU 来说是一件负担很重的工作，势必会给系统的并发性能带来很大的压力。
* 同时，我们发现在许多应用上面，**对象锁的锁状态只会持续很短一段时间**。为了这一段很短的时间，频繁地阻塞和唤醒线程是非常不值得的。所以引入自旋锁。



定义

* 所谓自旋锁，就是让该线程等待一段时间，不会被立即挂起，看持有锁的线程是否会很快释放锁。

* 怎么等待呢？**执行一段无意义的循环即可**（自旋）。



不能替代阻塞

* 对处理器数量的要求必须**多核**
* 虽然它可以避免线程切换带来的开销，但是它占用了处理器的时间。
  * 如果持有锁的线程很快就释放了锁，那么自旋的效率就非常好
  * 反之，自旋的线程就会白白消耗掉处理的资源，它不会做任何有意义的工作，反而会带来性能上的浪费。

* 所以说，自旋等待的时间（自旋的**次数**）必须要有一个限度，如果自旋超过了定义的时间仍然没有获取到锁，则应该被挂起。



开启

* 自旋锁在 JDK 1.4.2 中引入，默认关闭，但是可以使用 `-XX:+UseSpinning` 开开启。

* 在 JDK1.6 中默认开启。同时自旋的默认次数为 10 次，可以通过参数 `-XX:PreBlockSpin` 来调整。 

  如果通过参数 `-XX:PreBlockSpin` 来调整自旋锁的自旋次数，会带来诸多不便。假如我将参数调整为 10 ，但是系统很多线程都是等你刚刚退出的时候，就释放了锁（假如你多自旋一两次就可以获取锁），于是 JDK 1.6 引入自适应的自旋锁，让虚拟机会变得越来越聪明。



适应自旋锁

* JDK 1.6 引入了更加聪明的自旋锁，即自适应自旋锁。

* 所谓自适应就意味着自旋的次数**不再是固定**的，**它是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定**。它怎么做呢？
  * 线程如果自旋成功了，那么下次自旋的次数会更加多，因为虚拟机认为既然上次成功了，那么此次自旋也很有可能会再次成功，那么它就会允许自旋等待持续的次数更多。
  * 反之，如果对于某个锁，很少有自旋能够成功的，那么在以后要或者这个锁的时候自旋的次数会减少甚至省略掉自旋过程，以免浪费处理器资源。





##### 锁消除

由来

* 为了保证数据的完整性，我们在进行操作时需要对这部分操作进行同步控制。但是，在有些情况下，JVM检测到不可能存在共享数据竞争，这是JVM会对这些同步锁进行锁消除。如果不存在竞争，锁消除可以节省毫无意义的请求锁的时间。 

定义

* 锁消除的依据是**逃逸分析的数据支持**，对于虚拟机来说需要使用数据流分析来确定。
* 出现原因
  * 程序员自己在不存在数据竞争的代码块前加上同步吗
  * 一些 JDK 的内置 API 时，如 StringBuffer、Vector、HashTable 等，这个时候会存在**隐性的加锁操作**。比如 StringBuffer 的 `#append(..)`方法，Vector 的 `add(...)` 方法
  * 一些三方库为了保证数据同步安全，也会加锁



举例

* 代码

  ```java
  public void vectorTest(){
      Vector<String> vector = new Vector<String>();
      for (int i = 0 ; i < 10 ; i++){
      	vector.add(i + "");
      }
      System.out.println(vector);
  }
  ```

  

* 结果：在运行这段代码时，JVM 可以明显检测到变量 `vector` 没有逃逸出方法 `#vectorTest()` 之外，所以 JVM 可以大胆地将 `vector` 内部的加锁操作消除



##### 锁粗化

* 由来

  在使用同步锁的时候，需要让同步块的作用范围尽可能小：仅在共享数据的实际作用域中才进行同步。这样做的目的，是为了使需要同步的操作数量尽可能缩小，如果存在锁竞争，那么等待锁的线程也能尽快拿到锁。

  在大多数的情况下，上述观点是正确的，但是如果一系列的连续加锁解锁操作，可能会导致不必要的性能损耗，所以引入**锁粗话**的概念。



* 定义

  锁粗话概念比较好理解，就是将**多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁**。

  如上面实例：`vector` 每次 add 的时候都需要加锁操作，JVM 检测到对同一个对象（`vector`）连续加锁、解锁操作，会合并一个更大范围的加锁、解锁操作，即加锁解锁操作会移到 `for` 循环之外。


#### 补充
**用sync构建一个死锁场景，写代码**

```java
Object lockA = new Object();
Object lockB = new Object();

new Thread(()->{
    synchronized (lockA){
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        synchronized (lockB){
        }
    }
}).start();

new Thread(() -> {
    synchronized (lockB){
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();  
        }
        synchronized (lockA){
        }
    }
}).start();
```




**死锁的产生原因、死锁4条件、如何预防、如何排查**

原因

* 竞争的资源是不可剥夺资源（该类资源被分配后不可以被强行收回，只能自行释放）
* 进程间推进顺序非法，例如 P1 持有 R1 并且想要获取 R2，P2 持有资源 R2 并且想要获得 R1



四条件

* 互斥条件：一个资源在一段时间内只可以被一个进程占用
* 不剥夺条件：资源只能在使用完由自己释放
* 保持条件：阻塞时对以获得资源不释放
* 环路等待条件：形成了 进程-资源 的环形链



预防死锁

* 互斥条件不可以破坏，破坏了就线程共享了，那无法保证线程安全
* 不剥夺条件不可以破坏，破坏了就可以随意获得锁，那也无法保证线程安全
* 保持条件可以破坏，当线程获得锁失败阻塞时，可以释放已持有的锁
* 环路等待可以避免，给资源编号，进程按编号递增的顺序请求资源



具体做法

* 超时放弃（对应破坏保持条件）

  当使用 synchronized 关键词提供的内置锁时，只要线程没有获得锁，那么就会永远等待下去，然而 Lock 接口提供了 boolean tryLock(long time, TimeUnit unit) throws InterruptedException方法，该方法可以按照固定时长等待锁，因此线程可以在获取锁超时以后，主动释放之前已经获得的所有的锁。通过这种方式，也可以很有效地避免死锁。

* 以确定的顺序获得锁（对应避免环路等待）

  针对两个特定的锁，可以尝试按照锁对象的hashCode值大小的顺序，分别获得两个锁，这样锁总是会以特定的顺序获得锁，那么死锁也不会发生。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015093722618.png#pic_center)




排查

* jstack 命令	`jstack [pid]`

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015093758559.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




* jconsole、jvisualvm 之类可视化工具

  ![](https://img-blog.csdnimg.cn/20201015093656497.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


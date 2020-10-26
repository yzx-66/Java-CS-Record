### JMM

#### 内存模型

* Java 内存模式是一种虚拟机**规范**，用于屏蔽掉各种硬件和操作系统的内存访问差异，因为 JVM 会针对操作系统不同对 JMM 有不同实现，所以让Java程序在各种平台下都能达到一致的并发效果。

* JMM 规范了 Java 虚拟机与计算机内存是如何协同工作的，主要规范了 **可见性**、**顺序性**、**原子性**



##### 硬件内存结构

结构：

![](https://img-blog.csdnimg.cn/20201015092226972.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


- **多CPU**：一个现代计算机通常由两个或者多个CPU，其中一些CPU还有多核，所以可以同时运行多个线程。

- **CPU寄存器**：每个CPU都包含一系列的寄存器，它们是CPU内内存的基础。CPU在寄存器上访问和执行操作的速度远大于在主存上执行的速度。

- **高速缓存cache**：由于计算机的存储设备与处理器的运算速度之间有着几个数量级的差距，所以现代计算机都使用读写速度尽可能接近处理器运算速度的高速缓存（Cache）来作为内存与处理器之间的缓冲。

  - 将运算需要使用到的数据复制到缓存中，让运算能快速进行，当运算结束后再从缓存同步回内存之中，这样处理器就无须等待缓慢的内存读写了。
  - CPU访问缓存层的速度快于访问主存的速度，但通常比访问内部寄存器的速度还要慢一点。每个CPU可能有一个CPU缓存层，一些CPU还有多层缓存。

- **内存**：一个计算机还包含一个主存。所有的CPU都可以访问主存。主存通常比CPU中的缓存大得多。

  

  

运作原理：

* 当一个CPU需要读取主存时，它会将主存的部分读到CPU缓存中，然后从缓存读取数据，或者将缓存中的部分内容读到内部寄存器中，再在寄存器中执行操作

* 当CPU需要将结果写回到主存中去时，它会将内部寄存器的值刷新到缓存中，然后在某个时间点将值刷新回主存。



存在问题：

* **缓存一致性问题**：在多处理器系统中，每个处理器都有自己的高速缓存，但它们又共享同一主内存，因此当多个处理器的操作都涉及同一块主内存区域时，将可能导致各自的缓存数据不一致的情况。
  * 解决缓存一致性方案有两种
    * 通过在总线加 LOCK# 锁的方式，但存在一个问题，它是采用一种独占的方式来实现的，只能有一个 CPU 能够运行，其他 CPU 阻塞，效率较为低下
    * 通过缓存一致性协议（如 MESI 协议），其核心思想如下：当某个 CPU 在写数据时，如果发现操作的变量是共享变量，则会通知其他 CPU 告知该变量的缓存行是无效的，因此其他 CPU 在读取该变量时，发现其无效会重新从主存中加载数据。
* **指令重排序问题**：为了使得处理器内部的运算单元能尽量被充分利用，处理器可能会对输入代码进行乱序执行优化，再在计算之后将乱序执行的结果重组，保证该结果与顺序执行的结果是一致的，**但并不保证程序中各个语句计算的先后顺序与输入代码中的顺序一致**。
  * 因此，如果存在一个计算任务依赖另一个计算任务的**中间结果**，那么其顺序性并不能靠代码的先后顺序来保证。可以通过内存屏障来保证不被重排序。





##### JMM 定义

概念上的区域划分

* 本地内存：每个线程均有自己的本地内存，是线程私有的。

* 主内存：存储共享变量存储，是线程共享的。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/2020101509231763.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




JMM 的实现

* 硬件层面

  * Java 内存模型和硬件内存架构并不完全一致。对于硬件内存来说只有寄存器、缓存内存、主内存的概念，并没有工作内存和主内存之分，即 Java 内存模型对内存的划分对硬件内存并没有任何影响
  * JMM只是一种抽象的概念，是一组规则，并不实际存在。不管是工作内存的数据还是主内存的数据，对于计算机硬件来说都会存储在计算机主内存中，当然也有可能存储到CPU缓存或者寄存器中。
    * 主内存就是硬件的内存。
    * 本地内存为了获取更好的运行速度，虚拟机及硬件系统可能会让工作内存优先存储于寄存器和高速缓存中。
  * 因此总体上来说，Java内存模型和计算机硬件内存架构是一个相互交叉的关系，是一种抽象概念划分与真实物理硬件的交叉。

  

* JVM 层面

  * JMM 与 Java内存区域的划分 是不同的概念层次，一个是规范一个是实现，如果硬说有关系的话如下：

    * 主内存属于共享数据区域，从某个程度上讲应该包括了堆和方法区。通常存被描述为堆内存。
    * 工作内存数据线程私有数据区域，从某个程度上讲则应该包括程序计数器、虚拟机栈以及本地方法栈。通常存被描述为栈内存。

    

  ​	![在这里插入图片描述](https://img-blog.csdnimg.cn/2020101509234112.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)
















##### 线程间通信

* 线程间通信必须要经过主内存

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015092403835.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




线程 A、B 通信步骤：

* 线程A把本地内存A中更新过的共享变量刷新到主内存中去。

* 线程B到主内存中去读取线程A之前已更新过的共享变量。



实现细节

* 八种操作（并不是指令，具体怎么实现要 JVM 自己决定）
  * lock（锁定）：作用于主内存的变量，把一个变量标识为一条线程独占状态。
  * unlock（解锁）：作用于主内存变量，把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定。
  * read（读取）：作用于主内存变量，把一个变量值从主内存传输到线程的工作内存中，以便随后的load动作使用
  * load（载入）：作用于工作内存的变量，它把read操作从主内存中得到的变量值放入工作内存的变量副本中。
  * use（使用）：作用于工作内存的变量，把工作内存中的一个变量值传递给执行引擎，每当虚拟机遇到一个需要使用变量的值的字节码指令时将会执行这个操作。
  * assign（赋值）：作用于工作内存的变量，它把一个从执行引擎接收到的值赋值给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。
  * store（存储）：作用于工作内存的变量，把工作内存中的一个变量的值传送到主内存中，以便随后的write的操作。
  * write（写入）：作用于主内存的变量，它把store操作从工作内存中一个变量的值传送到主内存的变量中。

* 规则
  * 如果要把一个变量从主内存中复制到工作内存，就需要按顺寻地执行 read 和 load 操作， 如果把变量从工作内存中同步回主内存中，就要按顺序地执行store和write操作。但Java内存模型只要求上述操作必须按顺序执行，而没有保证必须是连续执行，比如 read a、read b、load b、load a。
  * 不允许 read 和 load 、 store 和 write 操作之一单独出现
  * 不允许一个线程丢弃它的最近 assign 的操作，即变量在工作内存中改变了之后必须同步到主内存中。
  * 不允许一个线程无原因地（没有发生过任何assign操作）把数据从工作内存同步回主内存中。
  * 一个新的变量只能在主内存中诞生，不允许在工作内存中直接使用一个未被初始化（load或assign）的变量。即就是对一个变量实施use和store操作之前，必须先执行过了assign和load操作。
  * 一个变量在同一时刻只允许一条线程对其进行lock操作，但lock操作可以被同一条线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁。lock和unlock必须成对出现



##### 并发关键字内存语义

为了实现 JSR133 中定义的新的内存模型，JSR133 中增强了 volatile 和 final 关键字的语义。



volatile 的内存语义

* 当写一个 volatile 内存变量的时候，JMM 会把该线程对应的本地内存中的共享变量值刷新到主内存。

* 当读一个 volatile 变量时，JMM 会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。





synchronized 的内存语义

* 当线程释放锁时，JMM 会把该线程对应的本地内存中共享变量刷新到主内存中。

* 当线程获取锁时，JMM 会把该线程中对应的本地内存置为无效。从而使被监视器保护的临界区代码必须从主内存中读取共享变量。



final 的内存语义

* 初次读一个包含 final 域的对象的引用，与随后初次读这个 final 域，这两个操作之间不能重排序。

* 在构造函数中对一个 final 域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

  * 解释：final 的 初始化是在构造方法，所以如果构造函数还有把 fianl 修适的变量赋值给其他变量，那么有可能这两部重排序，被赋值的变量看到 fianl 变量的初始值。

    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015094030971.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)








##### 解决的问题

**原子性**

- JMM 保证对于变量读写的原子性
  -  Java 内存模型提供的 read 、load 、assign 、use 、store 、write 都是原子性变量操作，所以基本数据类型变量、引用类型变量，这些类型变量的读、写天然具有原子性。但类似于 “ 基本变量++ ” 或 “ volatile ++ ” 这种复合操作并没有原子性，因为是 读 赋值 写 三个操作。
     - long和double的非原子性协定：对于64位的数据，如long和double，Java 内存模型规范允许虚拟机将没有被 volatile 修饰的 64 位数据的读写操作划分为两次32位的操作来进行，即允许虚拟机实现选择可以不保证64位数据类型的 load 、store 、read 和 write 这四个操作的原子性。
       - 即如果有多个线程共享一个并未声明为 volatile 的 long 或 double 类型的变量，并且同时对它们进行读取和修改操作，那么某些线程可能会读取到一个既非原值，也不是其他线程修改值的代表了“半个变量”的数值。
       - 但由于目前各种平台下的商用虚拟机几乎都选择把64位数据的读写操作作为原子操作来对待，因此在编写代码时一般也不需要将用到的 long 和double 变量专门声明为 volatile。
  -  声明为 volatile 的任何类型变量的访问读写是具备原子性的，因为 volatile 定义中说” 线程应该**确保通过排他锁单独获得这个变量** “，所以就算 64 位数据也不会出现读到写了一半的 32 位，因为必须等 volatile 完全写完才可以读。
- JMM 保证对于代码块的原子性
  - 如果应用场景需要一个更大范围的原子性保证，需要使用同步块技术。Java 内存模型提供了 lock 和 unlock 操作来满足这种需求。
    - JVM 提供了字节码指令 monitorenter 和 monitorexist 来隐式地使用这两个操作，反映到 Java代码中就是 synchronized 关键字。



**可见性**

JMM 通过 happens-before 规则来保证可见性 （其中也包括了 volatile、synchronized）



**有序性**

JMM 提供 volatile 来保证一定的有序性。

* 有序性是程序执行的顺序按照代码的先后顺序执行，即使用内存屏障防止重排序

* 一定有序性的原因在 volatile 关键字讲解时说明













#### happens-fore

* JSR-133 使用 happens-before 的概念来阐述操作之间的内存可见性
  * 在 JMM 中，如果一个操作执行的结果需要对另一个操作可见（两个操作既可以是在一个线程之内，也可以是在不同线程之间），那么这两个操作之间必须要存在happens-before关系。

##### 规则

原生 Java 满足的 happens-before 关系规则

- **程序次序规则：一个线程内，按照代码顺序，书写在前面的操作，happens-before 于书写在后面的操作。**
- **锁定规则：一个 unLock 操作，happens-before 于后面对同一个锁的 lock 操作。**
- **volatile 变量规则：对一个变量的写操作，happens-before 于后面对这个变量的读操作。**
- **传递规则：如果操作 A happens-before 操作 B，而操作 B happens-before 操作C，则可以得出，操作 A happens-before 操作C**
- 线程启动规则：Thread 对象的 start 方法，happens-before 此线程的每个一个动作。
- 线程中断规则：对线程 interrupt 方法的调用，happens-before 被中断线程的代码检测到中断事件的发生。
- 线程终结规则：线程中所有的操作，都 happens-before 线程的终止检测，我们可以通过Thread.join() 方法结束、Thread.isAlive() 的返回值手段，检测到线程已经终止执行。
- 对象终结规则：一个对象的初始化完成，happens-before 它的 finalize() 方法的开始



推导出其他满足 happens-before 的规则

* 将一个元素放入一个线程安全的队列的操作，happens-before 从队列中取出这个元素的操作。

* 将一个元素放入一个线程安全容器的操作，happens-before 从容器中取出这个元素的操作。

* 在 CountDownLatch 上的 countDown 操作，happens-before CountDownLatch 上的 await 操作。

* 释放 Semaphore 上的 release 的操作，happens-before 上的 acquire 操作。

* Future 表示的任务的所有操作，happens-before Future 上的 get 操作。

* 向 Executor 提交一个 Runnable 或 Callable 的操作，happens-before 任务开始执行操作。



##### 结论

* 如果操作 A happens-before 操作 B，那么操作 A 在内存上的操作结果对操作 B 都是可见的。


#### volatile

* JSR133 中 volatile 内存语义可以保证变量的可见性

* 且为了配合 happens-before 的 volatile 规则 和 传递规则 所以还采用了内存屏障，防止了一些重排序，提供了一定的有序性

  * 比如 操作 A B C，A 和 B 在一个线程、C 是另外一个线程。
    * 已知关系：A happens-before B（程序次序规则）、B happns-before C （volatile 规则）
    * 那么 A happends-before C（传递规则），因此 A 对 C 必须可见，即使同一线程内 A 对 B 无意义，A 和 B 也不允许重排序。

* 无法保证原子性。

  

##### 定义（可见性）

Java 编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致地更新，线程应该**确保通过排他锁单独获得这个变量**。

即：

* 如果有线程在写 volatile 变量，那么其他线程要读或者写的话，要阻塞等待其写完；

* 并且在其写完之后，其他线程在下次使用该变量时（即读取时）能保证立马看到这个更新，这就是所谓的线程可见性。



##### 实现（可见性）

Lock 前缀指令

* 锁总线，其它CPU对内存的读写请求都会被阻塞，直到锁释放，不过实际后来的处理器都采用锁缓存替代锁总线，因为锁总线的开销比较大，锁总线期间其他CPU没法访问内存

* lock后的写操作会回写已修改的数据，同时让其它CPU相关缓存行失效，从而重新从主存中加载最新的数据

* 不是内存屏障却能完成类似内存屏障的功能，阻止屏障两遍的指令重排序



CPU 缓存一致性保证（非 volatile 实现，层面不同）

* 嗅探（ 一致性协议消息传递基础）

  * 它的基本思想是：缓存本身是独立的，但是内存是共享资源，所以所有内存的传输都发生在一条共享的总线上，而所有的处理器都能看到这条总线，CPU缓存不仅仅在做内存传输的时候才与总线打交道，而是不停在嗅探总线上发生的数据交换，跟踪其他缓存在做什么。
  * 只要某个处理器一写内存，其它处理器马上知道这块内存在它们的缓存段中已失效。
  * 当一个缓存代表它所属的处理器去读写内存时，其它处理器都会得到通知，它们以此来使自己的缓存保持同步。

* MSEI 协议。每个缓存行有4个状态，可用2个bit表示


  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015093126821.png#pic_center)


  写数据：

  * 开始修改某块内存之前，只有当缓存行处于 E 或者 M 状态时，处理器才能去写它，即写时处理器是独占这个缓存行的。只有在获得独占权后，处理器才能开始修改数据，因为这个缓存行只有一份拷贝，并且是在自己的缓存里，所以不会有任何冲突。
  * 但如果处理器想写某个缓存行时，如果它没有独占权，它必须先发送一条”我要独占权”的请求给总线，这会通知其它处理器把它们拥有的同一缓存段的拷贝失效（如果有）。

  读数据：

  * 如果有其它处理器想读取这个缓存行（马上能知道，因为一直在嗅探总线），独占或已修改的缓存行必须先回到 ” 共享 ” 状态。如果是已修改的缓存行，那么还要先把内容回写到内存中。









##### 重排序

在执行程序时，为了提高性能，处理器和编译器常常会对指令进行重排序，但是不能随意重排序，需要满足以下两个**条件**：

* 在**单线程**环境下，不能改变程序运行的**结果**。

* 存在**数据依赖**关系的情况下，不允许重排序。



as-if-serial 语义

* 所有的操作均可以为了优化而被重排序，但是你必须要保证重排序后执行的结果不能被改变，编译器、runtime、处理器都必须遵守 as-if-serial 语义。
* 注意，as-if-serial 只保证单线程环境，多线程环境下无效。



和 happens-before 规则联系

* A happens-before B 不是 A 一定会在 B 之前执行，而是 A 的**执行结果**对 B 可见
* 但如果在一个线程中，A 的执行结果**不需要**对 B 可见，且他们重排序后不会影响结果，所以 JMM 认为这种重排序合法。 
* **JMM 是为了在不改变程序执行结果的前提下，尽可能提高程序的运行效率**。



重排序导致的异常先发生

* 代码

  ```java
  public static void main(String[] args){
      int a = 1;
      int b = 2;
      try {
          a = 3;           // A
          b = 1 / 0;       // B
      } catch (Exception e) {        
      } finally {     
          System.out.println("a = " + a);    
      }
  }
  ```

* 出现的问题

  按照重排序的规则，操作 A 与操作 B 有可能会进行重排序，如果重排序了，B 会抛出异常（ / by zero），此时A语句一定会执行不到，那么 `a` 还会等于 3 么？如果按照 as-if-serial 原则它就改变了程序的结果。

* 处理方式

  为了保证 as-if-serial 语义，Java 异常处理机制对重排序做了**一种特殊的处理**：JIT 在重排序时，会在`catch` 语句中插入错误代偿代码（`a = 3`），这样做虽然会导致 `catch` 里面的逻辑变得复杂，但是 JIT 优化原则是：**尽可能地优化程序正常运行下的逻辑，哪怕以 `catch` 块逻辑变得复杂为代价**。



多线程的重排序问题

* 代码如下。

  问：A 线程**先**执行 `#writer()`，线程 B **后**执行 `#read()`，线程 B 在执行时能否读到 `a = 1` 呢？

  ```java
  public class Example {
      
      int a = 0;
      boolean flag = false;
  
      // A线程执行
      public void writer() {
          a = 1;                  // 1
          flag = true;            // 2
      }
  
      // B线程执行
      public void read(){
          if (flag) {            // 3
             int i = a + a;      // 4
          }
      }
  }
  ```



* 答案：不一定（**注：x86 CPU 不支持写写重排序，如果是在 x86 上面操作，这个一定会是 `a = 1` **）。

  * 由于操作 1 和操作 2 之间没有数据依赖性，所以可以进行重排序处理。
  * 操作 3 和操作 4 之间也没有数据依赖性，他们亦可以进行重排序，但是操作 3 和操作 4 之间存在**控制依赖性**，只有操作 3 成立操作 4 才会执行。
    * **当代码中存在控制依赖性时，会影响指令序列的执行的并行度，所以编译器和处理器会采用猜测执行来克服控制依赖对并行度的影响**。假如操作 3 和操作 4 重排序了，操作 4 先执行，则**先**会把计算结果**临时保存**到重排序缓冲中，当操作 3 为真时，才会将计算结果写入变量 `i` 中。

* 结论

  **重排序不会影响单线程环境的执行结果，但是会破坏多线程的执行语义**



##### 有序性

示例

* 代码

  问：A 线程**先**执行 `#writer()`，线程 B **后**执行 `#read()`，线程 B 在执行时能否读到 `a = 1` 呢？

  ```java
  public class VolatileExample {
      
      int a = 0;
      volatile boolean flag = false;
  
      // A线程执行
      public void writer() {
          a = 1;                  // 1
          flag = true;            // 2
      }
  
      // B线程执行
      public void read(){
          if (flag) {            // 3
             int i = a + a;      // 4
          }
      }
  }
  ```

* 答案：b 一定可以读到 a = 1

* 由 依据 happens-before 原则分析

  - 程序顺序原则：操作 1 happens-before 操作 2 ，操作 3 happens-before 操作 4 。
  - `volatile` 原则：操作 2 happens-before 操作 3 。
  - 传递性原则：操作 1 happens-before 操作 4 。



##### 实现（有序性）

* 限制重排序，即 lock 前缀指令也有限制重排序的功能



volatile 的重排序规则如下： 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015092505100.png#pic_center)


* 如果第一个操作为 `volatile` 读，则不管第二个操作是啥，都不能重排序。这个操作确保`volatile` 读**之后**的操作，**不会**被编译器重排序到 `volatile` 读之前；
* 如果第二个操作为 `volatile` 写，则不管第一个操作是啥，都不能重排序。这个操作确保`volatile` 写**之前**的操作，不会被编译器重排序到 `volatile` 写之后；
* 当第一个操作 `volatile` 写，第二个操作为 `volatile` 读时，不能重排序。



内存屏障

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015092529752.png#pic_center)


* 说明：

  * 在每一个 `volatile` 读操作后面，插入一个 **LoadLoad** 屏障， 禁止处理器把上面的该`volatile`读，与下面的所有读操作重排序。
  * 在每一个 `volatile` 读操作后面，插入一个 **LoadStore** 屏障，禁止处理器把上面的该 `volatile`读，与下面的所有写重排序。
  * 在每一个 `volatile` 写操作前面，插入一个 **StoreStore** 屏障，确保在该 `volatile` 写之前，其前面的所有写操作，都已经刷新到主内存中。
  * 在每一个 `volatile` 写操作后面，插入一个 **StoreLoad** 屏障， 确保该 `volatile` 写及之前写缓冲都已刷新到主内存，并且不会与后面任意操作重排序。

* 解释 

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015092549138.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  * 注意：

    StoreLoad Barriers 是一个“全能型”的屏障，它同时具有其他3个屏障的效果。现代的多处理器大多支持该屏障（其他类型的屏障不一定被所有处理器支持）。执行该屏障开销会很昂贵，因为当前处理器通常要**把写缓冲区中的数据全部刷新到内存中（Buffer Fully Flush）**。



内存屏障优化

* 代码

  ```java
  public class VolatileBarrierExample {
      int a = 0;
      volatile int v1 = 1;
      volatile int v2 = 2;
  
      void readAndWrite(){
          int i = v1;     //volatile读
          int j = v2;     //volatile读
          a = i + j;      //普通读
          v1 = i + 1;     //volatile写
          v2 = j * 2;     //volatile写
      }
  }
  ```




示例图
* ​                  没有优化的示例图    --->                                                                                                                                                     			优化后示例图

  <img src = 'https://img-blog.csdnimg.cn/20201015092612618.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center' align = 'left' />
   <img src = 'https://img-blog.csdnimg.cn/20201015092640483.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center'/>
   





优化步骤
  - 2：禁止下面所有写与上面的 volatile 读重排序，但是由于存在第二个  volatile 读，所以读根本无法越过第二个  volatile 读。可以省略。
  - 3：下面已经不存在读了，可以省略。
  - 6：下面跟着一个 `volatile` 写，可以省略
  - 总结：优化掉重复的或者不存在被预防操作的。



  

  



​	



























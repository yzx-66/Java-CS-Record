##### G1收集器

特点

* 可以面向堆内存任何部分来组成回收集（Collection Set，一般简称CSet）进行回收，衡量标准不再是它属于哪个分代，而是哪块内存中存放的垃圾数量最多，回收收益最大，这就是G1收集器的Mixed GC模式。
* 能够建立起“停顿时间模型”（Pause Prediction Model）的收集器，停顿时间模型的意思是能够支持指定在一个长度为M毫秒的时间片段内，消耗在垃圾收集上的时间大概率不超过N毫秒这样的目标，以后会完全取代 CMS（CMS 在 JDK 9 已经过时）。



可预测的停顿时间模型

* G1收集器之所以能建立可预测的停顿时间模型，是因为它将Region作为单次回收的最小单元，即每次收集到的内存空间都是Region大小的整数倍，这样可以有计划地避免在整个Java堆中进行全区域的垃圾收集。
* G1 收集器的停顿预测模型是以衰减均值（Decaying Average）为理论基础来实现的，在垃圾收集过程中，G1收集器会记录每个Region的回收耗时、每个Region记忆集里的脏卡数量等各个可测量的步骤花费的成本，并分析得出平均值、标准偏差、置信度等统计信息。然后在后台维护一个优先级列表，根据用户设定允许的收集停顿时间（使用参数-XX：MaxGCPauseMillis指定，默认值是200毫秒），预测现在开始回收的话，由哪些Region组成回收集，才可以在不超过期望停顿时间的约束下获得最高的收益。
  * 这里强调的“衰减平均值”是指它会比普通的平均值更容易受到新数据的影响，平均值代表整体平均状态，但衰减平均值更准确地代表“最近的”平均状态。
* 所以综上，这种使用Region划分内存空间，以及具有优先级的区域回收方式，保证了G1收集器在有限的时间内获取尽可能高的收集效率。





![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026144419567.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


Region划分策略

* G1不再坚持固定大小以及固定数量的分代区域划分，而是把连续的Java堆划分为多个大小相等的独立区域（Region），每一个Region都可以根据需要，扮演新生代的Eden空间、Survivor空间，或者老年代空间。收集器能够对扮演不同角色的Region采用不同的策略去处理，这样无论是新创建的对象还是已经存活了一段时间、熬过多次收集的旧对象都能获取很好的收集效果
* Region中还有一类特殊的Humongous区域，专门用来存储大对象。G1认为只要大小超过了一个Region容量一半的对象即可判定为大对象。
  * 每个Region的大小可以通过参数-XX：G1HeapRegionSize设定，取值范围为1MB～32MB，且应为2的N次幂。G1的大多数行为都把Humongous Region作为老年代的一部分来进行看待。
  * 巨型对象会会被存放在N个连续的Humongous Region之中。巨型对象无法利用年轻代里的TLAB（对象分配本地缓冲）和PLAB（分代晋升本地缓冲）。在JDK 8u40之前，它只能在并发收集周期的清除阶段回收，但是在JDK 8u40之后，巨型分区可以在年轻代收集中和full GC被回收
* Region 还有一个缺陷，经过实验发现，每个region实际上是放不满的，假设我设置堆有 32 M，一个 region 4 M，那么 oom 时确实有 8 个region，但内存才占用 23 m，所以有堆内存浪费。





![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026144448805.png#pic_center)


宏观上 mixed gc 回收步骤

* 初始标记（Initial Marking）：仅仅只是标记一下GC Roots能直接关联到的对象，并且修改TAMS指针的值，让下一阶段用户线程并发运行时，能正确地在可用的Region中分配新对象。这个阶段需要停顿线程，但耗时很短，而且是借用进行Minor GC的时候同步完成的，所以G1收集器在这个阶段实际并没有额外的停顿。
* 并发标记（Concurrent Marking）：从GC Root开始对堆中对象进行可达性分析，递归扫描整个堆里的对象图，找出要回收的对象，这阶段耗时较长，但可与用户程序并发执行。当对象图扫描完成以后，还要重新处理SATB记录下的在并发时有引用变动的对象。
* 最终标记（Final Marking）：对用户线程做另一个短暂的暂停，用于处理并发阶段结束后仍遗留下来的最后那少量的SATB记录。
* 筛选回收（Live Data Counting and Evacuation）：负责更新Region的统计数据，对各个Region的回收价值和成本进行排序，根据用户所期望的停顿时间来制定回收计划，可以自由选择任意多个Region构成回收集，然后把决定回收的那一部分Region的存活对象复制到空的Region中，再清理掉整个旧Region的全部空间。这里的操作涉及存活对象的移动，是必须暂停用户线程，由多条收集器线程并行完成的。







G1的垃圾回收（JDK 11)

G1垃圾收集活动时序图（主要有4种类型：年轻代收集周期、多级并发标记周期、混合收集周期和full GC（转移失败的安全保护机制）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026144504197.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




* 年轻代收集（即Pause young(Normal)，回收eden region + survivor region，因为新对象肯定先经过新生代，且回收收益肯定大于老年代）

  * 触发条件：应用刚启动，慢慢流量进来，开始生成对象。G1会选一个分区并指定他为eden分区，当这块分区用满了之后，G1会选一个新的分区作为eden分区，这个操作会一直进行下去直到达到新生代上限，那么会触发一次年轻代收集。

  * Young GC是并行和整个阶段都是 stop-the-world的，因为新生代存活对象很少，所以很快。其要经历 Root scanning、Update RS、Scan RS 、Object copy ，other 里还有一些其他操作，比如根据停顿时间来 Choose CSet ，总之是将活着的对象复制到Survivor区，或晋升至Old区(达到晋升年龄)

  * 调优：我们可以通过--XX:MaxGCPauseMillis，调优年轻代收集，缩小暂停时间，或者 -XX:G1MaxNewSizePercent 设置小一点，来减少新生代回收时间

    | **参数**                 | **含义**                                               |
    | ------------------------ | ------------------------------------------------------ |
    | -XX:NewRatio             | 老年代/年轻代，默认值 2，即 1/3 的年轻代，2/3 的老年代 |
    | -XX:TargetSurvivorRatio  | survivor占比，默认值 8                                 |
    | -XX:MaxTenuringThreshold | 晋升年龄，默认15                                       |
    | **-XX:MaxGCPauseMillis** | 设置G1收集过程目标时间，默认值200ms                    |
    | -XX:G1NewSizePercent     | 新生代（eden + survivor）最小值，默认值5%              |
    | -XX:G1MaxNewSizePercent  | 新生代（eden + survivor）最大值，默认值60%             |

　　

* 并发标记周期（即 Pause young(Concurrent Start)，回收完全存活对象的region，主要作用是全堆的并发标记）

  * 触发条件：同年轻代收集，都是 eden 区放不下了，但是老年代大于IHOP参数，那么G1首先会触发并发标记周期（上图的Concurrent Marking Cycle）

  * 每次触发GC(n) Pause young(Concurrent Start) ，之后都会跟一个GC(n+1)  Concurrent Cycle

    * Pause young(Concurrent Start)  

      * 和 Pause young(Normal) 一样，都是执行新生代的回收，从 Pause 可以看出来是 stop word 的。

    * Concurrent Cycle 是并发标记

      * Concurrent Cycle 是并发的，如果还没结束但出现了堆中剩余空间不足以分配，那么会执行立即中断并发标记，然后执行full gc，等 full gc 执行完了才能继续，不过一般这种情况并发继续时也都显示 abort 然后该次 gc 就结束了。

      * 五个阶段：

        第一阶段：初始标记（上图Young Collection with Initial Mark），收集所有GC根（对象的起源指针，根引用），STW，在年轻代完成

        第二阶段：根区间扫描，标记所有幸存者区间的对象引用

        第三阶段：并发标记（上图Concurrent Marking），标记存活对象

        第四阶段：重新标记（上图Remark），是最后一个标记阶段，STW，很短，完成所有标记工作

        第五阶段：清除（上图Clean），回收没有存活对象的Region并加入可用Region队列

  * 后续阶段：并发标记结束后，G1 也就知道了哪些区块是最适合被回收的，那些完全空闲的区块会在这这个阶段被回收。

    * 如果这个阶段释放了足够的内存出来，其实也就可以认为结束了一次 GC，下一次还是会先回收年轻代。
    * 否则进入混合回收，会利用这次并发标记的结果，选择一些活跃度最低的老年代区块进行回收。（相当于 mixed 的并发标记阶段）

  * 调优：

    * 可以通过--XX:InitiatingHeapOccupancyPercent，配置适合应用的IHOP值（过大会可能转移失败，过小可能过早引起并发标记周期）
    * 也可以通过--XX:ConcGCThreads，增加并发线程数

    | 参数                                   | 含义                                                         |
    | -------------------------------------- | ------------------------------------------------------------ |
    | **-XX:InitiatingHeapOccupancyPercent** | IHOP值，当老年代（old+humongous）大小占整个堆大小百分比达到该阈值会触发，默认 45 |
    | -XX:ConcGCThreads                      | 并发标记阶段的垃圾收集线程数，增加这个值可以让并发标记更快完成，如果没有指定这个值，JVM 会通过以下公式计算得到：ConcGCThreads=(ParallelGCThreads + 2) / 4^3 |
    | -XX:ParallelGCThreads                  | 并行收集时候的垃圾收集线程数                                 |

  

* 混合收集周期（即 Pause young(Mixed)，回收 eden region + survivor region + 部分 old region）

  * 触发条件：当达到 IHOP 参数并完成上并发标记周期之后，如果还是大于 IHOP 混合收集周期就启动了。该过程也是 STW 的，混合收集和年轻代收集是类似的，唯一区别就是在混合收集过程中会包含一部分老年分区，所以也叫混合收集。

  * 看上图的Mixed Collection Cycle，中间有好几段Mixed Collection，说明混合收集周期包含多次收集次数。有两个参数比较重要：

    * -XX:G1MixedGCCountTarget：缺省值为8，意思是能-XX:G1MixedGCCountTarget。G1根据将回收的老年分区除以该参数值得到每次混合收集的老年代CSet最小数量
    * -XX:G1HeapWastePercent：缺省值为5%，每次混合收集暂停，G1算出废物百分比，根据堆废物百分比，当收集达到参数时，不再启动新的混合收集

  * 调优：当暂停时间和运行时间呈现指数级增长，可以通过-XX:G1HeapWastePercent，调高该参数会有所帮助，但这也导致更多碎片化

    | 参数                     | 含义                                                   |
    | ------------------------ | ------------------------------------------------------ |
    | -XX:G1MixedGCCountTarget | 混合收集时，除以该参数得每次老年代CSet最小数量，默认 8 |
    | -XX:G1HeapWastePercent   | 混合收集时允许的剩余垃圾占比，默认 5%                  |

  * 注意：

    * 该 gc 是 stop the world 的，因为并发标记已经在 young(concurrent marking) 已经执行过了。

      ```java
      // mixed gc 模拟代码
      public static void main(String[] args) throws InterruptedException {
          byte[][] bytes = new byte[30][];
      
           for(int i = 0 ;i < 30 ;i++){
               bytes[i] = new byte[1024*1024];
               System.out.println(i);
               Thread.sleep(1000);
          }
      }
      ```

  

* full GC（即全堆整理回收）

  * full GC中，**单个线程**会对整个堆的所有代中所有分区做标记、清除以及压缩动作，是单线程执行的serial old gc，非常昂贵！

    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026144521431.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  * 有2个条件同时满足则会触发full GC

    * 拷贝存活对象晋升（promotion）失败，无法找到可用的空闲分区，GC日志记录为 to-space exhausted。或者分配巨型对象无法在老年代找到连续足够的分区（对象内存分配速度过快，mixed gc来不及回收，导致老年代被填满）

     ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026144549141.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


    

    

    

    

    

    

    

    

    

    * 当发生第一个条件后，G1会尝试增加堆使用量，如果扩展失败，那么会触发安全措施机制同时发生full GC

    

​		

与 CMS 部分细节比较

* 回收算法
  * CMS是“标记-清除”算法
  * G1从整体来看是基于“标记-整理”算法实现的收集器，但从局部（两个Region之间）上看又是基于“标记-复制”算法实现，无论如何，这两种算法都意味着G1运作期间不会产生内存空间碎片，垃圾收集完成之后能提供规整的可用内存。

* 并发的标记的处理
  * 处理方案：G1收集器是通过原始快照（SATB）算法来实现的。但与 CMS 在并发标记时可能产生的“Concurrent Mode Failure”类似，如果内存回收的速度赶不上内存分配的速度，G1收集器也要被迫冻结用户线程执行，导致Full GC而产生长时间“Stop The World”。
  * 并发过程新创建对象内存分配：
    * 指 G1 的并发标记阶段（其余三个阶段都是 Stop the world的），G1为每一个Region设计了两个名为TAMS（Top at Mark Start）的指针，把Region中的一部分空间划分出来用于并发回收过程中的新对象分配，并发回收时新分配的对象地址都必须要在这两个指针位置以上。G1收集器默认在这个地址以上的对象是被隐式标记过的，即默认它们是存活的，不纳入回收范围。
    * 指 CMS 的并发标记和并发清除阶段，并发标记阶段，因为采用增量更新，所以可以增加新的节点，并发清除阶段，该阶段产生的垃圾不被标记，称为浮动垃圾，要等下一次回收时才能收集。

  * 性能对比：相比起增量更新算法，原始快照搜索能够减少并发标记和重新标记阶段的消耗，避免CMS那样在最终标记阶段停顿时间过长的缺点，但是在用户程序运行过程中确实会产生由跟踪引用变化带来的额外负担。

* 跨代引用的记忆集
  * 实现方式，G1收集器上记忆集的应用其实要复杂很多，每个Region都有一个RSet，RSet记录了其他Region中的对象引用本Region中对象的关系，同时每个 region 也都有一份Card Table，通过卡表来进行内存的划分（一般为512Bytes）。所以每个Region会记录下别的Region有指向自己的指针，并标记这些指针分别在哪些Card的范围内。即 RSet是一个Hash Table，Key是别的Region的起始地址，Value是一个集合，里面的元素是Card Table的Index。
  * 内存占用，每个Region，无论扮演的是新生代还是老年代角色，都必须有一份自己的记忆集，这导致G1的记忆集（和其他内存消耗）可能会占整个堆容量的20%乃至更多的内存空间；相比起来CMS的卡表就相当简单，只有唯一一份，而且只需要处理老年代到新生代的引用，反过来则不需要，由于新生代的对象具有朝生夕灭的不稳定性，引用变化频繁，能省下这个区域的维护开销是很划算的（代价就是当CMS发生Old GC时，要把整个新生代作为GC Roots来进行扫描）。
  * 执行负载，它们都使用到写屏障，CMS用写后屏障来更新维护卡表；而G1除了使用写后屏障来进行同样的（由于G1的卡表结构复杂，其实是更烦琐的）卡表维护操作外，为了实现原始快照搜索（SATB）算法，还需要使用写前屏障来跟踪并发时的指针变化情况。

* 总结：

  * 相比CMS，G1的优点有很多，暂且不论可以指定最大停顿时间、分Region的内存布局、按收益动态确定回收集这些创新性设计带来的红利，单从最传统的算法理论上看，G1也更有发展潜力。
  * 不过，G1相对于CMS仍然不是占全方位、压倒性优势的，从它出现几年仍不能在所有应用场景中代替CMS就可以得知这个结论，如在用户程序运行过程
    中，G1无论是为了垃圾收集产生的内存占用（Footprint）还是程序运行时的额外执行负载（Overload）都要比CMS要高。

  * 按照实践经验，目前在小内存应用上CMS的表现大概率仍然要会优于G1，而在大内存应用上G1则大多能发挥其优势，这个优劣势的Java堆容量平衡点通常在6GB至8GB之间，不过随着HotSpot的开发者对G1的不断优化，也会让对比结果继续向G1倾斜。

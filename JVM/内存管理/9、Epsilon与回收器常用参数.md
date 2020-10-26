#### 无操作

##### Epsilon收集器

自动内存管理子系统

* Epsilon，是一款以不能够进行垃圾收集为“卖点”的垃圾收集器，被形容成一个无操作的收集器（A No-Op Garbage Collector）。而事实上只要Java虚拟机能够工作，垃圾收集器便不可能是真正“无操作”的。
* 原因是“垃圾收集器”这个名字并不能形容它全部的职责，更贴切的名字应该是“自动内存管理子系统”。一个垃圾收集器除了垃圾收集这个本职工作之外，它还要负责堆的管理与布局、对象的分配、与解释器的协作、与编译器的协作、与监控子系统协作等职责，其中至少堆的管理和对象的分配这部分功能是Java虚拟机能够正常运作的必要支持，是一个最小化功能的垃圾收集器也必须实现的内容。



应用场景

* 从JDK 10开始，为了隔离垃圾收集器与Java虚拟机解释、编译、监控等子系统的关系，RedHat提出了垃圾收集器的统一接口，即JEP 304提案，Epsilon是这个接口的有效性验证和参考实现，同时也用于需要剥离垃圾收集器影响的性能测试和压力测试。
* 传统Java有着内存占用较大，在容器中启动时间长，即时编译需要缓慢优化等特点，这对大型应用来说并不是什么太大的问题，但对短时间、小规模的服务形式就有诸多不适。如果应用只要运行数分钟甚至数秒，只要Java虚拟机能正确分配内存，在堆耗尽之前就会退出，那显然运行负载极小、没有任何回收行为的Epsilon便是很恰当的选择。





#### 参数

##### 分代通用参数

* ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026150044150.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  补充说明

  * -XX:+UseAdaptiveSizePolicy：如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到-XX：MaxTenuringThreshold中要求的年龄。
  * -XX:+HandlePromotionFailure：在发生Minor GC之前，虚拟机必须先检查老年代最大可用的连续空间是否大于新生代所有对象总空间
    * 如果这个条件成立，那这一次Minor GC可以确保是安全的。
    * 如果不成立，则虚拟机会先查看-XX：HandlePromotionFailure参数的设置值是否允许担保失败（Handle Promotion Failure）
      * 如果允许，那会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试进行一次Minor GC，尽管这次Minor GC是有风险的；
      * 如果不允许冒险，或者老年代剩余空间小于历次晋升到老年代对象的平均大小，那这时就要改为进行一次Full GC。



##### 不通用参数

parallel

* ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026150102676.png#pic_center)




cms

* ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026150118886.png#pic_center)




g1

* ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026150133555.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




shenandoah

* ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026150151534.png#pic_center)




zgc

* ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026150208691.png#pic_center)



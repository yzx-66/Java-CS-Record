#### 分析工具

##### 进程查询工具

###### jps

jps 主要用来输出JVM中运行的进程状态信息

```jps [options] [hostid]```

* options
  * -q：不输出类名、Jar名 和传入main方法的参数

  * -m：输出传入main方法的参数

  * -l：输出main类或Jar的全限名

  * -v：输出传入JVM的参数
* hostid
* 如果不指定hostid就默认为当前主机或服务器。



##### 参数查改工具

###### jinfo

jinfo 的作用是实时查看和调整虚拟机各项参数

```jinfo [ option ] pid```

* option
  * flag \<name>  : 打印 name 参数的值
  * -flag [+|-]\<name>：开启或者关闭 name 参数 
  * -flag \<name>=\<value> 把 name 的 参数值 设置为 value
  * -flags ： 打印所有参数
  * -sysprops ： 打印 java 的 system 设置的属性值
  * \<no option> ：打印所有 vm 参数和 System 属性值



##### 监视工具

###### jstat

jstat 用来实时监测系统堆的使用情况

```jstat [ general0ption| outputOptions vmid [ interval[s |ms] [ count]] ]```

* general0ption 根据jstat统计的维度不同，可以使用如下表中的选项进行不同维度的统计，不同的操作系统支持的选项可能会不一样，可以通过-options选项，查看不同操作系统所支持选项。

  * -gc					   用于查看JVM中堆的垃圾收集情况的统计
  * -gccapacity	     用于查看新生代、老生代及持久代的存储容量情况
  * -gcnew 			   用于查看新生代垃圾收集的情况
  * -gcnewcapacity		用于查看新生代的存储容量情况
  * -gcold 				        用于查看老生代及持久代发生GC的情况
  * -gcoldcapacity		  用于查看老生代的容量
  * -gcpermcapacity	  用于查看持久代的容量
  * -gcutil				         用于查看新生代、老生代及持代垃圾收集的情况
  * -gccause			 用于查看垃圾收集的统计情况(这个和- gcutil选项一样)，如果有发生垃圾收集，它还会显示最后一次及当前正在发生垃圾收集的原因。
  * -class	    		   用于查看类加载情况的统计
  * -compiler			用于查看HotSpot中即时编译器编译情况的统计
  * -printcompilation	 用于查看HotSpot编译方法的统计

  

* outputOptions

  * -h：用于指定每隔几行就输出列头，如果不指定，默认是只在第行出现列头。 
  * -J：用于将给定的javaOption传给java应用程序加载器，例如，“-J-Xms48m"将把启动内存设置为48M。
  * -t：用于在输出内容的第列显示时间戳，这个时间戳代表的时JVM开始启动到现在的时间

  

* vmid 是虚拟机ID,在LinuxUnix系统 上一般就是进程ID。

* interval 是采样时间**间隔毫秒ms**。

* count 是采样数目。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026145620815.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




##### 堆栈dump工具

###### jmap

jmap 命令用于生成堆转储快照（如果不使用这个命令，还可以使用-XX:+HeapDumpOnOutOfMemoryError参数来让虚拟机出现OOM的时候自动生成dump文件）。

```jmap [ option ] vmid```

* option（基于 jdk 11，只有下面四个）

  * -dump：生成Java堆转储快照。格式为-dump:[live,]format=b,file=\<filename>，其中 live子参数说明是否只 dump出存活的对象。必须指定文件
  * -histo：显示准中对象统计信息，包括类、实例数量、合计容量。和 -dump 的区别是不能dump到文件，并且只显示堆上对象信息。
  * -finalizerinfo：显示在F-Queue中等待Finalizer线程执行finalize方法的对象。
  * -clstats：类加载器的统计情况

* 使用 VisualVM 查看 -dump 的文件

  ![](https://img-blog.csdnimg.cn/20201026145656920.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  

###### jstack

jstack（Stack Trace for Java）命令用于生成虚拟机当前时刻的线程快照

```jstack [ option ] vmid```

* option
  * -l：long listings, 会打印出额外的锁信息，在发生死锁时可以用jstack -l pid来观察锁持有情况
  * -m：mixed mode, 不仅会输出Java堆栈信息，如果调用本地方法，还会输出C/C++ 堆栈信息(比如Native方法)



#### 六种OOM发生及解决办法


##### StackOverFlowError

java.lang.StackOverFlowError

* 导致原因
  * 栈帧太多，虚拟机栈没有地方容纳了

* 解决办法
  * 把虚拟机栈变大：```-Xss1024k```



##### heap space

java.lang.OutOfMemoryError:Java heap space

* 导致原因
  * 间接原因：存在一种可能是因为内存泄漏（即对象使用完毕后，没有及时释放其占用的内存空间）
  * 直接原因：full gc 后，堆上仍然没有空间用来让 young gen 晋升 或者 没有地方给大对象直接分配（分代的 -XX:PretenureSizeThreshold 或 G1 的 Humongous  region）。

* 解决办法
  * 一般都会加上参数 -XX:+HeapDumpOnOutOfMemoryError 当JVM发生OOM时，自动生成DUMP文件（默认为：java\_\<pid>.hprof，也可以指定文件名称 -XX:HeapDumpPath=${目录}参数表示生成DUMP文件的路径），然后按照 size 进行排序，查看哪个对象最占用堆内存
  * 把堆内存变大：```-Xmx2g -Xms2g```



##### Metaspace

java.langOutOfMemoryError:Metaspace

* 导致原因
  * JDK 8 之后，元空间只存放类信息和一些字面量常量，如果达到 MaxMetaspaceSize 就触发 full gc，但是动态代理加载的类太多了， Full gc 时类卸载条件苛刻，所以 oom

* 解决办法
  * 把元空间内存变大：```-XX:MetaSpaceSize=256m  -XX:MaxMetaSpaceSize=512m```
  * 有必要的话，因为类卸载条件比较苛刻，所以给生成大量动态代理的地方，都是用自定义类加载器，比如 Tomcat 就为每个 jsp 都有一个类加载器。





##### Direct buffer

java.lang.OutOfMemoryError:Direct buffer memory

java.lang.OutOfMemoryError:null

* 导致原因：

  * 写NIO程序经常使用ByteBuffer来读取或者写入数据，这是一种基于通道（Channel）与缓冲区（Buffer）的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在java堆和Native堆中来回复制数据。
  * ByteBuffer.allocate(capability)第一种方式是分配JVM堆内存，属于GC管辖范围，由于需要拷贝所以速度相对较慢。
  * ByteBuffer.allocateDirect(capability)第一种方式是分配OS本地内存，不属于GC管辖范围，由于不需要内存拷贝，所以速度相对较快
  * 但如果不断分配内存，堆内存很少使用，那么JVM就不需要执行GC，DirectByteBuffer对象们就不会被回收，这时候堆内存充足，但本地内存可能已经使用光了，再次尝试分配本地内存就会出现OutOfMemoryError，那程序就直接崩溃了。

* 解决办法

  * 因为 direct buffer 必须要等 full gc 时才能回收，所以通过 XX:MaxDirectMemorySize 加大直接内存

  * 有必要的话 -XX:-DisableExplicitGC 关闭不能调用 System.gc ，在使用了大量直接内存后手动 System.gc。

    



##### native thread

java.lang.OutOfMemoryError:unable to create new native thread

* 该native thread异常与对应的平台有关

* 高并发请求服务器时，经常出现如下异常：java.lang.OutOfMemoryError:unbale to create new native thread

* 导致原因：

  * 应用创建了太对线程，一个应用进程创建多个线程，超过系统承载极限。
  * 服务器并不允许应用程序创建那么多线程，linux系统默认允许单个进程可以创建的线程数是1024个，如果应用创建超过这个数量，就会报java.lang.OutOfMemoryError:unable to create new native thread

* 解决办法：

  * 想办法降低应用程序创建线程的数量，分析应用是否真的需要创建那么多线程，如果不是，改代码将线程数降到最低。
  * 对于有的应用，确实需要创建多个线程，远超过linux系统默认的1024个线程的限制，可以通过修改linux服务器配置，扩大linux默认限制。

  

##### overhead limit

java.lang.OutOfMemoryError:GC overhead limit exceeded

* GC回收时间过长时会抛出OutOfMemoryError。过长的定义是，超过98%的时间用来做GC并且回收了不到2%的堆内存，连续多次GC都只回收了不到2%的极端情况下才会抛出。
* 假设不抛出GC overhead limit错误，会发生GC清理的内存很快会再次填满，迫使GC再次执行，这样就形成恶性循环，CPU使用率一直是100%，而GC缺没有任何成果。
* 解决办法:
  * 其实这是个预警，可以关闭verheadLimit， -XX:-UseGCOverheadLimit。但极其不推荐，只是推迟了 oom 的发生时间

# Java-CS-Record
这是一个**频繁更新**的项目，主要原因是目前没有对 Java 开发知识点系统梳理的仓库，大多都是对某些技术点的面试题，但是没有解说部分，所以这个仓库会对 Java 开发所需知识点进行梳理与讲解。

文章说明
* 这个仓库的文章，都是**关于计算机基础，还有 Java 后台相关原理源码，几乎不涉及怎么调用 api**。
* 每次会把一个技术点整理完才进行更新，很多技术体系太庞大，比如某些框架源码，我会只挑选关键部分整理。
  
  
<a href ='https://yzx66.blog.csdn.net'>博客同步更新</a>，还放到 Github 有如下原因
* 博客是平铺式结构，无法按照目录式结构保存。
* 也为了更好帮助想要使用或者进行改动的同学，所以把所有 markdown 也在这里开源。


最后，**欢迎 star 该项目，也欢迎使用、修改、与提出意见**，希望多多支持！

<p align="center">
  <img src="https://img.shields.io/badge/Java-%E5%90%8E%E5%8F%B0%E5%BC%80%E5%8F%91-yellowgreen">
  <img src="https://img.shields.io/badge/Content-%E5%8E%9F%E7%90%86%E6%BA%90%E7%A0%81-brightgreen">
  <img src="https://img.shields.io/badge/CS-%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80-brightgreen"> 
  <a href="https://blog.csdn.net/weixin_43934607"><img src="https://img.shields.io/badge/Blog-%E5%90%8C%E6%AD%A5%E6%9B%B4%E6%96%B0-orange"></a>
</p>

<h1 align="center">目录</h1>

# 基础
## 1、Java 核心
### 常见特性源码
类型相关
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%BF%85%E5%A4%87%E7%9F%A5%E8%AF%86/%E5%9F%BA%E6%9C%AC%E7%B1%BB%E5%9E%8B%E4%B8%8E%E5%8C%85%E8%A3%85%E7%B1%BB%E6%A2%B3%E7%90%86%E5%8F%8A%E6%BA%90%E7%A0%81.md '>基本类型及包装类源码解析 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%BF%85%E5%A4%87%E7%9F%A5%E8%AF%86/String%E5%A5%97%E9%A4%90%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98%E5%8F%8A%E5%BA%95%E5%B1%82.md '>String 套餐及编译后实现 </a>
* <a href = 'https://github.com/yzx66-net/Java_Review/blob/main/%E5%BF%85%E5%A4%87%E7%9F%A5%E8%AF%86/Object%E7%B1%BB%E4%B8%8E%E7%9B%B8%E5%85%B3%E9%97%AE%E9%A2%98.md'>Object 类与相关实现</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%BF%85%E5%A4%87%E7%9F%A5%E8%AF%86/%E5%BC%82%E5%B8%B8%E4%BD%93%E7%B3%BB%E5%8F%8A%E5%A4%84%E7%90%86%E5%92%8C%E8%AE%BE%E8%AE%A1%E5%8E%9F%E5%88%99.md '>异常设计原则与继承体系 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%BF%85%E5%A4%87%E7%9F%A5%E8%AF%86/final%E5%A5%97%E9%A4%90%E5%8F%8A%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98.md '>final 套餐及常见问题 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%BF%85%E5%A4%87%E7%9F%A5%E8%AF%86/%E5%BA%8F%E5%88%97%E5%8C%96%E4%B8%8E%E5%85%B6%E4%BB%96%E5%BA%8F%E5%88%97%E5%8C%96%E6%A1%86%E6%9E%B6.md'> 序列化原理及技术实现</a>

特性相关
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%BF%85%E5%A4%87%E7%9F%A5%E8%AF%86/%E5%8F%8D%E5%B0%84%E7%9B%B8%E5%85%B3%E9%97%AE%E9%A2%98%E4%B8%8E%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'> 反射相关问题与源码解析</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%BF%85%E5%A4%87%E7%9F%A5%E8%AF%86/%E6%B3%A8%E8%A7%A3%E5%BA%95%E5%B1%82%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'>注解底层源码解析 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%BF%85%E5%A4%87%E7%9F%A5%E8%AF%86/Stream%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md '> Stream 执行流程源码解析</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%BF%85%E5%A4%87%E7%9F%A5%E8%AF%86/Stream%E5%B9%B6%E8%A1%8C%E6%B5%81%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'> Stream 并行流源码解析</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%BF%85%E5%A4%87%E7%9F%A5%E8%AF%86/%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BC%96%E7%A8%8B%E4%B8%8ELamda%E5%BA%95%E5%B1%82.md'>函数式编程与 Lamda 底层 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%BF%85%E5%A4%87%E7%9F%A5%E8%AF%86/SPI%E6%9C%BA%E5%88%B6%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'> SPI 机制源码解析</a>

### Java 集合源码
list
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E9%9B%86%E5%90%88/list/ArrayList%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'>ArrayList 源码解析 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E9%9B%86%E5%90%88/list/LinkedList%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'> LinkedList 源码解析</a>

map
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E9%9B%86%E5%90%88/map/HashMap%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3%EF%BC%88%E4%B8%8A%EF%BC%89.md'>HashMap 核心方法源码详解 </a> 
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E9%9B%86%E5%90%88/map/HashMap%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3%EF%BC%88%E4%B8%8B%EF%BC%89.md '>HashMap 次要方法源码详解 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E9%9B%86%E5%90%88/map/HashMap%E6%BA%90%E7%A0%81%E8%A1%A5%E5%85%85.md '>HashMap 知识扩充 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E9%9B%86%E5%90%88/map/LinkedHashMap%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'> LinkedHashMap 源码解析</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E9%9B%86%E5%90%88/map/TreeMap%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3%EF%BC%88%E4%B8%8A%EF%BC%89.md '>TreeMap 核心方法源码详解 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E9%9B%86%E5%90%88/map/TreeMap%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3%EF%BC%88%E4%B8%8B%EF%BC%89.md '>TreeMap 次要方法源码详解 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E9%9B%86%E5%90%88/map/TreeMap%E4%B8%AD%E5%AD%90Map%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3.md '>TreeMap 中子Map源码详解 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E9%9B%86%E5%90%88/map/HashTable%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md '> HashTable 源码解析 </a>

set
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E9%9B%86%E5%90%88/set/HashSet%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md '>HashSet 源码解析 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E9%9B%86%E5%90%88/set/TreeSet%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'> TreeSet 源码解析</a>

### Java 并发源码
基础
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/%E5%9F%BA%E7%A1%80/%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86.md'>基础知识与概念 </a>

底层
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/%E5%BA%95%E5%B1%82/JMM%20%E5%85%A8%E8%A7%A3.md'>JMM 全解 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/%E5%BA%95%E5%B1%82/Sychnorized%20%E5%BA%95%E5%B1%82%E8%AF%A6%E8%A7%A3.md '>Sychnorized 底层详解 </a>

JUC
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/CAS%20%E4%B8%8E%E5%8E%9F%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'>CAS 与原子类源码解析 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/ThreadLocal%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'>ThreadLocal 源码解析 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/Fork-Join%20%E6%A1%86%E6%9E%B6%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md '>Fork-Join 框架源码解析 </a>


* AQS
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/AQS/AQS%20%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3.md'> AQS 源码详解 </a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/AQS/AQS%20%E4%B9%8B%20Condition%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md '>AQS 之 Condition 源码解析 </a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/AQS/AQS%20%E4%B9%8B%20ReentrantLock%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'>AQS 之 ReentrantLock 源码解析 </a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/AQS/AQS%20%E4%B9%8B%20ReentrantReadWriteLock%20%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3.md'>AQS 之 ReentrantReadWriteLock 源码详解 </a>

* 并发工具
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E5%B9%B6%E5%8F%91%E5%B7%A5%E5%85%B7/Semaphore%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'>Semaphore 源码解析 </a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E5%B9%B6%E5%8F%91%E5%B7%A5%E5%85%B7/CountDownLatch%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'> CountDownLatch 源码解析 </a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E5%B9%B6%E5%8F%91%E5%B7%A5%E5%85%B7/CyclicBarrier%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'>CyclicBarrier 源码解析 </a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E5%B9%B6%E5%8F%91%E5%B7%A5%E5%85%B7/Exchanger%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'>Exchanger 源码解析 </a>
  
* 并发容器
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E5%B9%B6%E5%8F%91%E5%AE%B9%E5%99%A8/CopyOnWriteArrayList%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'>CopyOnWriteArrayList 源码解析 </a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E5%B9%B6%E5%8F%91%E5%AE%B9%E5%99%A8/ConcurrentLinkedQueue%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'> ConcurrentLinkedQueue源码解析</a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E5%B9%B6%E5%8F%91%E5%AE%B9%E5%99%A8/ConcurrentHashMap%20%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3.md'>ConcurrentHashMap 源码详解 </a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E5%B9%B6%E5%8F%91%E5%AE%B9%E5%99%A8/JDK%201.7%20Concurrenthashmap%20%E6%BA%90%E7%A0%81%E5%AF%B9%E6%AF%94.md '>JDK 1.7 Concurrenthashmap 源码对比 </a>

* 阻塞队列
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97/ArrayBlockingQueue%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'> ArrayBlockingQueue 源码解析 </a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97/LinkedBlockingDeque%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'> LinkedBlockingDeque 源码解析</a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97/PriorityBlockingQueue%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'>PriorityBlockingQueue 源码解析 </a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97/DelayQueue%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'>DelayQueue 源码解析 </a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97/SynchronousQueue%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'>SynchronousQueue 源码解析 </a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97/LinkedTransferQueue%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'> LinkedTransferQueue 源码解析 </a>
  
* 线程池
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E7%BA%BF%E7%A8%8B%E6%B1%A0/Executor%20%E4%B8%8E%20Future%20%E6%A6%82%E8%A7%88.md'>Executor体系 </a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E7%BA%BF%E7%A8%8B%E6%B1%A0/ThreadPoolExecutor%20%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3.md'> ThreadPoolExecutor 源码详解</a>
  * <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E5%B9%B6%E5%8F%91/JUC/%E7%BA%BF%E7%A8%8B%E6%B1%A0/ScheduledThreadPoolExecutor%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md'> ScheduledThreadPoolExecutor 源码解析</a>



### Java 虚拟机
内存管理
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/1%E3%80%81%E8%BF%90%E8%A1%8C%E6%97%B6%E6%95%B0%E6%8D%AE%E5%8C%BA%E5%9F%9F%E8%AF%A6%E8%A7%A3.md'> 1、运行时数据区域详解 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/2%E3%80%81%E5%AF%B9%E8%B1%A1%E8%AF%A6%E8%A7%A3.md'>2、对象详解 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/3%E3%80%81%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E5%88%A4%E5%AE%9A%E4%B8%8E%E7%AE%97%E6%B3%95.md'>3、垃圾回收判定与算法 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/4%E3%80%81HotSpot%E5%9B%9E%E6%94%B6%E7%AE%97%E6%B3%95%E7%BB%86%E8%8A%82.md'> 4、HotSpot回收算法细节</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/5%E3%80%81%E7%BB%8F%E5%85%B8%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E5%99%A8.md'>5、经典垃圾回收器 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/6%E3%80%81Region%20%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80%E5%9B%9E%E6%94%B6%E5%99%A8%20G1.md '>6、Region 内存布局回收器 G1 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/7%E3%80%81%E4%BD%8E%E5%BB%B6%E8%BF%9F%E5%9B%9E%E6%94%B6%E5%99%A8%20Shenandoah.md '> 7、低延迟回收器 Shenandoah</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/8%E3%80%81%E4%BD%8E%E5%BB%B6%E8%BF%9F%E5%9B%9E%E6%94%B6%E5%99%A8%20ZGC.md'>8、低延迟回收器 ZGC</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/9%E3%80%81%E6%97%A0%E6%93%8D%E4%BD%9C%E5%9B%9E%E6%94%B6%E5%99%A8%20Epsilon.md '> 9、无操作回收器 Epsilon</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/10%E3%80%81%E5%9B%9E%E6%94%B6%E5%99%A8%E5%B8%B8%E7%94%A8%E5%8F%82%E6%95%B0 '> 10、回收器常用参数</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/11%E3%80%81%E6%95%85%E9%9A%9C%E5%A4%84%E7%90%86%E5%B7%A5%E5%85%B7%E4%B8%8E%E5%85%AD%E7%A7%8DOOM.md '> 11、故障处理工具与六种 OOM</a>

执行子系统
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E6%89%A7%E8%A1%8C%E5%AD%90%E7%B3%BB%E7%BB%9F/1%E3%80%81%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%97%B6%E6%9C%BA.md'>1、类加载时机</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E6%89%A7%E8%A1%8C%E5%AD%90%E7%B3%BB%E7%BB%9F/2%E3%80%81%E7%B1%BB%E5%8A%A0%E8%BD%BD%E8%BF%87%E7%A8%8B.md'>2、类加载过程 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E6%89%A7%E8%A1%8C%E5%AD%90%E7%B3%BB%E7%BB%9F/3%E3%80%81%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E8%AF%A6%E8%A7%A3%EF%BC%88JDK9%2B).md'>3、类加载器详解（JDK9+) </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E6%89%A7%E8%A1%8C%E5%AD%90%E7%B3%BB%E7%BB%9F/4%E3%80%81%E6%96%B9%E6%B3%95%E6%89%A7%E8%A1%8C%E6%8C%87%E4%BB%A4%E8%AF%A6%E8%A7%A3.md'> 4、方法执行指令详解</a>

编译与优化
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E7%BC%96%E8%AF%91%E4%B8%8E%E4%BC%98%E5%8C%96/1%E3%80%81%E5%89%8D%E7%AB%AF%E7%BC%96%E8%AF%91%E4%B8%8E%E8%AF%AD%E6%B3%95%E7%B3%96.md'>1、前端编译与语法糖 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/JVM/%E7%BC%96%E8%AF%91%E4%B8%8E%E4%BC%98%E5%8C%96/2%E3%80%81%E5%90%8E%E7%AB%AF%E7%BC%96%E8%AF%91%E4%BC%98%E5%8C%96.md'>2、后端编译优化 </a>

## 2、数据库
### Redis 底层实现
数据结构与对象
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E4%B8%8E%E5%AF%B9%E8%B1%A1/1%E3%80%81SDS%E3%80%81%E6%95%B4%E6%95%B0%E9%9B%86%E5%90%88%E3%80%81%E5%8E%8B%E7%BC%A9%E5%88%97%E8%A1%A8.md'> 1、SDS、整数集合、压缩列表 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E4%B8%8E%E5%AF%B9%E8%B1%A1/2%E3%80%81%E5%8F%8C%E7%AB%AF%E9%93%BE%E8%A1%A8%E3%80%81%E5%AD%97%E5%85%B8%E3%80%81%E8%B7%B3%E8%A1%A8.md'>2、双端链表、字典、跳表  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E4%B8%8E%E5%AF%B9%E8%B1%A1/3%E3%80%81RedisObject%E5%AE%9E%E7%8E%B0.md'> 3、RedisObject 实现 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E4%B8%8E%E5%AF%B9%E8%B1%A1/4%E3%80%81%E4%BA%94%E7%A7%8D%E5%9F%BA%E6%9C%AC%E7%B1%BB%E5%9E%8B%E5%AF%B9%E8%B1%A1%E5%AE%9E%E7%8E%B0.md'> 4、五种基本类型对象实现 </a>

核心实现
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/Redis%E5%AE%9E%E7%8E%B0/1%E3%80%81%E4%BA%8B%E4%BB%B6%E9%A9%B1%E5%8A%A8%E6%A8%A1%E5%9E%8B%E5%AE%9E%E7%8E%B0.md'>1、事件驱动模型实现  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/Redis%E5%AE%9E%E7%8E%B0/2%E3%80%81RedisServer%E5%AE%9E%E7%8E%B0.md'>2、RedisServer 实现  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/Redis%E5%AE%9E%E7%8E%B0/3%E3%80%81%E5%88%9D%E5%A7%8B%E5%8C%96%E6%9C%8D%E5%8A%A1%E5%99%A8.md'>  3、初始化服务器</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/Redis%E5%AE%9E%E7%8E%B0/4%E3%80%81RedisClient%E5%AE%9E%E7%8E%B0.md'>4、RedisClient 实现  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/Redis%E5%AE%9E%E7%8E%B0/5%E3%80%81RedisDB%E5%AE%9E%E7%8E%B0.md'> 5、RedisDB 实现 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/Redis%E5%AE%9E%E7%8E%B0/6%E3%80%81%E8%BF%87%E6%9C%9F%E9%94%AE%E5%A4%84%E7%90%86.md'>6、过期键处理  </a>

持久化实现
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E6%8C%81%E4%B9%85%E5%8C%96/1%E3%80%81RDB%E5%AE%9E%E7%8E%B0.md'>1、RDB 实现  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E6%8C%81%E4%B9%85%E5%8C%96/2%E3%80%81AOF%E5%AE%9E%E7%8E%B0.md'> 2、AOF 实现 </a>

多机实现
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E5%A4%9A%E6%9C%BA/1%E3%80%81%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E5%AE%9E%E7%8E%B0.md'> 1、主从复制实现 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E5%A4%9A%E6%9C%BA/2%E3%80%81%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E6%AD%A5%E9%AA%A4.md'> 2、主从复制步骤 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E5%A4%9A%E6%9C%BA/3%E3%80%81%E5%93%A8%E5%85%B5Sentinel%E5%AE%9E%E7%8E%B0.md'>3、哨兵 Sentinel 实现  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E5%A4%9A%E6%9C%BA/4%E3%80%81Sentinel%E8%87%AA%E5%8A%A8%E6%95%85%E9%9A%9C%E8%BD%AC%E7%A7%BB.md'>4、Sentinel 自动故障转移  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E5%A4%9A%E6%9C%BA/5%E3%80%81%E9%9B%86%E7%BE%A4%E5%AE%9E%E7%8E%B0.md'> 5、集群实现 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E5%A4%9A%E6%9C%BA/6%E3%80%81%E9%9B%86%E7%BE%A4MOVED%E9%94%99%E8%AF%AF%E4%B8%8EASK%E9%94%99%E8%AF%AF.md'> 6、集群 MOVED 错误与 ASK 错误 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E5%A4%9A%E6%9C%BA/7%E3%80%81%E9%9B%86%E7%BE%A4%E5%86%85%E9%83%A8%E6%B6%88%E6%81%AF.md'>7、集群内部消息  </a>

功能实现
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E7%8B%AC%E7%AB%8B%E5%8A%9F%E8%83%BD/%E9%80%9A%E7%9F%A5%E5%8A%9F%E8%83%BD%E5%AE%9E%E7%8E%B0.md'>  通知功能实现</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E7%8B%AC%E7%AB%8B%E5%8A%9F%E8%83%BD/%E5%8F%91%E5%B8%83%E8%AE%A2%E9%98%85%E5%AE%9E%E7%8E%B0.md'> 发布订阅实现 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E7%8B%AC%E7%AB%8B%E5%8A%9F%E8%83%BD/%E6%85%A2%E6%9F%A5%E8%AF%A2%E6%97%A5%E5%BF%97%E5%AE%9E%E7%8E%B0.md'> 慢查询日志实现 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E7%8B%AC%E7%AB%8B%E5%8A%9F%E8%83%BD/%E7%9B%91%E8%A7%86%E5%99%A8%E5%AE%9E%E7%8E%B0.md'> 监视器实现 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/Redis/%E7%8B%AC%E7%AB%8B%E5%8A%9F%E8%83%BD/%E4%BA%8B%E5%8A%A1%E5%8A%9F%E8%83%BD%E5%AE%9E%E7%8E%B0.md'> 事务功能实现 </a>

### 数据库原理
<a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E6%80%BB%E4%BD%93%E6%A6%82%E8%BF%B0.md'> 总体概述  </a>

关系模型
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%85%B3%E7%B3%BB%E6%A8%A1%E5%9E%8B/1%E3%80%81%E7%BB%93%E6%9E%84%E4%B8%8E%E6%A8%A1%E5%9E%8B.md'> 1、结构与模型 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%85%B3%E7%B3%BB%E6%A8%A1%E5%9E%8B/2%E3%80%81%E5%85%B3%E7%B3%BB%E6%A8%A1%E5%9E%8B.md'> 2、关系模型 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%85%B3%E7%B3%BB%E6%A8%A1%E5%9E%8B/3%E3%80%81%E5%85%B3%E7%B3%BB%E4%BB%A3%E6%95%B0.md'> 3、关系代数 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%85%B3%E7%B3%BB%E6%A8%A1%E5%9E%8B/4%E3%80%81%E5%85%B3%E7%B3%BB%E6%BC%94%E7%AE%97.md'> 4、关系演算 </a>

标准 SQL
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E6%A0%87%E5%87%86SQL/1%E3%80%81%E5%9F%BA%E6%9C%ACSQL%E8%AF%AD%E8%A8%80.md'>1、基本SQL语言  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E6%A0%87%E5%87%86SQL/2%E3%80%81%E5%A4%8D%E6%9D%82%E6%9F%A5%E8%AF%A2%E4%B8%8E%E8%A7%86%E5%9B%BE.md'> 2、复杂查询与视图 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E6%A0%87%E5%87%86SQL/3%E3%80%81%E5%AE%8C%E6%95%B4%E6%80%A7%E5%92%8C%E5%AE%89%E5%85%A8%E6%80%A7.md'> 3、完整性和安全性 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E6%A0%87%E5%87%86SQL/4%E3%80%81%E5%B5%8C%E5%85%A5%E5%BC%8FSQL.md'> 4、嵌入式SQL </a>

建模设计
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%BB%BA%E6%A8%A1%E8%AE%BE%E8%AE%A1/1%E3%80%81E-R%E6%A8%A1%E5%9E%8B.md'> 1、数据库E-R模型 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%BB%BA%E6%A8%A1%E8%AE%BE%E8%AE%A1/2%E3%80%81IDEF1x%E5%B7%A5%E7%A8%8B%E5%8C%96%E6%96%B9%E6%B3%95.md'> 2、IDEF1x工程化方法 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%BB%BA%E6%A8%A1%E8%AE%BE%E8%AE%A1/3%E3%80%81%E5%87%BD%E6%95%B0%E4%BE%9D%E8%B5%96.md'> 3、关系函数依赖 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%BB%BA%E6%A8%A1%E8%AE%BE%E8%AE%A1/4%E3%80%81%E5%85%B3%E7%B3%BB%E8%8C%83%E5%BC%8F.md'> 4、五大关系范式 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%BB%BA%E6%A8%A1%E8%AE%BE%E8%AE%A1/5%E3%80%81%E6%A8%A1%E5%BC%8F%E5%88%86%E8%A7%A3.md'> 5、关系模式分解 </a>

实现技术
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%AE%9E%E7%8E%B0%E6%8A%80%E6%9C%AF/1%E3%80%81%E5%AD%98%E5%82%A8%E4%BD%93%E7%B3%BB.md'> 1、存储体系与组织 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%AE%9E%E7%8E%B0%E6%8A%80%E6%9C%AF/2%E3%80%81%E7%B4%A2%E5%BC%95%E5%8F%8A%E5%88%86%E7%B1%BB.md'> 2、索引存储及分类 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%AE%9E%E7%8E%B0%E6%8A%80%E6%9C%AF/3%E3%80%81B%2B%E6%A0%91%E7%B4%A2%E5%BC%95%E7%BB%93%E6%9E%84.md'> 3、B+树索引结构 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%AE%9E%E7%8E%B0%E6%8A%80%E6%9C%AF/4%E3%80%81%E6%95%A3%E5%88%97%E7%B4%A2%E5%BC%95%E7%BB%93%E6%9E%84.md'> 4、散列索引结构 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%AE%9E%E7%8E%B0%E6%8A%80%E6%9C%AF/5%E3%80%81%E6%9F%A5%E8%AF%A2%E5%AE%9E%E7%8E%B0.md'> 5、关系查询实现 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%AE%9E%E7%8E%B0%E6%8A%80%E6%9C%AF/6%E3%80%81%E4%B8%A4%E8%B6%9F%E6%89%AB%E6%8F%8F%E7%AE%97%E6%B3%95.md'> 6、两趟扫描算法 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%AE%9E%E7%8E%B0%E6%8A%80%E6%9C%AF/7%E3%80%81%E6%9F%A5%E8%AF%A2%E4%BC%98%E5%8C%96.md'>7、内部查询优化  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%AE%9E%E7%8E%B0%E6%8A%80%E6%9C%AF/8%E3%80%81%E4%BA%8B%E5%8A%A1%E8%B0%83%E5%BA%A6%E4%B8%8E%E5%B0%81%E9%94%81.md'> 8、事务调度与封锁 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%AE%9E%E7%8E%B0%E6%8A%80%E6%9C%AF/9%E3%80%81%E5%9F%BA%E4%BA%8E%E5%9B%9E%E6%BB%9A%E4%BA%8B%E5%8A%A1'> 9、基于回滚事务调度 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%AE%9E%E7%8E%B0%E6%8A%80%E6%9C%AF/10%E3%80%81%E6%95%85%E9%9A%9C%E6%81%A2%E5%A4%8D.md'> 10、故障恢复实现 </a>

## 3、计算机
### 计算机组成原理
<a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/%E6%A6%82%E8%AE%BA/%E8%AE%A1%E7%AE%97%E6%9C%BA%E6%80%BB%E4%BD%93%E6%A6%82%E8%BF%B0.md'> 计算机总体概述 </a>

硬件结构
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/%E7%A1%AC%E4%BB%B6%E7%BB%93%E6%9E%84/1%E3%80%81%E7%B3%BB%E7%BB%9F%E6%80%BB%E7%BA%BF.md'> 1、系统总线 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/%E7%A1%AC%E4%BB%B6%E7%BB%93%E6%9E%84/2%E3%80%81%E5%AD%98%E5%82%A8%E5%99%A8.md'> 2、存储器 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/%E7%A1%AC%E4%BB%B6%E7%BB%93%E6%9E%84/3%E3%80%81%E4%B8%BB%E5%AD%98%E5%82%A8%E5%99%A8.md'>3、主存储器  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/%E7%A1%AC%E4%BB%B6%E7%BB%93%E6%9E%84/4%E3%80%81IO%E7%B3%BB%E7%BB%9F.md'> 4、IO系统 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/%E7%A1%AC%E4%BB%B6%E7%BB%93%E6%9E%84/5%E3%80%81IO%E6%8E%A7%E5%88%B6%E6%96%B9%E5%BC%8F.md'>5、IO控制方式  </a>

CPU
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/CPU/1%E3%80%81%E6%95%B0%E7%9A%84%E8%A1%A8%E7%A4%BA.md'>1、数的表示  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/CPU/2%E3%80%81%E6%95%B0%E7%9A%84%E8%BF%90%E7%AE%97.md'>2、数的运算  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/CPU/4%E3%80%81%E6%8C%87%E4%BB%A4%E7%B3%BB%E7%BB%9F.md'> 3、指令系统 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/CPU/5%E3%80%81%E5%AF%BB%E5%9D%80%E4%B8%8E%E8%AE%BE%E8%AE%A1.md'>4、寻址与设计  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/CPU/6%E3%80%81CPU%E7%BB%93%E6%9E%84.md'> 5、CPU 结构 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/CPU/7%E3%80%81%E6%8C%87%E4%BB%A4%E6%B5%81%E6%B0%B4.md'>6、指令流水  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/CPU/8%E3%80%81%E4%B8%AD%E6%96%AD%E7%B3%BB%E7%BB%9F.md'> 7、中断系统 </a>

CU
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/CU/1%E3%80%81%E6%8E%A7%E5%88%B6%E5%8D%95%E5%85%83%E4%B8%8E%E5%BE%AE%E6%93%8D%E4%BD%9C.md'> 1、微操作与 CU </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/CU/2%E3%80%81%E7%BB%84%E5%90%88%E9%80%BB%E8%BE%91%E8%AE%BE%E8%AE%A1.md'>  2、组合逻辑设计</a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E8%AE%A1%E7%BB%84/CU/3%E3%80%81%E5%BE%AE%E7%A8%8B%E5%BA%8F%E8%AE%BE%E8%AE%A1.md'> 3、微程序设计 </a>

### 计算机体系结构
<a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E6%A6%82%E8%AE%BA/%E5%85%A8%E5%B1%80%E6%A6%82%E8%BF%B0.md'>体系结构概述  </a>

<a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E6%A6%82%E8%AE%BA/%E8%A7%84%E5%88%99%E4%B8%8E%E5%85%AC%E5%BC%8F.md'> 规则与公式 </a>

指令系统
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E6%8C%87%E4%BB%A4%E7%B3%BB%E7%BB%9F/1%E3%80%81%E6%8C%87%E4%BB%A4%E9%9B%86%E4%B8%8EMIPS.md'> 1、指令集与MIPS </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E6%8C%87%E4%BB%A4%E7%B3%BB%E7%BB%9F/2%E3%80%81%E6%B5%81%E6%B0%B4%E7%BA%BF%E6%80%A7%E8%83%BD.md'> 2、流水线性能 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E6%8C%87%E4%BB%A4%E7%B3%BB%E7%BB%9F/3%E3%80%81%E9%9D%9E%E7%BA%BF%E6%80%A7%E6%B5%81%E6%B0%B4%E8%B0%83%E5%BA%A6.md'> 3、非线性流水调度 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E6%8C%87%E4%BB%A4%E7%B3%BB%E7%BB%9F/4%E3%80%81%E7%9B%B8%E5%85%B3%E4%B8%8E%E5%86%B2%E7%AA%81.md'> 4、相关与冲突 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E6%8C%87%E4%BB%A4%E7%B3%BB%E7%BB%9F/5%E3%80%81%E6%B5%81%E6%B0%B4%E7%BA%BF%E5%AE%9E%E7%8E%B0.md'> 5、流水线实现 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E6%8C%87%E4%BB%A4%E7%B3%BB%E7%BB%9F/6%E3%80%81%E6%8C%87%E4%BB%A4%E5%B9%B6%E8%A1%8C.md'>6、指令并行  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E6%8C%87%E4%BB%A4%E7%B3%BB%E7%BB%9F/7%E3%80%81%E6%8C%87%E4%BB%A4%E8%B0%83%E5%BA%A6.md'> 7、指令调度 </a>

硬件系统
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E7%A1%AC%E4%BB%B6%E7%B3%BB%E7%BB%9F/1%E3%80%81Cache%E6%98%A0%E5%83%8F%E5%8F%8A%E5%8F%98%E6%8D%A2.md'> 1、Cache映像及变换 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E7%A1%AC%E4%BB%B6%E7%B3%BB%E7%BB%9F/2%E3%80%81%E6%8F%90%E9%AB%98Cache%E5%91%BD%E4%B8%AD%E7%8E%87.md'> 2、提高Cache命中率 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E7%A1%AC%E4%BB%B6%E7%B3%BB%E7%BB%9F/3%E3%80%81%E9%99%8D%E4%BD%8ECache%E5%BC%80%E9%94%80.md'>3、降低Cache开销  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E7%A1%AC%E4%BB%B6%E7%B3%BB%E7%BB%9F/4%E3%80%81%E5%B9%B6%E8%A1%8C%E4%B8%BB%E5%AD%98.md'> 4、并行主存 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E7%A1%AC%E4%BB%B6%E7%B3%BB%E7%BB%9F/5%E3%80%81%E8%99%9A%E6%8B%9F%E5%AD%98%E5%82%A8%E5%99%A8.md'> 5、虚拟存储器 </a>

多处理器
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E5%A4%9A%E5%A4%84%E7%90%86%E5%99%A8/1%E3%80%81%E4%BA%92%E8%BF%9E%E7%BD%91%E7%BB%9C.md'> 1、互连网络 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E5%A4%9A%E5%A4%84%E7%90%86%E5%99%A8/2%E3%80%81%E5%8A%A8%E6%80%81%E4%BA%92%E8%BF%9E%E7%BD%91%E7%BB%9C.md'> 2、动态互连网络 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E5%A4%9A%E5%A4%84%E7%90%86%E5%99%A8/3%E3%80%81%E5%A4%9A%E5%A4%84%E7%90%86%E5%99%A8.md'>3、多处理器  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E5%A4%9A%E5%A4%84%E7%90%86%E5%99%A8/4%E3%80%81%E4%B8%80%E8%87%B4%E6%80%A7%E9%97%AE%E9%A2%98.md'>4、一致性问题  </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E5%A4%9A%E5%A4%84%E7%90%86%E5%99%A8/5%E3%80%81%E5%90%8C%E6%AD%A5%E5%AE%9E%E7%8E%B0.md'> 5、同步实现 </a>
* <a href ='https://github.com/yzx66-net/Java_Review/blob/main/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/%E5%A4%9A%E5%A4%84%E7%90%86%E5%99%A8/6%E3%80%81%E5%B9%B6%E5%8F%91%E4%BF%9D%E8%AF%81.md'> 6、并发保证 </a>

### 操作系统（Linux）
启动与接口
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%90%AF%E5%8A%A8%E4%B8%8E%E6%8E%A5%E5%8F%A3/1%E3%80%81Linux%20%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8%EF%BC%9Abootsect.s%E3%80%81setup.s.md'>1、Linux 系统启动：bootsect.s、setup.s  </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%90%AF%E5%8A%A8%E4%B8%8E%E6%8E%A5%E5%8F%A3/2%E3%80%81Linux%20%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8%EF%BC%9Ahead.s%E3%80%81main.c.md'> 2、Linux 系统启动：head.s、main.c </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%90%AF%E5%8A%A8%E4%B8%8E%E6%8E%A5%E5%8F%A3/3%E3%80%81%E5%86%85%E6%A0%B8%E6%8E%A5%E5%8F%A3%E4%B8%8E%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.md'> 3、内核接口与实现原理 </a>

进程管理
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86/1%E3%80%81%E8%BF%9B%E7%A8%8B%E8%A7%86%E5%9B%BE%E4%B8%8E%E5%9F%BA%E6%9C%AC%E9%97%AE%E9%A2%98.md'>1、进程视图与基本问题  </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86/2%E3%80%81%E7%94%A8%E6%88%B7%E7%BA%A7%E7%BA%BF%E7%A8%8B%E4%B8%8E%E5%86%85%E6%A0%B8%E7%BA%A7%E7%BA%BF%E7%A8%8B%E5%AE%9E%E7%8E%B0.md'>2、用户级线程与内核级线程实现  </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86/3%E3%80%81%E5%A4%9A%E8%BF%9B%E7%A8%8B%E8%B5%B7%E7%82%B9%200%20%E5%8F%B7%E5%92%8C%201%20%E5%8F%B7%E8%BF%9B%E7%A8%8B.md'> 3、多进程起点 0 号和 1 号进程 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86/4%E3%80%81CPU%20%E8%B0%83%E5%BA%A6%E7%AE%97%E6%B3%95%E4%B8%8E%E5%AE%9E%E7%8E%B0.md'> 4、CPU 调度算法与实现 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86/5%E3%80%81%E4%B8%B4%E7%95%8C%E5%8C%BA%E7%AE%97%E6%B3%95%E4%B8%8E%E4%BF%A1%E5%8F%B7%E9%87%8F%E5%AE%9E%E7%8E%B0.md'>5、临界区算法与信号量实现  </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86/6%E3%80%81%E6%AD%BB%E9%94%81%E9%97%AE%E9%A2%98%E5%8F%8A%E5%A4%9A%E7%A7%8D%E5%A4%84%E7%90%86%E7%AD%96%E7%95%A5.md'> 6、死锁问题及多种处理策略 </a>

内存管理
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/1%E3%80%81%E7%A8%8B%E5%BA%8F%E9%87%8D%E5%AE%9A%E4%BD%8D%E4%B8%8E%E5%86%85%E5%AD%98%E5%88%86%E5%8C%BA.md'> 1、程序重定位与内存分区 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/2%E3%80%81%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E5%8F%8A%20Linux%20%E5%AE%9E%E7%8E%B0.md'>2、虚拟内存及 Linux 实现  </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/3%E3%80%81%E9%A1%B5%E9%9D%A2%E6%8D%A2%E5%85%A5%E6%8D%A2%E5%87%BA%E5%AE%9E%E7%8E%B0.md'> 3、页面换入换出实现 </a>


外设管理
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%A4%96%E8%AE%BE%E7%AE%A1%E7%90%86/1%E3%80%81%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8%20printf%20%E4%B8%8E%20scanf%20%E5%AE%9E%E7%8E%B0.md'> 1、设备驱动 printf 与 scanf 实现 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%A4%96%E8%AE%BE%E7%AE%A1%E7%90%86/2%E3%80%81%E7%A3%81%E7%9B%98%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86%E4%B8%8E%E7%9B%98%E5%9D%97%E7%BC%96%E5%8F%B7.md'>2、磁盘基本原理与盘块编号  </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%A4%96%E8%AE%BE%E7%AE%A1%E7%90%86/3%E3%80%81%E7%A3%81%E7%9B%98%E8%AF%B7%E6%B1%82%E9%98%9F%E5%88%97%E8%B0%83%E5%BA%A6%E4%B8%8E%E5%86%85%E6%A0%B8%E9%AB%98%E9%80%9F%E7%BC%93%E5%AD%98.md'>  3、磁盘请求队列调度与内核高速缓存</a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%A4%96%E8%AE%BE%E7%AE%A1%E7%90%86/4%E3%80%81%E5%9F%BA%E4%BA%8E%E6%96%87%E4%BB%B6%E7%9A%84%E7%A3%81%E7%9B%98%E4%BD%BF%E7%94%A8.md'> 4、基于文件的磁盘使用 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%A4%96%E8%AE%BE%E7%AE%A1%E7%90%86/5%E3%80%81Linux%20%E5%AE%8C%E6%95%B4%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E5%AE%9E%E7%8E%B0.md'> 5、Linux 完整文件系统实现 </a>

Linux IO 特性
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/Linux%20IO%20%E7%89%B9%E6%80%A7/1%E3%80%81Linux%20%E7%BD%91%E7%BB%9C%20IO%20%E6%A8%A1%E5%9E%8B.md'> 1、Linux 网络 IO 模型 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/Linux%20IO%20%E7%89%B9%E6%80%A7/2%E3%80%81IO%20%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%20epoll%20%E8%AF%A6%E8%A7%A3.md'>2、IO 多路复用 epoll 详解  </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/Linux%20IO%20%E7%89%B9%E6%80%A7/3%E3%80%81Linux%20%E9%9B%B6%E6%8B%B7%E8%B4%9D%E6%8A%80%E6%9C%AF.md'>3、Linux 零拷贝技术  </a>

### 计算机网络
<a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E7%BD%91%E7%BB%9C%E6%A6%82%E8%BF%B0%E4%B8%8E%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84.md'>网络概述与分层结构  </a>

五层模型
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/1%E3%80%81%E7%89%A9%E7%90%86%E5%B1%82%E6%A0%B8%E5%BF%83%E5%9F%BA%E7%A1%80.md'> 1、物理层核心基础 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/2.1%E3%80%81%E9%93%BE%E8%B7%AF%E5%B1%82%E7%9A%84%E5%B7%AE%E9%94%99%E6%8E%A7%E5%88%B6%E4%B8%8E%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6.md'> 2.1、链路层的差错控制与流量控 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/2.2%E3%80%81%E5%B9%BF%E6%92%AD%E9%93%BE%E8%B7%AFMAC%E5%8D%8F%E8%AE%AE.md'>2.2、广播链路MAC协议  </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/2.3%E3%80%81%E5%A4%9A%E7%A7%8D%E5%B1%80%E5%9F%9F%E7%BD%91%E5%8D%8F%E8%AE%AE%E5%8F%8A%E6%8A%80%E6%9C%AF.md'>  2.3、多种局域网协议及技术</a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/3.1%E3%80%81%E7%BD%91%E7%BB%9C%E5%B1%82%E6%A6%82%E8%BF%B0.md'> 3.1、网络层概述 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/3.2%E3%80%81Internet%20%E8%B7%AF%E7%94%B1%E5%8D%8F%E8%AE%AE.md'>  3.2、Internet 路由协议</a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/3.3%E3%80%81IP%20%E5%8D%8F%E8%AE%AE.md'>3.3、IP 协议  </a>
* <a href ='https://github.com/yzx66-net/Java_Note/edit/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/3.4%E3%80%81%E7%BD%91%E7%BB%9C%E5%B1%82%E5%85%B6%E4%BB%96%E5%8D%8F%E8%AE%AE%E4%B8%8E%E6%8A%80%E6%9C%AF.md'> 3.4、网络层其他协议与技术 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/4.1%E3%80%81%E4%BC%A0%E8%BE%93%E5%B1%82%E4%B8%8E%20UDP.md'> 4.1、传输层与 UDP 协议 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/4.2%E3%80%81TCP%20%E5%8D%8F%E8%AE%AE.md'> 4.2、TCP 协议</a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/4.3%E3%80%81TCP%20%E5%8F%AF%E9%9D%A0%E4%BC%A0%E8%BE%93%E5%8E%9F%E7%90%86.md'> 4.3、TCP 可靠传输原理 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/4.4%E3%80%81TCP%20%E5%AE%9E%E7%8E%B0%E5%8F%AF%E9%9D%A0%E4%BC%A0%E8%BE%93%E7%9A%84%E6%96%B9%E5%BC%8F.md'> 4.4、TCP 实现可靠传输的方式 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/4.5%E3%80%81TCP%20%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6%E4%B8%8E%E6%8B%A5%E5%A1%9E%E6%8E%A7%E5%88%B6%E5%AE%9E%E7%8E%B0.md'> 4.5、TCP 流量控制与拥塞控制实现 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/5.1%E3%80%81HTTP%E3%80%81SMTP%E3%80%81POP3.md'> 5.1、HTTP、SMTP、POP3 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E4%BA%94%E5%B1%82%E6%A8%A1%E5%9E%8B/5.2%E3%80%81FTP%E3%80%81DNS%E3%80%81DHCP.md'>  5.2、FTP、DNS、DHCP </a>

网络安全
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E7%BD%91%E7%BB%9C%E5%AE%89%E5%85%A8/1%E3%80%81%E7%BD%91%E7%BB%9C%E5%AE%89%E5%85%A8%E5%9F%BA%E7%A1%80.md'>1、网络安全基础  </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E7%BD%91%E7%BB%9C%E5%AE%89%E5%85%A8/2%E3%80%81SSL%20%E5%8D%8F%E8%AE%AE.md'> 2、SSL 协议 </a>
* <a href ='https://github.com/yzx66-net/Java_Note/blob/main/%E8%AE%A1%E7%BD%91/%E7%BD%91%E7%BB%9C%E5%AE%89%E5%85%A8/3%E3%80%81IPSec%20%E5%8D%8F%E8%AE%AE.md'> 3、IPSec 协议 </a>

### 编译原理
流程
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E7%BC%96%E8%AF%91%E5%8E%9F%E7%90%86/%E7%BC%96%E8%AF%91%E5%8F%8A%E5%85%B6%E6%B5%81%E7%A8%8B%E6%A6%82%E8%BF%B0.md'>编译及其流程</a>

前端
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E7%BC%96%E8%AF%91%E5%8E%9F%E7%90%86/%E5%89%8D%E7%AB%AF/1%E3%80%81%E8%AF%8D%E6%B3%95%E5%8F%8A%E5%88%86%E6%9E%90%E4%B8%8E%E6%9C%89%E9%99%90%E8%87%AA%E5%8A%A8%E6%9C%BA.md'>  1、词法及分析与有限自动机</a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E7%BC%96%E8%AF%91%E5%8E%9F%E7%90%86/%E5%89%8D%E7%AB%AF/2%E3%80%81%E8%AF%8D%E6%B3%95%20DFA%20%E5%8F%8A%E5%88%86%E6%9E%90%E5%99%A8%E6%9E%84%E9%80%A0.md'> 2、词法 DFA 及分析器构造 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E7%BC%96%E8%AF%91%E5%8E%9F%E7%90%86/%E5%89%8D%E7%AB%AF/3%E3%80%81%E8%AF%AD%E6%B3%95%E5%88%86%E6%9E%90%E4%B8%8E%E4%B8%8A%E4%B8%8B%E6%96%87%E6%97%A0%E5%85%B3%E6%96%87%E6%B3%95.md'> 3、语法分析与上下文无关文法 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E7%BC%96%E8%AF%91%E5%8E%9F%E7%90%86/%E5%89%8D%E7%AB%AF/4%E3%80%81%E8%87%AA%E4%B8%8A%E8%80%8C%E4%B8%8B%E5%88%86%E6%9E%90%E6%B3%95%E4%B8%8E%20LL(1)%20%E6%96%87%E6%B3%95.md'> 4、自上而下分析法与 LL(1) 文法 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E7%BC%96%E8%AF%91%E5%8E%9F%E7%90%86/%E5%89%8D%E7%AB%AF/5%E3%80%81%E8%87%AA%E4%B8%8B%E8%80%8C%E4%B8%8A%E5%88%86%E6%9E%90%E6%B3%95%E4%B8%8E%20LR(1)%20%E6%96%87%E6%B3%95.md'> 5、自下而上分析法与 LR(1) 文法 </a>

中间代码
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E7%BC%96%E8%AF%91%E5%8E%9F%E7%90%86/%E4%B8%AD%E9%97%B4%E4%BB%A3%E7%A0%81/1%E3%80%81%E8%AF%AD%E4%B9%89%E5%88%86%E6%9E%90%E4%B8%8E%E4%B8%AD%E9%97%B4%E4%BB%A3%E7%A0%81%E3%80%81%E7%AC%A6%E5%8F%B7%E8%A1%A8.md'>1、语义分析与中间代码、符号表  </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E7%BC%96%E8%AF%91%E5%8E%9F%E7%90%86/%E4%B8%AD%E9%97%B4%E4%BB%A3%E7%A0%81/2%E3%80%81%E5%8F%98%E9%87%8F%E4%B8%8E%E8%BF%87%E7%A8%8B%E7%BF%BB%E8%AF%91.md'> 2、变量与过程翻译 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E7%BC%96%E8%AF%91%E5%8E%9F%E7%90%86/%E4%B8%AD%E9%97%B4%E4%BB%A3%E7%A0%81/3%E3%80%81%E7%AE%97%E6%9C%AF%E8%A1%A8%E8%BE%BE%E5%BC%8F%E4%B8%8E%E6%95%B0%E7%BB%84%E5%85%83%E7%B4%A0%E7%BF%BB%E8%AF%91.md'> 3、算术表达式与数组元素翻译 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E7%BC%96%E8%AF%91%E5%8E%9F%E7%90%86/%E4%B8%AD%E9%97%B4%E4%BB%A3%E7%A0%81/4%E3%80%81%E5%B8%83%E5%B0%94%E8%A1%A8%E8%BE%BE%E5%BC%8F%E5%8F%8A%E6%8E%A7%E5%88%B6%E8%AF%AD%E5%8F%A5%E7%BF%BB%E8%AF%91.md'> 4、布尔表达式及控制语句翻译 </a>

后端
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E7%BC%96%E8%AF%91%E5%8E%9F%E7%90%86/%E5%90%8E%E7%AB%AF/%E7%9B%AE%E6%A0%87%E4%BB%A3%E7%A0%81%E7%94%9F%E6%88%90%E4%B8%8E%E4%BC%98%E5%8C%96.md'>目标代码生成与优化  </a>

## 4、软件与算法
### 软件工程
概述
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B/1%E3%80%81%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B%E6%A6%82%E8%BF%B0.md'> 1、软件工程概述 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B/2%E3%80%81%E8%BD%AF%E4%BB%B6%E8%BF%87%E7%A8%8B%E6%A8%A1%E5%9E%8B.md'>2、软件过程模型  </a>

工程流程
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B/3%E3%80%81%E7%BB%93%E6%9E%84%E5%8C%96%E9%9C%80%E6%B1%82%E5%88%86%E6%9E%90.md'>1、结构化需求分析  </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B/4%E3%80%81%E7%BB%93%E6%9E%84%E5%8C%96%E6%80%BB%E4%BD%93%E8%AE%BE%E8%AE%A1.md'> 2、结构化总体设计 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B/5%E3%80%81%E7%BB%93%E6%9E%84%E5%8C%96%E8%AF%A6%E7%BB%86%E8%AE%BE%E8%AE%A1.md'>3、结构化详细设计  </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B/6%E3%80%81%E7%B3%BB%E7%BB%9F%E5%AE%9E%E7%8E%B0%E4%B8%8E%E8%BD%AF%E4%BB%B6%E6%B5%8B%E8%AF%95.md'> 4、系统实现与软件测试 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B/7%E3%80%81%E7%BB%B4%E6%8A%A4%E3%80%81%E8%AF%84%E4%BC%B0%E3%80%81%E6%94%B9%E8%BF%9B.md'>5、维护、评估、改进 </a>

面向对象
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B/8%E3%80%81%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E6%96%B9%E6%B3%95%E5%AD%A6.md'>  1、面向对象方法学</a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B/9%E3%80%81%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E5%88%86%E6%9E%90.md'> 2、面向对象分析 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B/10%E3%80%81%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E8%AE%BE%E8%AE%A1.md'> 3、面向对象设计 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B/11%E3%80%81%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E5%AE%9E%E7%8E%B0%E9%A3%8E%E6%A0%BC.md'> 4、面向对象实现风格 </a>

管理
* <a href ='https://github.com/yzx66-net/Java-CS-Note/blob/main/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B/12%E3%80%81%E8%BD%AF%E4%BB%B6%E9%A1%B9%E7%9B%AE%E7%AE%A1%E7%90%86.md'> 软件项目管理 </a>


### Leetcode 典例
数组
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E6%95%B0%E7%BB%84/1%E3%80%81%E5%BF%AB%E6%85%A2%E6%8C%87%E9%92%88.md'> 1、快慢指针 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E6%95%B0%E7%BB%84/2%E3%80%81%E5%AF%B9%E6%92%9E%E6%8C%87%E9%92%88.md'> 2、对撞指针 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E6%95%B0%E7%BB%84/3%E3%80%81%E6%BB%91%E5%8A%A8%E7%AA%97%E5%8F%A3.md'>3、滑动窗口  </a>

链表
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E9%93%BE%E8%A1%A8/1%E3%80%81%E5%BF%85%E4%BC%9A%E6%93%8D%E4%BD%9C.md'> 1、典型操作 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E9%93%BE%E8%A1%A8/2%E3%80%81%E8%99%9A%E6%8B%9F%E5%A4%B4%E8%8A%82%E7%82%B9.md'> 2、虚拟头节点 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E9%93%BE%E8%A1%A8/3%E3%80%81%E5%8F%8C%E6%8C%87%E9%92%88%E6%88%96%E5%9B%9E%E6%BA%AF.md'>3、双指针或回溯  </a>

查找表
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E6%9F%A5%E6%89%BE%E8%A1%A8/1%E3%80%81Set%20%E4%B8%8E%E6%BB%91%E5%8A%A8%E7%AA%97%E5%8F%A3.md'> 1、Set 与滑动窗口 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E6%9F%A5%E6%89%BE%E8%A1%A8/2%E3%80%81Map%20%E5%B8%B8%E8%A7%81%E5%85%B8%E4%BE%8B.md'> 2、Map 常见典例 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E6%9F%A5%E6%89%BE%E8%A1%A8/3%E3%80%81%E5%87%A0%E6%95%B0%E5%92%8C%E7%B3%BB%E5%88%97%E9%97%AE%E9%A2%98.md'> 3、几数和系列问题 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E6%9F%A5%E6%89%BE%E8%A1%A8/4%E3%80%81%E7%89%B9%E6%AE%8A%E9%94%AE%E5%80%BC%E9%80%89%E6%8B%A9.md'> 4、特殊键值选择 </a>

栈、队列
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E6%A0%88%E3%80%81%E9%98%9F%E5%88%97/1%E3%80%81%E6%A0%88%E5%B8%B8%E8%A7%81%E9%A2%98%E5%9E%8B.md'> 1、栈常见题型 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E6%A0%88%E3%80%81%E9%98%9F%E5%88%97/2%E3%80%81%E7%94%A8%E6%A0%88%E4%BB%A3%E6%9B%BF%E9%80%92%E5%BD%92.md'> 2、用栈代替递归 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E6%A0%88%E3%80%81%E9%98%9F%E5%88%97/3%E3%80%81%E9%98%9F%E5%88%97%E4%B8%8E%20BFS.md'> 3、队列与 BFS </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E6%A0%88%E3%80%81%E9%98%9F%E5%88%97/4%E3%80%81%E4%BC%98%E5%85%88%E9%98%9F%E5%88%97.md'> 4、优先队列 </a>

二叉树
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E4%BA%8C%E5%8F%89%E6%A0%91/1%E3%80%81%E5%85%B8%E5%9E%8B%E6%93%8D%E4%BD%9C.md'> 1、典型操作 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E4%BA%8C%E5%8F%89%E6%A0%91/2%E3%80%81%E7%A8%8D%E5%A4%8D%E6%9D%82%E9%80%92%E5%BD%92%E9%97%AE%E9%A2%98.md'>2、稍复杂递归问题  </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E4%BA%8C%E5%8F%89%E6%A0%91/3%E3%80%81%E4%BA%8C%E5%88%86%E6%90%9C%E7%B4%A2%E6%A0%91.md'>3、二分搜索树  </a>

回溯
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E5%9B%9E%E6%BA%AF/1%E3%80%81%E5%B8%B8%E8%A7%81%E5%85%B8%E4%BE%8B.md'> 1、常见典例 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E5%9B%9E%E6%BA%AF/2%E3%80%81%E6%8E%92%E5%88%97%E4%B8%8E%E7%BB%84%E5%90%88.md'>2、排列与组合  </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E5%9B%9E%E6%BA%AF/3%E3%80%81%E4%BA%8C%E7%BB%B4%E5%B9%B3%E9%9D%A2%E7%B1%BB%E5%9E%8B.md'>3、二维平面类型  </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E5%9B%9E%E6%BA%AF/4%E3%80%81N%20%E7%9A%87%E5%90%8E%E9%97%AE%E9%A2%98.md'> 4、N 皇后问题 </a>

动态规划
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E5%8A%A8%E6%80%81%E8%A7%84%E5%88%92/1%E3%80%81%E5%B8%B8%E8%A7%81%E9%A2%98%E5%9E%8B%EF%BC%88%E4%B8%8A%EF%BC%89.md'>  1、常见题型（上）</a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E5%8A%A8%E6%80%81%E8%A7%84%E5%88%92/2%E3%80%81%E5%B8%B8%E8%A7%81%E9%A2%98%E5%9E%8B%EF%BC%88%E4%B8%8B%EF%BC%89.md'>2、常见题型（下）  </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E5%8A%A8%E6%80%81%E8%A7%84%E5%88%92/3%E3%80%81%E8%83%8C%E5%8C%85%E9%97%AE%E9%A2%98%E7%B3%BB%E5%88%97.md'> 3、背包问题系列 </a>
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E5%8A%A8%E6%80%81%E8%A7%84%E5%88%92/4%E3%80%81LCS%20%E4%B8%8E%20LIS%20%E9%97%AE%E9%A2%98.md'>4、LCS 与 LIS 问题  </a>

贪心
* <a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E8%B4%AA%E5%BF%83/%E5%B8%B8%E8%A7%81%E5%85%B8%E4%BE%8B.md'> 常见典例 </a>

**算法补充**

<a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E6%8E%92%E5%BA%8F%E7%AE%97%E6%B3%95%EF%BC%9A%E6%89%80%E6%9C%89%E5%B8%B8%E8%A7%81%E6%96%B9%E6%B3%95.md'> 排序算法：所有常见方法 </a>

<a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E6%A0%91%E7%9B%B8%E5%85%B3%E7%AE%97%E6%B3%95%EF%BC%9AAVL%20%E6%A0%91%E3%80%81%E7%BA%A2%E9%BB%91%E6%A0%91%E3%80%81B%5CB%2B%20%E6%A0%91.md'> 树相关算法：AVL 树、红黑树、B\B+ 树 </a>

<a href ='https://github.com/yzx66-net/Java-CS-Record/blob/main/LeetCode/%E5%9B%BE%E8%AE%BA%E7%AE%97%E6%B3%95%EF%BC%9A%E6%9C%80%E7%9F%AD%E8%B7%AF%E5%BE%84%E4%B8%8E%E6%9C%80%E5%B0%8F%E7%94%9F%E6%88%90%E6%A0%91.md'> 图论算法：最短路径与最小生成树 </a>

# 开发
## 1、常用技术
### Spring
正在进行...
### SpringMVC
正在进行...
### SpringBoot
正在进行...
### Mybatis
待续...
### Tomcat
待续...
## 2、微服务技术
### Dubbo
待续...
### Zookeeper
待续...
### SpringCloud
待续...
## 3、其余技术
### Netty
待续...
### Shiro
待续...
### ...
待续...


# 分布式
## 消息队列
待续...
## 分布式相关
待续...


# 附录
## 书籍记录与推荐
仅代表我读完后的个人观点（只有力荐里的与豆瓣评分无冲突，几乎都是高分）
* 万分力荐：代表我认为特别好的，如果想读些 Java 相关的书，建议一定读我里面罗列的，绝对物超所值。
* 比较推荐：代表我认为的好书，看完确实可以学到东西那种，但算不上特别好，不过还是很值得一读。
* 可以看看：代表我认为还是有一定缺陷的书，不是讲的不特别清楚，就是有点泛或者浅。
* 比较一般：代表我读完后收获较小的书，或者主观上不是很喜欢的书，并不代表里面的书一定不好。

**链接是豆瓣中该书的所有短评，避免只被我读完时的感受影响！**

### 万分力荐
* <a href="https://book.douban.com/subject/34907497/comments/"> 深入理解Java虚拟机（第3版）</a>：无需多言，刷了两次。
* <a href="https://book.douban.com/subject/25900156/comments/">Redis设计与实现</a>：也是刷了两次，我看过最深入浅出的书，一点没有门槛，看完觉得 Redis 非常明了。
* <a href="https://book.douban.com/subject/30391722/comments/"> 操作系统原理、实现与实践</a>：哈工大老师出品，除实践部分看了两次，围绕 Linux 作为原理的现实，注重抠细节，特别厉害。
* <a href="https://book.douban.com/subject/4199741/comments/"> 代码整洁之道</a>：绝大部分观点都认可，很多观点都让人佩服，比如代码要短小精悍，还要可以自解释等等。
* <a href="https://book.douban.com/subject/27087564/comments/">Mybatis技术内幕</a>：好书，从模块讲起，再讲处理流程，主干清晰明了，源码也讲的清楚。
* <a href="https://book.douban.com/subject/10426640/comments/">深入刨析Tomcat</a>：读过最好的源码书，没有之一，从假设自己要设计一个服务器出发，然后分析 Tomcat 完善自己的服务器。
* <a href="https://book.douban.com/subject/24708143/comments/"> MySQL技术内幕：</a>看这本书之前最好懂操作系统，不然很难受，而且第一章提的很多东西后面才讲，但确实是好书。
* <a href="https://book.douban.com/subject/26292004/comments/">从Paxos到Zookeeper </a>：豆瓣7.7，但是我认为是好书，不过 Paxos 那块讲的不是很清楚，还需要配合博客看看。
* <a href="https://book.douban.com/subject/33425123/comments/">微服务架构设计模式 </a>：好书，改变了我对微服务的看法，微服务根本不是用个 Dubbo 或者 SpringCloud 的事。


### 比较推荐
* <a href="https://book.douban.com/subject/30412517/comments/">Effective Java中文版（第3版）</a>：列了 90 条，核心感觉还是讲怎么用 Java 写更健壮和灵活的程序，写得还算不错
* <a href="https://book.douban.com/subject/26591326/comments/"> Java并发编程艺术</a>：这本书讲述顺序就是按照内存模型->synchronized->源码，总体觉得还不错，但是开头两章有点劝退。
* <a href="https://book.douban.com/subject/33390560/comments/">Spring Boot编程思想（核心篇）</a>：豆瓣评分较低 6.5，但是我觉得把 SpringBoot 比较核心的部分都讲了，就是确实凑字数太明显，啥都贴。
* <a href="https://book.douban.com/subject/30417623/comments/"> RocketMQ技术内幕</a>：豆瓣评分较低 6.9，不过我觉得主要原因可能把 Client 还有 Server 串着讲，阅读体验确实差，但内容尚可吧。
* <a href="https://book.douban.com/subject/27591386/comments/">RabbitMQ实战指南</a>：远超我的期望，冲着如何实现去的，实战书里少有的既有实战又有深度。
* <a href="https://book.douban.com/subject/30280001/comments/">计算机网络（原书第7版） </a>：不用多说，比教材易懂，也比教材讲的内容多，总体自顶向下，更容易理解点。
 


### 可以看看
* <a href="https://book.douban.com/subject/30452948/comments/">Spring源码深度解析（第二版）</a>：当时读的时候豆瓣 5.9 分，倒不是说不好，只是对第一次看源码的新手不太友好，而且确实绝大部分照搬第一版。
* <a href="https://book.douban.com/subject/34455777/comments/"> 深入理解Apache Dubbo与实战</a>：是我读过的源码书里不算好的，讲的不透彻，但拓展点还有 RPC 策略那讲的确实还行。
* <a href="https://book.douban.com/subject/25953851/comments/"> 深入分析Java Web技术内幕（修订版）</a>：如果看了我说的其他书，这本书完全没必要看，各个模块讲的很浅，但要想快速了解一下可以看看。
* <a href="https://book.douban.com/subject/27038538/comments/"> Netty实战 </a>：我一般不看实战书的，但是 Netty 的书太少了，以为有源码，结果一点没提，不过 Netty 用法讲的确实比网课好。
* <a href="https://book.douban.com/subject/25863515/comments/"> 图解HTTP</a>：比较浅，看这个是因为 HTTP 权威指南太厚，不过比一般大学教材 HTTP 部分讲的多。
* <a href="https://book.douban.com/subject/24737674/comments/"> 图解TCP/IP</a>：当时看的入门书，如果想深入学一下，还是推荐计网的教材或者其他书籍。
* <a href="https://book.douban.com/subject/27091029/comments/"> 分布式服务架构：原理、设计与实战</a>：架构没讲什么，说了点分布式的问题，分布式事务、性能估算还有日志框架啥的还行，最后几章完全凑数。

### 比较一般
* <a href="https://book.douban.com/subject/25723064/comments/"> 大型网站技术结构</a>：扫盲书，三天就看完了，建立个概念而已。
* <a href="https://book.douban.com/subject/26702824/comments/"> 分布式服务框架：原理与实践</a>：讲咋设计微服务框架的，比较一般，就讲了下微服务框架的几个关键点，总体比较宏观一些。
* <a href="https://book.douban.com/subject/25723658/comments/"> 大规模分布式存储系统</a>：分布式入门书，讲的分布式存储系统的宏观架构，并没有一些具体的细节，还讲了一些 OceanBase 基本原理。
* <a href="https://book.douban.com/subject/30386800/comments/"> Elasticsearch源码解析与优化实战</a>：叫源码解析，冲着核心源码去看的，结果没啥源码，讲的都是模块，而且也不够深入浅出。
* <a href="https://book.douban.com/subject/26919457/comments/"> 代码简洁之道：程序员的职业素养</a>：总体还行，存在部分观点很不认可，有点教条主义与理想化，尤其程序员对抗加班，还有什么必须完全 TDD。

### Cache映像及变换

#### 全相联

<img src="https://img-blog.csdnimg.cn/20201225155309796.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" />

映像规则

* 主存的任意一块可以映像到 cache 的任意一块（有 Cb * Mb 种）



地址变换规则

* 用硬件实现非常复杂

* 维护了一个目录表，去比较目录表

  <img src="https://img-blog.csdnimg.cn/20201225155340268.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" />





#### 直接映像
<img src="https://img-blog.csdnimg.cn/20201225155402610.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%"/>



映像规则

* 主存储器中一块只能映像到Cache的一个特定块中

* Cache 地址的计算公式

  b = B mod Cb

  * b 为 Cache 号
  * B 为主存块号
  * Cb 是 Cache 块数

* 实际上，Cache 地址与主存地址的低位部分完全相同

  * 即 cache 地址是主存的 块号 + 块内地址



地址映像变换过程

* 用主存地址中的块号B去访问区号存储器，把读出来的区号与主存地址中的区号E进行比较:

* 比较结果相等，有效位为1，则Cache命中，否则该块已经作废。

* 比较结果不相等，有效位为1，Cache中的该块是有用的，否则该块是空的。

  <img src="https://img-blog.csdnimg.cn/2020122515543611.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" />





直接映像优缺点

* 主要优点:
  * 硬件实现很简单，不需要相联访问存储器
  * 访问速度也比较快，实际上不需要进行地址变换 
* 主要缺点:
  * 块的冲突率比较高。



#### 组组相联

<img src="https://img-blog.csdnimg.cn/20201225155113944.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="45%"/>

映象规则:

* 主存和Cache按同样大小划分成块和组。
* 主存和Cache的组之间采用直接映象方式。
* 在两个对应的组内部采用全相联映象方式。



组相联映象的地址变换过程:

* 用主存地址中的组号G按地址访问块表存储器，把读出来的一组区号和块号与主存地址中的区号和块号进行相联比较。

* 如果有相等的，表示Cache命中;

* 如果全部不相等，表示Cache没有命中。

  <img src="https://img-blog.csdnimg.cn/20201225155139689.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="45%"/>





组相联映象优缺点

* 优点:
  * 块的冲突概率比较低，块的利用率大幅度提高，块失效率明显降低。
* 缺点:
  * 实现难度和造价要比直接映象方式高。 



提高 Cache 访问速度的一种方法

* 用多个相等的比较器代替相联访问

  <img src="https://img-blog.csdnimg.cn/20201225155240311.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" />





#### Cache与主存

如何写

* 写穿（Write through） :
  * 写cache，同时写内存
  * 优点:一致性好，缺点:慢
* 写回(write back);
  * 只写cache，只有被替换时才写回内存
  * 优点:快;缺点:出错率高一倍





如何换（Cache 替换算法）

* 使用的场合:
  * 直接映象方式实际上不需要替换算法
  * 全相联映象方式的替换算法最复杂
  * 主要用于组相联、段相联等映象方式中
* 要解决的问题:
  * 记录每次访问Cache的块号
  * 在访问过程中，对记录的块号进行管理根据记录和管理结果，找出替换的块号
* 主要特点:
  * 全部用硬件实现



轮转算法及实现（用于组相联映象方式中，有两种实现方法）

* 方法一：每块一个计数器
  * 在块表内增加一个替换计数器字段
    * 计数器的长度与Cache地址中的组内块号字段的长度相同。
  * 替换方法及计数器的管理规则
    * 新装入或替换的块，它的计数器清0，同组其它块的计数器都加“1”。
    * 在同组中选择计数器的值最大的块作为被替换的块。
* 方法二：每组一个计数器（轮转的思想）
  * 替换规则和计数器的管理：
    * 本组有替换时，计数器加“1”
    * 计数器的值就是要被替换出去的块号。

* 轮换法的优缺点
  * 优点
    * 实现比较简单，能够利用历史上的块地址流情况
  * 缺点
    * 没有利用程序的局部性特点





LRU 算法及实现

* 方法：为每一块设置一个计数器
  * 计数器的长度与块号字段的长度相同
  * 计数器的使用及管理规则:
    * 新装入或替换的块，计数器清0，同组中其它块的计数器加1。
    * 命中块的计数器清0，同组的其它计数器中，凡计数器的值小于命中块计数器原来值的加1，其余计数器不变。
    * 需要替换时，在同组的所有计数器中选择计数值最大的计数器，它所对应的块被替换。
* 优缺点
  * 主要优点:
    * 命中率比较高，
    * 能够比较正确地利用程序的局部性特点
    * 充分地利用历史上块地址流的分布情况
    * 是一种堆栈型算法，随着组内块数增加,命中率单调上升。
  * 主要缺点:
    * 控制逻辑复杂，因为增加了判断和处理是否命中的情况。


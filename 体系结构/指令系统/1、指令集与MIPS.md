### 指令集

#### 指令集结构

指令概述

* 指令集： 
  * 一些指令的集合；
  * 每条指令都是 直接由 由 CPU  硬件执行。

* 指令的表示方法：
  * 二进制格式；
  * 物理存储空间组织方式是位、字节、字和多字等；
  * 当前的指令字长有： 16 、  32 、  64  位；

* 指令的特点
  * 指令的操作十分简单，其操作由操作码编码表示。
  * 每个操作需要的操作数个数为 0-3  个不等。
    * 操作数是一些存储单元的地址；
    * 典型的存储单元通常有：主存、寄存器、堆栈和累加器。
  * 操作数地址隐含表示或显式表示。



指令集分类

* 五个因素

  * 在 CPU  中操作数的存储方法；
  * 指令中显式表示的操作数个数；
  * 操作数的寻址方式；
  * 指令集所提供的操作类型；
  * 操作数的类型和大小。

* 按存储操作数分类

  * 分类

    * 堆栈；
    * 累加器；
    * 一组寄存器

  * Z=X+Y  表达式在这三种类型指令集结构上的实现方法

    <img src="https://img-blog.csdnimg.cn/20201225150611549.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" />

  * 通用寄存器型指令集结构的分类

    <img src="https://img-blog.csdnimg.cn/20201225150540469.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" />

* 将当前大多数通用寄存器型指令集分为三类

  * 寄存器－寄存器型 (R - R ： register-register)
    * 优点：指令字长固定，指令结构简洁，是一种简单的代码生成模型，各种指令的执行时钟周期数相近 。
    * 缺点：与指令中含存储器操作数的指令系统结构相比，指令条数多，目标代码不够紧凑，因而程序占用的空间比较大 。
  * 寄存器－存储器型 (R － M ： register-memory)
    * 优点：可以在 ALU  指令中直接对存储器操作数进行引用，而不必先用 load  指令进行加载，容易对指令进行编码，目标代码比较紧凑 。
    * 缺点：由于有一个操作数的内容将被破坏，所以指令中的两个操作数不对称。在一条指令中同时对寄存器操作数和存储器操作数进行编码，有可能限制指令所能够表示的寄存器个数。指令的执行时钟周期因操作数的来源（寄存器或存储器）的不同而差别比较大
  * 存储器－存储器型 (M － M ： memory-memory)
    * 优点：目标代码最紧凑，不需要设置存储器来保存变量 。
    * 缺点：指令字长变换很大，特别是 3  个操作数指令。而且每条指令完成的工作也差别很大。对存储器的频率访问会 使存储器成为瓶颈 。这种类型的指令系统现在 已经不用 了



#### MIPS 指令集

概述

* Load/Store 型指令集结构
* 注重指令流水效率
* 简化指令的译码
* 高效支持编译器



寄存器

* 32  个 32  位的通用寄存器（ GPRs ）,寄存器 R0  的内容全恒为 0 

  <img src="https://img-blog.csdnimg.cn/20201225150343310.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="20%" />

* 32  个 32  位浮点寄存器（ FPRs ）。单精度浮点数表示和双精度浮点数表示。

  <img src="https://img-blog.csdnimg.cn/20201225150408819.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="20%" />





 寻址方式

* 寄存器寻址；如 ADD R1,R2,R3
* 立即值寻址；如 ADD R1,R2,#42
* 偏移寻址； 如 ADD R1,R2,40(R3)
* 寄存器间接寻址。存储器地址宽度为 32  位。如 ADD R1,R2,40(R3)



 指令格式

* I 类型指令

  * 字节 、 半字 、 字的载入和存储 ；

  <img src="https://img-blog.csdnimg.cn/20201225150433780.png" width="40%" />

* R 类型指令

  * 寄 存 器 － 寄 存 器 器 ALU 操 作 
  * 函数对数据的操作进行编码 ： 加 、 减 、 ... ；
  * 对特殊寄存器的读/ 写和移动

  <img src="https://img-blog.csdnimg.cn/20201225150455960.png" width="40%" />

* J  类型指令

  * 跳转 ，跳转并链接 ，从异常（exception ）处自陷和返 回

  <img src="https://img-blog.csdnimg.cn/20201225150515519.png" width="40%" />



 操作类型

* Load 和 和 Store  操作；
* ALU  操作；
* 分支和跳转操作；
* 浮点操作


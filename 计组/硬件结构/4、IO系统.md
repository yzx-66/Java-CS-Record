# 概述

**发展概况**

* 早期
  * 分散连接
  * CPU  和 I/O 设备 串行工作  程序查询方式
* 接口模块和 DMA  阶段
  * 总线连接
  * CPU  和 I/O 设备 并行工作：中断方式、DMA 

* 具有通道结构的阶段
* 具有 I/O  处理机的阶段



**输入输出系统的组成**

* I/O  软件

  * I/O  指令

    * CPU  指令的一部分

    <img src="https://img-blog.csdnimg.cn/20201219122521696.png" width="30%" height="50%"  />

  * 通道指令

    * 通道自身的指令
    * 指出数组的首地址、传送字数、操作命令
    * 如 IBM/370  通道指令为 64  位

* I/O  硬件

  *  设备  I/O接口
  *  设备  设备控制器  通道



**I/O  设备与主机的联系方式**

* I/O  设备编址方式

  * 统一编址：用取数、存数指令
  * 不统一编址：有专门的  I/O  指令

* 设备选址

  * 用 设备选择电路 识别是否被选中

* 传送方式

  * 串行
  * 并行

* 联络方式

  * 立即响应

  * 异步工作采用应答信号

    <img src="https://img-blog.csdnimg.cn/20201219122543744.png" width="50%" height="50%"  />

  * 同步工作采用同步时标

    <img src="https://img-blog.csdnimg.cn/20201219122611211.png" width="50%" height="50%"  />

  

**I/O  设备与主机的连接方式**

* 辐射式连接

  * 每台设备都配有一套控制线路和一组信号线
  * 不便于增删设备

  <img src="https://img-blog.csdnimg.cn/20201219122631105.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="30%" height="50%"  />

* 总线连接

  * 便于增删设备



**信息传送的控制方式**

* 程序查询方式

  * CPU  和 I/O  串行工作踏步等待

    <img src="https://img-blog.csdnimg.cn/20201219122657939.png" width="30%" height="50%"  />

  * 流程

    <img src="https://img-blog.csdnimg.cn/20201219122724144.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="30%" height="50%"  />

* 程序中断方式

  * I/O  工作

    * 自身准备，CPU 不查询
    * 与主机交换信息，CPU   暂停现行程序

  * CPU  和 I/O  部分的并行工作（无踏步等待）

    ​	<img src="https://img-blog.csdnimg.cn/20201219122802947.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="35%" height="50%"  />

  

  

  

  * 流程

    <img src="https://img-blog.csdnimg.cn/20201219122830377.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" height="50%" />

* DMA 方式

  * 主存和 I/O  之间有一条直接数据通道
  * 不中断现行程序
  * 周期挪用（周期窃取）
  * CPU  和 I/O 并行工作

  <img src="https://img-blog.csdnimg.cn/20201219122900499.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" height="50%" />

  

* 三种方式的 CPU 工作效率比较

  * 程序查询方式

    <img src="https://img-blog.csdnimg.cn/20201219122926848.png" width="50%" height="50%"  />

  * 程序中断方式

    <img src="https://img-blog.csdnimg.cn/20201219122952602.png" width="50%" height="50%"  />

  * DMA方式

    <img src="https://img-blog.csdnimg.cn/20201219123013943.png" width="50%" height="50%"  />

  * 自治能力

    <img src="https://img-blog.csdnimg.cn/20201219123039311.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="25%" height="50%"  />





# 外部设备

**概述**

<img src="https://img-blog.csdnimg.cn/20201219123102679.png" width="50%" height="50%"  />

* 外部设备大致分三类
  * 人机交互设备：键盘、鼠标、打印机、显示器
  * 计算机信息存储设备：磁盘、光盘、磁带
  * 机 机通信设备  调制解调器等



**输入设备**

* 键盘

  * 按键
  * 判断哪个键按下
  * 将此键翻译成 ASCII  码  （编码键盘法）
* 鼠标
  * 机械式 金属球 电位器
  * 光电式 光电转换器
* 触摸屏



**输出设备**

* 显示器
  * 字符显示
  * 图形显示
  * 图像显示
* 打印机
  * 击打式：点阵式（逐字、逐行）
  * 非击打式：激光（逐页）、喷墨（逐字）



**其他**

* A/D 、D/A：模拟/ 数字（数字/ 模拟）转换器
* 终端：由键盘和显示器组成，完成显示控制与存储、键盘管理及通信控制
* 汉字处理：汉字输入、汉字存储、汉字输出





# I/O接口

**功能和组成**

* 总线连接


  <img src="https://img-blog.csdnimg.cn/20201219123128449.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="30%" height="50%"  />

  *  设备选择线
  *  数据线
  *  命令线
  *  状态线

* 功能组成

  * 主干组成

    <img src="https://img-blog.csdnimg.cn/20201219123151471.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />

  * 其余核心部件

    <img src="https://img-blog.csdnimg.cn/20201219123214734.png" width="25%" height="50%"  />

* 基本组成

  <img src="https://img-blog.csdnimg.cn/20201219123239539.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" height="50%"  />



**接口类型**

*   按数据  传送方式  分类
    * 并行接口：Intel 8255
    * 串行接口：Intel 8251
*   按功能  选择的灵活性  分类
    * 可编程接口：Intel 8255 、 Intel 8251
    * 不可编程接口：Intel 8212
*   按  通用性  分类
    * 通用接口：Intel 8255 、 Intel 8251
    * 专用接口：Intel 8279 、 Intel 8275
*   按数据传送的  控制方式  分类
    * 中断接口：Intel 8259
    * DMA  接口：Intel 8257

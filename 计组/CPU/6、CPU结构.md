# CPU 的结构

### CPU  的功能

* 控制器的功能
  * （指令控制）取指令
  * （操作控制）分析指令
  * （时间控制）执行指令，发出各种操作命令
  * （时间控制）控制程序输入及结果的输出
  * （处理中断）总线管理
  * （数据加工）处理异常情况和特殊请求
* 运算器的功能
  * 实现算术运算和逻辑运算



### CPU 结构框图

* CPU  与系统总线

  <img src="https://img-blog.csdnimg.cn/20201219140737289.png" width="25%" height="50%"  />

* 框图

  <img src="https://img-blog.csdnimg.cn/20201219140804947.png" width="30%" height="50%"  />



### CPU  的寄存器

* 用户可见寄存器
  * 通用寄存器：存放操作数
    * 可作  某种寻址方式所需的  专用寄存器
  * 数据寄存器：存放操作数 （满足各种数据类型）
    * 两个寄存器拼接存放双倍字长数据
  * 地址寄存器：存放地址 ，其位数应满足最大的地址范围
    * 用于特殊的寻址方式 段基值 栈指针
  * 条件码寄存器：存放条件码 ，可作程序分支的依据
    * 如  正、负、零、溢出、进位等
* 控制和状态寄存器
  * 控制寄存器：PC -> MAR ->   M  ->  MDR ->IR
    * 控制 CPU  操作
    * 其中 MAR 、MDR 、IR 用户不可见、PC 用户可见
  * 状态寄存器
    * 状态寄存器：存放条件码
    * PSW  寄存器：存放程序状态字



### 控制单元 CU  和中断系统

* CU 产生全部指令的微操作命令序列
  * 组合逻辑设计：硬连线逻辑
  * 微程序设计：存储逻辑
* 中断系统



### ALU
ALU 电路

* 组合逻辑电路

  * Ki 不同取值 -> Fi 不同

  <img src="https://img-blog.csdnimg.cn/20201219133429952.png" width="20%" height="50%"  />

* 四位 ALU 74181

  * M = 0 算术运算
  * M = 1 逻辑运算
  * S3 ~ S0 不同取值，可做不同运算



快速进位链

* 并行加法器

	* 电路

  <img src="https://img-blog.csdnimg.cn/20201219133455805.png" width="40%" height="40%"  />

	* 运算

  <img src="https://img-blog.csdnimg.cn/20201219133517845.png" width="40%" height="40%"  />



1、串行进位链

* 电路

  <img src="https://img-blog.csdnimg.cn/20201219133548992.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />

* 运算

  <img src="https://img-blog.csdnimg.cn/20201219133611176.png" width="40%" height="40%"  />



2、并行进位链（先行进位，跳跃进位）


  * 电路

    <img src="https://img-blog.csdnimg.cn/20201219133637659.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />

  * 运算

    <img src="https://img-blog.csdnimg.cn/20201219133657491.png" width="40%" height="50%"  />

  

2.1、并行进位链：单重分组跳跃进位链

  * n  位全加器分若干小组，小组中的进位同时产生，小组与小组之间采用串行进位

  * 电路（以 n = 16  为例）

    <img src="https://img-blog.csdnimg.cn/20201219133716395.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%" />

2.2、并行进位链：双重分组跳跃进位链

  * n  位全加器分若干大组，大组中又包含若干小组。每个大组中小组的最高位进位同时产生。

  * 大组与大组之间采用串行进位。

  * 分组

    <img src="https://img-blog.csdnimg.cn/20201219133737932.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />

  * 大组进位分析

    <img src="https://img-blog.csdnimg.cn/20201219133804146.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />

  

  * 大组进位线路（以第 2  大组为例）

    <img src="https://img-blog.csdnimg.cn/20201219133828440.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_7" width="40%" height="50%" />

  * 小组进位线路（以第 8  小组为例）

    <img src="https://img-blog.csdnimg.cn/20201219133846778.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%" />

    

  * n =16 双重分组跳跃进位链

    <img src="https://img-blog.csdnimg.cn/20201219133905585.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />

    * 大组进位线路

      <img src="https://img-blog.csdnimg.cn/20201219133926947.png"  />

    * 小组进位电路

      <img src="https://img-blog.csdnimg.cn/20201219134008113.png" />

    

  * n =32  双重分组跳跃进位链

    <img src="https://img-blog.csdnimg.cn/20201219134036818.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />

    * 大组进位线路

      <img src="https://img-blog.csdnimg.cn/20201219134059260.png" />

    * 小组进位电路

      <img src="https://img-blog.csdnimg.cn/20201219134134352.png"  />







# 指令周期
### 基本概念

* 指令周期

  <img src="https://img-blog.csdnimg.cn/20201219140832250.png" width="35%" height="50%"  />

  * 取出并执行一条指令所需的全部时间

  * 完成一条指令

    * （取指周期）取指、分析
    * （执行周期）执行

  * 每条指令的指令周期不同

    <img src="https://img-blog.csdnimg.cn/20201219140856546.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_7" width="50%" height="50%"  />

* 具有间接寻址的指令周期

  <img src="https://img-blog.csdnimg.cn/20201219140924733.png" width="40%" height="50%" />

* 带有中断周期的指令周期

  <img src="https://img-blog.csdnimg.cn/20201219140951240.png" width="40%" height="50%" />

* 指令周期流程

  <img src="https://img-blog.csdnimg.cn/20201219141011993.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_7" width="30%" height="50%"  />

* CPU  工作周期的标志

  * 有四种性质
    * （取指周期）取指令
    * （间址周期）取地址
    * （间址周期）存取 操作数或结果
    * （中断周期）存程序断点

  <img src="https://img-blog.csdnimg.cn/20201219141034953.png" width="50%" height="50%" />



### 指令周期的数据流

* 取指周期数据流

  <img src="https://img-blog.csdnimg.cn/20201219141100711.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="35%" height="50%"  />

* 间址周期数据流

  <img src="https://img-blog.csdnimg.cn/2020121914113043.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="35%" height="50%"  />

* 执行周期数据流

  * 不同指令的执行周期数据流不同

* 中断周期数据流

  <img src="https://img-blog.csdnimg.cn/20201219141152697.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="35%" height="50%" />


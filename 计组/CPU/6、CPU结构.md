# CPU 的结构

**CPU  的功能**

* 控制器的功能
  * （指令控制）取指令
  * （操作控制）分析指令
  * （时间控制）执行指令，发出各种操作命令
  * （时间控制）控制程序输入及结果的输出
  * （处理中断）总线管理
  * （数据加工）处理异常情况和特殊请求
* 运算器的功能
  * 实现算术运算和逻辑运算



**CPU 结构框图**

* CPU  与系统总线

  <img src="https://img-blog.csdnimg.cn/20201219140737289.png" width="25%" height="50%"  />

* 框图

  <img src="https://img-blog.csdnimg.cn/20201219140804947.png" width="30%" height="50%"  />



**CPU  的寄存器**

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



**控制单元 CU  和中断系统**

* CU 产生全部指令的微操作命令序列
  * 组合逻辑设计：硬连线逻辑
  * 微程序设计：存储逻辑
* 中断系统



**ALU** 
运算器（略）



# 指令周期

**基本概念**

* 指令周期

  <img src="https://img-blog.csdnimg.cn/20201219140832250.png" width="35%" height="50%"  />

  * 取出并执行一条指令所需的全部时间

  * 完成一条指令

    * （取指周期）取指、分析
    * （执行周期）执行

  * 每条指令的指令周期不同

    <img src="https://img-blog.csdnimg.cn/20201219140856546.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_7" width="50%" height="50%"  />

* 具有间接寻址的指令周期

  <img src="https://img-blog.csdnimg.cn/20201219140924733.png" width="50%" height="50%" />

* 带有中断周期的指令周期

  <img src="https://img-blog.csdnimg.cn/20201219140951240.png" width="50%" height="50%" />

* 指令周期流程

  <img src="https://img-blog.csdnimg.cn/20201219141011993.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_7" width="30%" height="50%"  />

* CPU  工作周期的标志

  * 有四种性质
    * （取指周期）取指令
    * （间址周期）取地址
    * （间址周期）存取 操作数或结果
    * （中断周期）存程序断点

  <img src="https://img-blog.csdnimg.cn/20201219141034953.png" width="50%" height="50%" />



**指令周期的数据流**

* 取指周期数据流

  <img src="https://img-blog.csdnimg.cn/20201219141100711.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />

* 间址周期数据流

  <img src="https://img-blog.csdnimg.cn/2020121914113043.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />

* 执行周期数据流

  * 不同指令的执行周期数据流不同

* 中断周期数据流

  <img src="https://img-blog.csdnimg.cn/20201219141152697.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%" />


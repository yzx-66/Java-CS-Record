# 程序查询方式

**流程**

* 查询流程

  * 单个设备


    <img src="https://img-blog.csdnimg.cn/20201219123922492.png" width="15%" height="50%" />

  * 多个设备

    <img src="https://img-blog.csdnimg.cn/20201219124021355.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="25%" height="50%"  />

* 程序流程

  <img src="https://img-blog.csdnimg.cn/20201219124045336.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="25%" height="50%"  />



**接口电路**

* 以输入为例

  <img src="https://img-blog.csdnimg.cn/20201219124109702.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />



# 程序中断方式

**中断的概念**

<img src="https://img-blog.csdnimg.cn/20201219124143490.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="30%" height="50%"  />



**I/O 中断的产生（以打印机为例）**

<img src="https://img-blog.csdnimg.cn/20201219124203996.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />



**接口电路**

* 配置中断请求触发器和中断屏蔽触发器

  <img src="https://img-blog.csdnimg.cn/20201219124226828.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="35%" height="50%"  />

  * INTR 中断请求触发器，INTR = 1 有请求
  * MASK 中断屏蔽触发器，MASK = 1 被屏蔽
  * D 完成触发器

* 排队器

  * 硬件方式：在 CPU 内或在接口电路中（链式排队器）
  * 软件方式：通过设置屏蔽字

  <img src="https://img-blog.csdnimg.cn/20201219124252402.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" height="50%"   />

* 中断向量地址形成部件

  ​	<img src="https://img-blog.csdnimg.cn/20201219124314448.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="30%" height="50%"  />

  

  

  

  

  

  * 硬件向量法：由 硬件 产生 向量地址，再由 向量地址 找到 入口地址

    <img src="https://img-blog.csdnimg.cn/20201219124351119.png" width="30%" height="50%"  />

  * 软件产生方式：软件查询法

* 接口电路的基本组成

  <img src="https://img-blog.csdnimg.cn/20201219124423705.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" height="50%"  />





**I/O   中断处理过程**

* CPU  响应中断的条件和时间

  * 条件
    * 允许中断触发器 EINT = 1
    * 用  开中断  指令将 EINT  置  “1 ”
    * 用  关中断  指令将 EINT 置 置“ “ 0 ”  或硬件  自动复位
  * 时间
    * 当 D = 1（ 随机）且 MASK = 0  时
    * 在每条指令执行阶段的结束前，CPU  发  中断查询信号 （将 INTR 置 置“1”） 

* I/O 中断处理过程（接口）

  * 以输入为例

  <img src="https://img-blog.csdnimg.cn/20201219124450747.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" height="50%"  />



**中断服务程序流程**

* 中断服务程序的流程

  * 保护现场
    * 程序断点的保护：中断隐指令完成
    * 寄存器内容的保护：进栈指令
  * 中断服务
    * 对不同的 I/O 设备具有不同内容的设备服务
  * 恢复现场
    * 出栈指令
  * 中断返回
    * 中断返回指令

* 单重中断和多重中断

  * 单重 中断：不允许中断  现行的  中断服务程序
  * 多重 中断：允许级别更高  的中断源中断  现行的  中断服务程序

* 单重中断和多重中断的服务程序流程

  * 单重

    <img src="https://img-blog.csdnimg.cn/20201219124519879.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_7" width="25%" height="50%" />

  * 多重

    <img src="https://img-blog.csdnimg.cn/20201219124548630.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="25%" height="50%"  />

* 主程序和服务程序抢占 CPU

  <img src="https://img-blog.csdnimg.cn/20201219124642269.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" height="50%"  />

  * 宏观 上 CPU 和 I/O 并行 工作
  * 微观 上 CPU 中断现行程序 为 I/O 服务



​	

# DMA 方式

**特点**

* DMA  和程序中断两种方式的数据通路

  <img src="https://img-blog.csdnimg.cn/20201219124710651.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />

* DMA  与主存交换数据的三种方式

  * 停止 CPU  访问主存

    * 控制简单
    * CPU  处于不工作状态或保持状态
    * 未充分发挥 CPU  对主存的利用率

    <img src="https://img-blog.csdnimg.cn/20201219124734325.png" width="50%" height="50%"  />

  * 周期挪用（或周期窃取）

    * DMA  访问主存有三种可能
    * CPU  此时不访存
      * CPU  正在访存
      * CPU  与 DMA  同时请求访存（此时 CPU  将总线控制权让给 DMA）

    <img src="https://img-blog.csdnimg.cn/2020121912475715.png" width="50%" height="50%"  />

  * DMA  与 CPU  交替访问

    * CPU  工作周期
    * C1  专供 DMA  访存
      * C2  专供 CPU  访存

  * 不需要  申请建立和归还总线的使用权

    <img src="https://img-blog.csdnimg.cn/2020121912482089.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" height="50%"  />

* DMA  方式与程序中断方式的比较

  <img src="https://img-blog.csdnimg.cn/2020121912484180.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />





**DMA   接口的功能和组成**

* DMA  接口与系统的连接方式

  * 具有公共请求线的 DMA  请求

    <img src="https://img-blog.csdnimg.cn/2020121912490577.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" height="50%"  />

  * 独立的 DMA  请求

    <img src="https://img-blog.csdnimg.cn/20201219124934242.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" height="50%"  />

* DMA  接口功能

  * 向 CPU  申请 DMA  传送
  * 处理总线  控制权的转交
  * 管理  系统总线、 控制  数据传送
  * 确定  数据传送的  首地址和长度，修正  传送过程中的数据  地址  和  长度
  * DMA  传送结束时， 给出操作完成信号

* DMA 接口组成

  <img src="https://img-blog.csdnimg.cn/20201219124954764.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" height="50%"  />







**DMA  的工作过程**

* DMA 传送过程：预处理、数据传送、后处理

  * 预处理：通过几条输入输出指令预置如下信息

    * 通知 DMA  控制逻辑传送方向（入/ 出）
    * 设备地址 DMA  的 DAR
    * 主存地址 DMA  的 AR
    * 传送字数 DMA  的 WC

* DMA  传送过程示意

  * CPU
    <img src="https://img-blog.csdnimg.cn/20201219125022909.png" width="20%" height="50%"  />    


  * 数据传送

    <img src="https://img-blog.csdnimg.cn/20201219125047595.png" width="20%" height="50%"  />

* 数据传送过程

  <img src="https://img-blog.csdnimg.cn/20201219125112706.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_7" width="50%" height="50%" />

    

* 后处理（由中断服务程序完成）

  * 校验送入主存的数是否正确
  * 是否继续用 DMA
  * 测试传送过程是否正确，错则转诊断程序





**DMA 接口的类型**

* 选择型  

  * 在  物理上  连接  多个  设备
  * 在  逻辑上  只允许连接  一个  设备

  <img src="https://img-blog.csdnimg.cn/20201219125141463.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="40%" height="50%"  />

* 多路型 

  * 在  物理上  连接  多个  设备

  * 在  逻辑上  允许连接  多个  设备同时工作

    <img src="https://img-blog.csdnimg.cn/20201219125203339.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" height="50%" />

  * 多路型 DMA  接口的工作原理

    <img src="https://img-blog.csdnimg.cn/20201219125224790.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" width="50%" height="50%"  />

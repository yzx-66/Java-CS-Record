### IPsec 

VPN 关键技术

* 隧道技术
* 数据加密
* 身份认证
* 密钥管理
* 访问控制
* 网络管理



隧道技术

* 构建VPN的核心技术
* 隧道：通过Internet提供安全的点到点(或端到端)的数据传输“安全通道”
* 实质上是一种封装
  * VPN隧道利用隧道协议对通过隧道传输的数据进行封装
  * 使数据安全穿越公共网络(通常是Internet)
  * 通过加密和认证以确保安全
  * 数据包进入隧道时，由VPN封装成IP数据报
  * 通过隧道在Internet上安全传输
  * 离开隧道后，进行解封装，数据便不再被保护



常见VPN隧道协议：

* 第二层隧道：PPTP、L2TP
  * 主要用于远程客户机访问局域网方案
* 第三层隧道：IPSec
  * 主要用于网关到网关、或网关到主机方案
  * 不支持远程拨号访问



典型VPN 实现技术

* IPSec：最安全、适用面最广
* SSL：具有高层安全协议的优势
* L2TP：最好的实现远程接入VPN的技术
* 典型VPN技术结合：IPSec与SSL、IPSec与L2TP



#### 概述

IPsec 体系结构

<img src="https://img-blog.csdnimg.cn/20210112204806792.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210109183353730"  width="45%"/>





IPsec 服务

* 机密性(confidentiality )
* 数据完整性(data integrity)
* 源认证/鉴别(origin authentication)
* 重放攻击预防(replay attack prevention)
* 提供不同服务模型的两个协议:
  * AH
  * ESP



IPsec 的模式

* 传输(transport)模式
  * IPsec数据报的发送与接收均由端系统完成
  * 主机是IPsec感知的(IPsec-ware)
* 隧道(tunneling)模式
  * 边缘路由器是IPsec感知的(IPsec-ware)



两个IPsec 协议

* 提供IPsec服务的两个协议:
  * AH：在IP数据报文头中的协议号为51
  * ESP：在IP数据报文头中的协议号为50
* 认证头协议AH(Authentication Header)
  * 提供源认证/鉴别和数据完整性检验，但不提供机密性
* 封装安全协议ESP(Encapsulation Security Protocol)
  * 提供源认证/鉴别、数据完整性检验以及机密性
  * 比AH应用更广泛



IPsec 模式与协议的4 种组合

* 传输模式 AH

  <img src="https://img-blog.csdnimg.cn/20210112204835488.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210109184643954"  width="35%"/>

* 传输模式 ESP

  <img src="https://img-blog.csdnimg.cn/20210112204856977.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210109184755353"  width="50%" />

* 隧道模式 AH

  <img src="https://img-blog.csdnimg.cn/20210112204941880.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210109184708316"  width="40%" />

* 隧道模式 ESP（最普遍、最重要）

  <img src="https://img-blog.csdnimg.cn/20210112205018267.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210109184817111"  width="40%" />

    * ESP尾部：填充以便应用分组密码



所有组合的都有的核心属性

* 认证字段，基于共享的秘密MAC密钥

* SPI，接收实体基于此知道该做什么
* 序列号，抵抗重放攻击
  * 对于新SA，发送方初始化序列号为0
  * 每次通过SA发送数据报:
    * 发送方增加序列号计数器（加1）
    * 将计数器值置于序列号字段
  * 目的：
    * 预防嗅探与回放分组攻击
    * 接收重复的、已认证的IP分组，会破坏正常服务
  * 方法：
    * 接收方检验分组重复
    * 无需记录所有已接收分组；而是利用一个窗口



小结

* IPsec 的传输模式

  <img src="https://img-blog.csdnimg.cn/20210112204644865.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210109184924612"  width="40%" />

* IPsec 的隧道模式

  <img src="https://img-blog.csdnimg.cn/20210112204710953.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210109184950842"  width="40%" />






#### 安全关联

安全关联(SA)

* 发送数据前，从发送实体到接收实体之间需要建立安全关联SA (security association)
  * **SA是单工的: 单向**
* 发送实体与接收实体均需维护SA的状态信息
  * 回顾: TCP连接的端点也需要维护状态信息
  * IP是无连接的；IPsec是面向连接的！



安全关联主要参数：

* 安全参数索引(SPI)：32位SA唯一标识(ID)
* 加密密钥、认证密钥
* 密码算法标识
* 序列号(32位) ：抗重放攻击
* 抗重播窗口：接收方使用滑动窗口检测恶意主机重放数据报
* 生存周期：规定SA的有效使用周期
* 运行模式：传输模式或隧道模式
* IPSec隧道源、目的地址



安全策略数据库(SPD)

* Security Policy Database (SPD)
* 安全策略(SP)：定义了对什么样的数据流实施什么样的安全处理
  * 应用IPSec、绕过、丢弃
* 安全策略组成了SPD，每个记录就是一条SP
  * 提取关键信息填充到一个称为“选择符”的结构
  * 包括目标IP、源IP、传输层协议、源和目标端口等
  * 利用选择符去搜索SPD，检索匹配的SP
* 安全处理需要的参数存储在SP指向的SA结构



安全关联数据库(SAD)

* IPsec端点将SA状态保存在安全关联数据库SAD (security association database)中
  * 在处理IPsec数据报时，定位这些信息
* 对于n个销售人员，1个分支机构的VPN，总部的路由器R1的SAD中存储 2 + 2n 条 SAs
* 当发送IPsec数据报时，R1访问SAD，确定如何处理数据报
* 当IPsec数据报到达R2
  * R2检验IPsec数据报中的SPI
  * 利用SPI检索SAD
  * 处理数据报





#### 处理过程

* 隧道模式 ESP

  <img src="https://img-blog.csdnimg.cn/20210112204741536.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210109185203969"  width="40%" />





R1:  将原IP 数据报转换为 IPsec 数据报

* 检索SPD，确定处理策略
* 检索SAD，确定SA
* 在原IP数据报(包括原IP首部域！)后面附加“ESP尾部”.
* 利用SA定义的算法与密钥，加密上述结果.
* 在加密结果前面附加“ESP头”，创建“enchilada”.
* 针对整个enchilada，利用SA定义的算法与密钥，创建报文认证码MAC；
* 在enchilada后面附加MAC，构成载荷(新IP数据报载荷)；
* 构造全新的IP头，包含所有经典的IPv4首部字段；
* 将新IP头附加在载荷的前面



R2:  解封 IPsec

* 从原始IP数据报中提取选择符，并搜索SPD，确定处理策略
* 丢弃或转入系统IP协议栈进行后继处理
* 判断是否为IPsec数据报
* 从头部提取\<SPI>，并检索SAD
* 若找不到SA，则触发IKE或丢弃包；
* 若找到，则根据SA解封数据报，得到原始IP数据报

### SSL

Web 安全实现方式

* 基于应用层实现Web 安全

  <img src="https://img-blog.csdnimg.cn/20210112204150468.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_7" alt="image-20210106114605034"  width="30%" />

* 基于传输层实现Web 安全

  * SSL或TLS可作为基础协议栈的组成部分，对应用透明（也可直接嵌入到浏览器中使用）
  * 使用SSL或TLS后，传送的应用层数据会被加密（保证通信的安全）

  <img src="https://img-blog.csdnimg.cn/20210112204212337.png" alt="image-20210106114707247"  width="25%" />

* 基于网络层实现Web安全

  * IPSec提供端到端（主机到主机）的安全机制（通用解决方案）
  * 各种应用程序均可利用IPSec提供的安全机制（减少了安全漏洞的产生）

  <img src="https://img-blog.csdnimg.cn/2021011220423961.png" alt="image-20210106114824661"  width="25%" />






SSL 和TCP/IP

<img src="https://img-blog.csdnimg.cn/20210112204259449.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210106115044052"  width="30%" />

* SSL 为网络应用提供应用编程接口 (API)
* C 语言和Java 语言的 SSL 库/



#### 简化 SSL

简化的 SSL 流程:

* 握手(handshake):
  * Alice和Bob利用他们的证书、私钥认证（鉴别）彼此，以及交换共享密钥
* 密钥派生(key derivation):
  * Alice和Bob利用共享密钥派生出一组密钥
* 数据传输(data transfer):
  * 待传输数据分割成一系列记录
* 连接关闭(connection closure): 
  * 通过发送特殊消息，安全关闭连接



密钥派生

* 不同加密操作使用不同密钥会更加安全
  * 例如：报文认证码(MAC)密钥和数据加密密钥
* 4个密钥：
  * Kc = 用于加密客户向服务器发送数据的密钥
  * Mc = 用于客户向服务器发送数据的MAC密钥
  * Ks = 用于加密服务器向客户发送数据的密钥
  * Ms =用于服务器向客户发送数据的MAC密钥
* 通过密钥派生函数(KDF)实现密钥派生
  * 提取主密钥和（可能的）一些额外的随机数，生成密钥



数据记录1.0：分段

* 为什么不直接加密发送给TCP的字节流?

  * MAC放到哪儿？如果放到最后，则只有全部数据收全才能进行完整性认证。
  * e.g , 对于即时消息应用， 在显示一段消息之前，如何针对发送的所有字节进行完整性检验？

* 方案：将字节流分割为一系列记录

  * 每个记录携带一个MAC
  * 接收方可以对每个记录进行完整性检验

* 问题：对于每个记录， 接收方需要从数据中识别出MAC

  * 需要采用变长记录

  <img src="https://img-blog.csdnimg.cn/20210112204322635.png" alt="image-20210106120220810"  width="30%" />





数据记录2.0：增加序列号

* 问题: 攻击者可以捕获和重放记录或者重新排序记录
  * 解决方案：在MAC中增加序列号，MAC = MAC(Mx , sequence||data)
  * 注意：记录中没有序列号域
* 问题: 攻击者可以重放所有记录
  * 解决方案: 使用一次性随机数(nonce)



数据记录3.0：再增加控制信息

* 问题：截断攻击

  * 攻击者伪造TCP连接的断连段，恶意断开连接
  * 一方或双方认为对方已没有数据发送

* 解决方案：记录类型, 利用一个类型的记录专门用于断连

  * type 0 用于数据记录；type 1 用于断连

* **MAC = MAC(Mx , sequence||type||data)**

  <img src="https://img-blog.csdnimg.cn/20210112204345807.png" alt="image-20210106121013692"  width="30%"/>





简化的SSL :  总结

<img src="https://img-blog.csdnimg.cn/20210112204409442.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210106121736442"  width="40%" />



#### SSL 协议

简化的SSL 不完整

* 每个域多长？
* 采用哪种加密协议？
* 需要协商吗？
  * 允许客户与服务器支持不同加密算法
  * 允许客户与服务器在数据传输之前共同选择特定的算法



SSL 协议栈

* 介于HTTP与TCP之间的一个可选层

  * 绝大多数应用层协议可直接建立在SSL之上

* SSL不是一个单独的协议，而是两层协议

  <img src="https://img-blog.csdnimg.cn/2021011220443461.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210106122109991"  width="40%" />





SSL 密码组

* 密码组(cipher suite)

  * 对称加密算法(symmetricencryption algorithm)

    * 常见的SSL 对称密码：

      DES（分组密码）、3DES（分组密码）、RC2 – Rivest Cipher 2（分组密码）、RC4 – Rivest Cipher 4（流密码）

  * 公开密钥算法(public-key algorithm)

    * SSL 公开密钥加密：RSA

  * MAC算法

    * 报文认证码 MAC (Message Authentication Code)：

      报文m + 认证密钥s + 密码散列函数H -> 扩展报文 (m, H(m+s))

* SSL支持多个密码组

* 协商(negotiation): 客户与服务器商定密码组

  * 客户提供选项(choice)
  * 服务器挑选其一





SSL 更改密码规格协议 (Change Cipher Spec Protocol)

* 更新当前连接的密钥组
  * 标志着加密策略的改变
* 位于SSL记录协议之上
* ContentType=20
* 协议只包含一条消息（一个值为1的字节）





SSL 警告协议 (Alert Protocol)

* Alert消息：
  * 当握手过程或数据加密等出错或发生异常时，为对等实体传递SSL警告或终止当前连接
* 位于SSL记录协议之上
* ContentType=21
* 协议包含两个字节：警告级别和警告代码



SSL 握手协议 (Handshake Protocol)

* 协商结果是SSL记录协议的基础，ContentType=22
* SSL v3.0 的握手过程用到三个协议：握手协议、更改密码规格协议和警告协议
* 目的：
  * 服务器认证/鉴别
  * 协商: 商定加密算法
  * 建立密钥
  * 客户认证/鉴别(可选)





SSL 记录协议(Record Protocol)

* 描述SSL信息交换过程中的记录格式
* 所有数据（含SSL握手信息）都被封装在记录中
* 一个记录由两部分组成：记录头和数据



SSL记录协议的操作步骤：

* 将数据分段成可操作的数据块
* 对分块数据进行数据压缩
* 计算MAC值
* 对压缩数据及MAC值加密
* 加入SSL记录头
* 在TCP中传输



SSL 记录格式

* 生成过程

  <img src="https://img-blog.csdnimg.cn/2021011220445651.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210106132520312"  width="40%" />

  * data fragment ：每个SSL 片段为 $2^{14}$ 字节 (16KB)

  * MAC：包括 data fragment 的报文摘要、序列号、MAC 密钥 Mx（客户端用Mc、服务端用Ms）
  * encrypted data and MAC：经过**对称加密**的 data fragment 和 MAC（客户端用Kc、服务端用Ks）
  * 记录头(record header): 内容类型(ContentType);  版本;  长度

* 最终记录

  <img src="https://img-blog.csdnimg.cn/20210112204007962.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210106133048834"  width="40%" />

  * 数据和MAC 是加密的 (对称密钥加密算法)

  

  





#### 握手过程

步骤

* 1、客户发送其支持的算法列表，以及客户一次随机数(nonce)
* 2、服务器从算法列表中选择算法，并发回给客户：选择 + 证书 + 服务器一次随机数
* 3、客户验证证书，提取服务器公钥，生成预主密钥 (pre_master_secret)，并利用服务器的公钥加密预主密钥，发送给服务器
* 4、客户与服务器基于预主密钥和一次随机数分别独立计算加密密钥和MAC密钥
* 5、客户发送一个针对所有握手消息的MAC
* 6、服务器发送一个针对所有握手消息的MAC



为什么使用两个一次随机数？

* 假设Trudy嗅探Alice与Bob之间的所有报文
* 第二天，Trudy与Bob建立TCP连接，发送完全相同的记录序列
  * Bob(如Amazon)认为Alice对同一产品下发两个分离的订单
* 解决方案: Bob为每次连接发送完全不同的一次随机数
  * 确保两天的加密密钥不同
  * Trudy的报文将无法通过Bob的完整性检验





密钥派生

* 客户一次数、服务器一次数和预主密钥输入伪随机数发生器
  * 产生主密钥MS
* 主密钥和新一次随机数输入另一个随机数发生器：“密钥块(key block)”
* 密钥块“切片”:
  * 客户MAC密钥
  * 服务器MAC密钥
  * 客户加密秘钥
  * 服务器加密秘钥
  * 客户初始向量(IV)
  * 服务器初始向量(IV)
* 即：client 和 server 都有这数值完全相同的六个切块





最后2步的意义：保护握手过程免遭篡改

* 客户提供的算法，安全性有强、有弱
  * 明文传输
* 中间人攻击可以从列表中删除安全性强的算法
* 最后2步可以预防这种情况发生
  * 最后两步传输的消息是加密的 





SSL 握手

* 消息及参数

  <img src="https://img-blog.csdnimg.cn/20210112204026433.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210106130150526"  width="40%"/>

* 工作过程

  * \* 为可选步骤

  <img src="https://img-blog.csdnimg.cn/2021011220404737.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="image-20210106130254566"  width="60%"/>





实际的 SSL 连接（ **SSL四次握手**）

<img src="https://img-blog.csdnimg.cn/20210112204122163.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70" alt="img"  width="45%" />

* 一次握手：

  * 步骤 1： 客户端通过发送 Client Hello 报文开始 SSL通信。 报文中包含客户端支持的 SSL的指定版本、 加密组件（Cipher Suite） 列表（所使用的加密算法及密钥长度等） 。

* 二次握手：

  * 步骤 2： 服务器可进行 SSL通信时， 会以 Server Hello 报文作为应答。 和客户端一样， 在报文中包含 SSL版本以及加密组件。 **服务器的加密组件内容是从接收到的客户端加密组件内筛选出来的**。

  * 步骤 3： 之后服务器发送 Certificate 报文。 报文中包含公开密钥证书。
* 步骤 4： 最后服务器发送 Server Hello Done 报文通知客户端， 来标志 Server 消息发送完毕。

* 三次握手：

  * 步骤 5： 客户端以 Client Key Exchange 报文作为回应。 报文中包含通信加密中使用的一种被称为 Pre-mastersecret 的随机密码串。 **该报文已用步骤 3 中的公开密钥进行加密**。
  * **Server收到这个报文就会开始密钥的派生**
  * 步骤 6： 接着客户端继续发送 Change Cipher Spec 报文。 该报文会提示服务器， **在此报文之后的通信会采用Pre-master secret 密钥加密**。
  * **Client的所有密钥已经派生完毕，Client的握手阶段也即将结束**
  * 步骤 7： 客户端发送 Finished 报文。 该报文**包含连接至今全部报文的整体校验值**。 这次握手协商是否能够成功， 要以服务器是否能够正确解密该报文作为判定标准。

* 四次握手：

  * 步骤 8： 服务器同样发送 Change Cipher Spec 报文。
* **Server的所有密钥已经派生完毕，Server的握手阶段也即将结束**
  * 步骤 9： 服务器同样发送 Finished 报文，该报文**包含连接至今全部报文的整体校验值**。

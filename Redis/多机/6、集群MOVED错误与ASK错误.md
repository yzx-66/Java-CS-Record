#### 键操作与Moved错误

在对数据库中的16384个槽都进行了指派之后，集群就会进入上线状态，这时客户端就可以向集群中的节点发送数据命令了。



##### 实现原理

当客户端向节点发送与数据库键有关的命令时，接收命令的节点会计算出命令要处理的数据库键属于哪个槽，并检查这个槽是否指派给了自己：

* 如果键所在的槽正好就指派给了当前节点，那么节点直接执行这个命令。

* 如果键所在的槽并没有指派给当前节点，那么节点会向客户端返回一个MOVED错误，指引客户端转向（redirect）至正确的节点，并再次发送之前想要执行的命令。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110101351402.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


集群模式中每个节点的数据库

* 集群节点保存键值对以及键值对过期时间的方式，与单机Redis服务器保存键值对以及键值对过期时间的方式完全相同。
* 节点和单机服务器在数据库方面的一个区别是，节点只能使用0号数据库，而单机Redis服务器则没有这一限制。





##### 执行步骤

计算键属于哪个槽

* 节点使用以下算法来计算给定键key属于哪个槽：

  ```python
  def slot_number(key):
      return CRC16(key) & 16383
  ```

  其中CRC16（key）语句用于计算键key的CRC-16校验和，而&16383语句则用于计算出一个介于0至16383之间的整数作为键key的槽号。

* 使用CLUSTER KEYSLOT \<key>命令可以查看一个给定键属于哪个槽



判断槽是否由当前节点负责处理

* 当节点计算出键所属的槽i之后，节点就会检查自己在clusterState.slots数组中的项i，判断键所在的槽是否由自己负责：
  * 如果clusterState.slots[i]等于clusterState.myself，那么说明槽i由当前节点负责，节点可以执行客户端发送的命令。
  * 如果clusterState.slots[i]不等于clusterState.myself，那么说明槽i并非由当前节点负责，节点会根据clusterState.slots[i]指向的clusterNode结构所记录的节点IP和端口号，向客户端返回MOVED错误，指引客户端转向至正在处理槽i的节点。



返回MOVED错误重定向

* 当节点发现键所在的槽并非由自己负责处理的时候，节点就会向客户端返回一个MOVED错误，指引客户端转向至正在负责槽的节点。

* MOVED错误的格式为：

  ```
  MOVED <slot> <ip>:<port>
  ```

  * 其中slot为键所在的槽，而ip和port则是负责处理槽slot的节点的IP地址和端口号

* 被隐藏的MOVED错误：

  * 集群模式的redis-cli客户端在接收到MOVED错误时，并不会打印出MOVED错误，而是根据MOVED错误自动进行节点转向，并打印出转向信息，所以我们是看不见节点返回的MOVED错误的

  * 但是，如果我们使用单机（stand alone）模式的redis-cli客户端，再次向节点7000发送相同的命令，那么MOVED错误就会被客户端打印出来，因为单机模式的redis-cli客户端不清楚MOVED错误的作用，所以它只会直接将MOVED错误直接打印出来，而不会进行自动转向。

  ```
  $ redis-cli -c -p 7000 # 集群模式
  127.0.0.1:7000> SET msg "happy new year!"
  -> Redirected to slot [6257] located at 127.0.0.1:7001
  OK
  127.0.0.1:7001>
  
  ===========================================
  
  $ redis-cli -p 7000 # 单机模式
  127.0.0.1:7000> SET msg "happy new year!"
  (error) MOVED 6257 127.0.0.1:7001
  127.0.0.1:7000>
  ```

    

* 一个集群客户端通常会与集群中的多个节点创建套接字（Socket）连接，而所谓的节点转向实际上就是换一个套接字（Socket）来发送命令。

  * 如果客户端尚未与想要转向的节点创建套接字（Socket）连接，那么客户端会先根据MOVED错误提供的IP地址和端口号来连接节点，然后再进行转向。







#### 重分片与ASK错误

Redis集群的重新分片操作（比如新增节点）可以将任意数量已经指派给某个节点（源节点）的槽改为指派给另一个节点（目标节点），并且相关槽所属的键值对也会从源节点被移动到目标节点。

* 重新分片操作可以在线（online）进行，在重新分片的过程中，集群不需要下线，并且源节点和目标节点都可以继续处理命令请求。



##### 实现原理

* Redis集群的重新分片操作是由Redis的集群管理软件redis-trib负责执行的，Redis提供了进行重新分片所需的所有命令，而redis-trib则通过向源节点和目标节点发送命令来进行重新分片操作。



redis-trib对集群的单个槽slot进行重新分片的步骤如下：

* redis-trib对目标节点发送 CLUSTER SETSLOT \<slot> IMPORTING <source_id> 命令，让目标节点准备好从源节点导入（import）属于槽slot的键值对。

* redis-trib对源节点发送 CLUSTER SETSLOT \<slot> MIGRATING <target_id> 命令，让源节点准备好将属于槽 slot 的键值对迁移（migrate）至目标节点。

* redis-trib向源节点发送 CLUSTER GETKEYSINSLOT \<slot> \<count> 命令，获得最多 count 个属于槽 slot 的键值对的键名（key name）。

* 对于步骤3获得的每个键名，redis-trib 都向源节点发送一个 MIGRATE <target_ip> <target_port> <key_name> 0 \<timeout> 命令（用来标志该节点的这个 key 已经被迁移到了 target 节点），将被选中的键原子地从源节点迁移至目标节点。

* 重复执行步骤3和步骤4，直到源节点保存的所有属于槽 slot 的键值对都被迁移至目标节点为止。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110101410556.png#pic_center)


  

* redis-trib 向集群中的任意一个节点发送 CLUSTER SETSLOT \<slot> NODE <target_id> 命令，将槽slot指派给目标节点，这一指派信息会通过消息发送至整个集群，最终集群中的所有节点都会知道槽slot已经指派给了目标节点。

  

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110101424365.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)






##### ASK错误

在进行重新分片期间，源节点向目标节点迁移一个槽的过程中，可能会出现这样一种情况：属于被迁移槽的一部分键值对保存在源节点里面，而另一部分键值对则保存在目标节点里面。



当客户端向源节点发送一个与数据库键有关的命令，并且命令要处理的数据库键恰好就属于正在被迁移的槽时：

* 源节点会先在自己的数据库里面查找指定的键，如果找到的话，就直接执行客户端发送的命令。
* 相反地，如果源节点没能在自己的数据库里面找到指定的键，那么这个键有可能已经被迁移到了目标节点，源节点将向客户端返回一个ASK错误，指引客户端转向正在导入槽的目标节点，并再次发送之前想要执行的命令。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110101437630.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




被隐藏的ASK错误

* 和接到MOVED错误时的情况类似，集群模式的redis-cli在接到ASK错误时也不会打印错误，而是自动根据错误提供的IP地址和端口进行转向动作。如果想看到节点发送的ASK错误的话，可以使用单机模式的redis-cli客户端：

  ```
  $ redis-cli -p 7002
  127.0.0.1:7002> GET "love"
  (error) ASK 16198 127.0.0.1:7003
  ```




ASK错误和MOVED错误的区别

* ASK错误和MOVED错误都会导致客户端转向
  * 原理和 dict 的 rehash 很像，都是先到路由到自己的索引表，查看是否包含要查找的 key，不包含的话就路由去新的位置。

* 区别
  * MOVED错误代表槽的负责权已经从一个节点转移到了另一个节点：在客户端收到关于槽i的MOVED错误之后，客户端每次遇到关于槽i的命令请求时，都可以直接将命令请求发送至MOVED错误所指向的节点，因为该节点就是目前负责槽i的节点。
  * 与此相反，ASK错误只是两个节点在迁移槽的过程中使用的一种临时措施：
    * 在客户端收到关于槽i的ASK错误之后，客户端只会在接下来的一次命令请求中将关于槽i的命令请求发送至ASK错误所指示的节点，因为可能该槽位还有没迁移完的元素，所以只有全部迁移完之后才会把这个槽位标记成 target 节点的。
    * 所以这种转向不会对客户端今后发送关于槽i的命令请求产生任何影响，客户端仍然会将关于槽i的命令请求发送至目前负责处理槽i的节点，除非ASK错误再次出现。





##### 执行步骤

CLUSTER SETSLOT IMPORTING 实现

* clusterState结构的 importing_slots_from 数组记录了当前节点正在从其他节点导入的槽：

  ```c
  typedef struct clusterState {
    // ...
    clusterNode *importing_slots_from[16384];
    // ...
  } clusterState;
  ```

  如果importing_slots_from[i]的值不为NULL，而是指向一个clusterNode结构，那么表示当前节点正在从clusterNode所代表的节点导入槽i。

  

* 在对集群进行重新分片的时候，向目标节点发送命令：

  ```shell
  # 将目标节点 clusterState.importing_slots_from[i] 的值设置为 source_id 所代表节点的clusterNode结构。
  CLUSTER SETSLOT <i> IMPORTING <source_id> 
  ```

```
  
  例如：7003 客户端向节点发送以下命令
  
  ```shell
  # 9dfb... 是节点7002 的ID 
  127.0.0.1:7003> CLUSTER SETSLOT 16198 IMPORTING 9dfb4c4e016e627d9769e4c9bb0d4fa208e65c26
  OK
```

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110101450749.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  

CLUSTER SETSLOT MIGRATING 实现

* clusterState结构的 migrating_slots_to 数组记录了当前节点正在迁移至其他节点的槽：

  ```c
  typedef struct clusterState {
     // ...
     clusterNode *migrating_slots_to[16384];
     // ...
  } clusterState;
  ```

  如果migrating_slots_to[i]的值不为NULL，而是指向一个clusterNode结构，那么表示当前节点正在将槽i迁移至clusterNode所代表的节点。

  

* 在对集群进行重新分片的时候，向源节点发送命令：

  ```shell
  # 将源节点clusterState.migrating_slots_to[i]的值设置为target_id所代表节点的clusterNode结构。
  CLUSTER SETSLOT <i> MIGRATING <target_id>
  ```

  例如：7002客户端向节点发送以下命令

  ```shell
  # 0457... 是节点7003 的ID 
  127.0.0.1:7002> CLUSTER SETSLOT 16198 MIGRATING 04579925484ce537d3410d7ce97bd2e260c459a2
  OK
  ```

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110101502855.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




ASK 错误流程

* 如果节点收到一个关于键key的命令请求，并且键key所属的槽i正好就指派给了这个节点，那么节点会尝试在自己的数据库里查找键key，如果找到了的话，节点就直接执行客户端发送的命令。

  与此相反，如果节点没有在自己的数据库里找到键key，那么节点会检查自己的clusterState.migrating_slots_to[i]，看键key所属的槽i是否正在进行迁移，如果槽i的确在进行迁移的话，那么节点会向客户端发送一个ASK错误，引导客户端到正在导入槽i的节点去查找键key。

* 接到ASK错误的客户端会根据错误提供的IP地址和端口号，转向至正在导入槽的目标节点，然后首先向目标节点发送一个ASKING命令，之后再重新发送原本想要执行的命令。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110101515223.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


* ASKING 命令

  ASKING命令唯一要做的就是打开发送该命令的客户端的REDIS_ASKING标识，以下是该命令的伪代码实现：

  ```python
  def ASKING():
      # 打开标识
      client.flags |= REDIS_ASKING
      # 向客户端返回OK 回复
      reply("OK")
  ```

  * 在一般情况下，如果客户端向节点发送一个关于槽i的命令，而槽i又没有指派给这个节点的话，那么节点将向客户端返回一个MOVED错误；但是，如果节点的clusterState.importing_slots_from[i]显示节点正在导入槽i，并且发送命令的客户端带有REDIS_ASKING标识，那么节点将破例执行这个关于槽i的命令一次

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110101530622.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  * 另外要注意的是，客户端的REDIS_ASKING标识是一个一次性标识，当节点执行了一个带有REDIS_ASKING标识的客户端发送的命令之后，客户端的REDIS_ASKING标识就会被移除。

  ```shell
  127.0.0.1:7003> GET "love"  # 单机模式，love 所在的槽位正在被重分配
  (error) MOVED 16198 127.0.0.1:7002
  
  #========================================================
  
  127.0.0.1:7003> ASKING       # 打开 REDIS_ASKING 标识
  OK
  127.0.0.1:7003> GET "love"   # 访问一次后自动移除 REDIS_ASKING 标识
  "you get the key 'love'"
  127.0.0.1:7003> GET "love"   # 此时 REDIS_ASKING 标识未打开，执行失败
  (error) MOVED 16198 127.0.0.1:7002
  ```

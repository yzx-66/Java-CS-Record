### Sentinel

Sentinel（哨岗、哨兵）是Redis的高可用性（high availability）解决方案：

* 由一个或多个Sentinel实例（instance）组成的Sentinel系统（system）可以监视任意多个主服务器，以及这些主服务器属下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器，然后由新的主服务器代替已下线的主服务器继续处理命令请求。



示例：

* Sentinel系统监视服务器：

  双环图案表示的是当前的主服务器server1；单环图案表示的是主服务器的三个从服务器server2、server3以及server4

  server2、server3、server4三个从服务器正在复制主服务器server1，而Sentinel系统则在监视所有四个服务器

  

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100317162.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



* 假设这时，主服务器server1进入下线状态

  那么从服务器server2、server3、server4对主服务器的复制操作将被中止，并且Sentinel系统会察觉到server1已下线

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100330803.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  

* Sentinel系统就会对server1执行故障转移操作

  * 首先，Sentinel系统会挑选server1属下的其中一个从服务器，并将这个被选中的从服务器升级为新的主服务器。

  * 之后，Sentinel系统会向server1属下的所有从服务器发送新的复制指令，让它们成为新的主服务器的从服务器，当所有从服务器都开始复制新的主服务器时，故障转移操作执行完毕。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100343979.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  

* 另外，Sentinel还会继续监视已下线的server1，并在它重新上线时，将它设置为新的主服务器的从服务器。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/2020111010035996.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  



#### 启动过程

启动一个Sentinel可以使用命令：

```
redis-sentinel /path/to/your/sentinel.conf

# 或者命令(这两个命令的效果完全相同)：
redis-server /path/to/your/sentinel.conf --sentinel
```



当一个Sentinel启动时，它需要执行以下步骤：

* 初始化服务器。

* 将普通Redis服务器使用的代码替换成Sentinel专用代码。

* 初始化Sentinel状态。

* 根据给定的配置文件，初始化Sentinel的监视主服务器列表。

* 创建连向主服务器的网络连接。





##### 初始化服务器

首先，因为Sentinel本质上只是一个运行在特殊模式下的Redis服务器，所以启动Sentinel的第一步，就是初始化一个普通的Redis服务器

不过，因为Sentinel执行的工作和普通Redis服务器执行的工作不同，所以Sentinel的初始化过程和普通Redis服务器的初始化过程并不完全相同。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100410493.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)






##### 使用Sentinel专用代码

启动Sentinel的第二个步骤就是将一部分普通Redis服务器使用的代码替换成Sentinel专用代码。

比如说端口：

* 普通Redis服务器使用redis.h/REDIS_SERVERPORT常量的值作为服务器端口：

  ```c
  #define REDIS_SERVERPORT 6379
  ```

* Sentinel则使用sentinel.c/REDIS_SENTINEL_PORT常量的值作为服务器端口：

  ```c
  #define REDIS_SENTINEL_PORT 26379
  ```



除此之外还有命令表：

* 普通Redis服务器使用redis.c/redisCommandTable作为服务器的命令表：

  ```c
  struct redisCommand redisCommandTable[] = {
      {"get",getCommand,2,"r",0,NULL,1,1,1,0,0},
      {"set",setCommand,-3,"wm",0,noPreloadGetKeys,1,1,1,0,0},
      {"setnx",setnxCommand,3,"wm",0,noPreloadGetKeys,1,1,1,0,0},
      // ...
      {"script",scriptCommand,-2,"ras",0,NULL,0,0,0,0,0},
      {"time",timeCommand,1,"rR",0,NULL,0,0,0,0,0},
      {"bitop",bitopCommand,-4,"wm",0,NULL,2,-1,1,0,0},
      {"bitcount",bitcountCommand,-2,"r",0,NULL,1,1,1,0,0}
  }
  ```

* 而Sentinel则使用sentinel.c/sentinelcmds作为服务器的命令表

  ```c
  struct redisCommand sentinelcmds[] = {
      {"ping",pingCommand,1,"",0,NULL,0,0,0,0,0},
      {"sentinel",sentinelCommand,-2,"",0,NULL,0,0,0,0,0},
      {"subscribe",subscribeCommand,-2,"",0,NULL,0,0,0,0,0},
      {"unsubscribe",unsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
      {"psubscribe",psubscribeCommand,-2,"",0,NULL,0,0,0,0,0},
      {"punsubscribe",punsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
      // INFO命令会使用Sentinel模式下的专用实现sentinel.c/sentinelInfoCommand函数，
      // 而不是普通Redis服务器使用的实现redis.c/infoCommand函数
      {"info",sentinelInfoCommand,-1,"",0,NULL,0,0,0,0,0}
  };
  ```

  * sentinelcmds命令表也解释了为什么在Sentinel模式下，Redis服务器不能执行诸如SET、DBSIZE、EVAL等等这些命令，因为服务器根本没有在命令表中载入这些命令。
  * PING、SENTINEL、INFO、SUBSCRIBE、UNSUBSCRIBE、PSUBSCRIBE和PUNSUBSCRIBE这七个命令就是客户端可以对Sentinel执行的全部命令了。



##### 初始化sentinelState

* 主要是初始化 masters 属性

在应用了 Sentinel 的专用代码之后，接下来，服务器会初始化一个sentinel.c/sentinelState结构，这个结构保存了服务器中所有和Sentinel功能有关的状态（服务器的一般状态仍然由redis.h/redisServer结构保存）：

```c
struct sentinelState {
    // 当前纪元，用于实现故障转移
    uint64_t current_epoch;
    // 保存了所有被这个sentinel监视的主服务器
    // 字典的键是主服务器的名字
    // 字典的值则是一个指向sentinelRedisInstance结构的指针
    dict *masters;
    // 是否进入了TILT模式？
    int tilt;
    // 目前正在执行的脚本的数量
    int running_scripts;
    // 进入TILT模式的时间
    mstime_t tilt_start_time;
    // 最后一次执行时间处理器的时间
    mstime_t previous_time;
    // 一个FIFO队列，包含了所有需要执行的用户脚本
    list *scripts_queue;
} sentinel;
```



对 Sentine l状态的初始化将引发对masters字典的初始化，而masters字典的初始化是根据被载入的Sentinel配置文件来进行的

* 只有 masters 没有 slaves，每个 master 节点 sentinelRedisInstance 的 slaves 属性会保存它 slave 节点的 sentinelRedisInstance

* 每个 sentinelRedisInstance 都有一个 addr 代表 sentinel.c/sentinelAddr结构的指针，这个结构保存着实例的IP地址和端口号：

  ```c
  typedef struct sentinelAddr {
      char *ip;
      int port;
  } sentinelAddr;
  ```

* 示例（只用配置 master ， 不用配置 slave ， slave 会通过 master 发现）：

  ```python
  #####################
  # master1 configure #
  #####################
  # Sentinel monitor <master-name> <ip><port> <quorum>
  #  <master-name> sentinel要监控的主节点的名称，或者应该说叫集群名字
  #  <ip><port> 启动时主节点的地址，后面如果主节点下线了会自动故障迁移，所以主节点并不总是该 ip：port
  #  <quorum> 代表要判定主节点最终不可达所需要的票数
  sentinel monitor master1 127.0.0.1 6379 2
  # 主观下线时间
  sentinel down-after-milliseconds master1 30000  
  # 故障转移后，有多少个从节点可以同时重新同步主节点
  # 该参数主要为了一个主从复制集群全部都在进行复制，从而导致无法对外提供服务
  sentinel parallel-syncs master1 1
  # 故障转移超时时间，作用于各个阶段。
  #  A) 选出合适从节点
  #  B) 晋升选出的从节点为主节点
  #  C) 命令其余从节点复制新的主节点
  #  D) 等待原主节点恢复后命令它去复制新的主节点
  sentinel failover-timeout master1 900000
  
  #####################
  # master2 configure #
  #####################
  sentinel monitor master2 127.0.0.1 12345 5
  sentinel down-after-milliseconds master2 50000
  sentinel parallel-syncs master2 5
  sentinel failover-timeout master2 450000
  ```

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100424521.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




##### 连接主服务器

初始化Sentinel的最后一步是创建连向被监视主服务器的网络连接，Sentinel将成为主服务器的客户端，它可以向主服务器发送命令，并从命令回复中获取相关的信息。



对于每个被Sentinel监视的主服务器来说，Sentinel会给每个服务器创建两个连接

* 一个是命令连接，这个连接专门用于向主服务器发送命令，并接收命令回复。

* 另一个是订阅连接（作用是用来发现新的监视该服务器的 sentinel），这个连接专门用于订阅主服务器的\__sentinel__:hello频道。 

因为Sentinel需要与多个实例创建多个网络连接，所以Sentinel使用的是异步连接。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100437532.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)






#### 发现Slave和Sentinel

##### 获取主服务器信息

Sentinel默认会以每十秒一次的频率，通过命令连接向被监视的主服务器发送INFO命令，并通过分析INFO命令的回复来获取主服务器的当前信息。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100449981.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




通过分析主服务器返回的INFO命令回复，Sentinel可以获取以下两方面的信息：

  ```
# Server
...
run_id:7611c59dc3a29aa6fa0609f841bb6a1019008a9c
...
# Replication
role:master
...
slave0:ip=127.0.0.1,port=11111,state=online,offset=43,lag=0
slave1:ip=127.0.0.1,port=22222,state=online,offset=43,lag=0
slave2:ip=127.0.0.1,port=33333,state=online,offset=43,lag=0
...
# Other sections
...
  ```

* 一方面是关于主服务器本身的信息，包括run_id域记录的服务器运行ID，以及role域记录的服务器角色；
* 另一方面是关于主服务器属下所有从服务器的信息，每个从服务器都由一个"slave"字符串开头的行记录，每行的ip域记录了从服务器的IP地址，而port域则记录了从服务器的端口号。根据这些IP地址和端口号，Sentinel无须用户提供从服务器的地址信息，就可以自动发现从服务器。



Sentinel在分析INFO命令中包含的从服务器信息时，会检查从服务器对应的实例结构是否已经存在于slaves字典：

* 如果从服务器对应的实例结构已经存在，那么Sentinel对从服务器的实例结构进行更新。

* 如果从服务器对应的实例结构不存在，那么说明这个从服务器是新发现的从服务器，Sentinel会在slaves字典中为这个从服务器新创建一个实例结构。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100503399.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


注意主服务器实例结构和从服务器实例结构之间的区别：

* 主服务器实例结构的flags属性的值为SRI_MASTER，而从服务器实例结构的flags属性的值为SRI_SLAVE。

* 主服务器实例结构的name属性的值是用户使用Sentinel配置文件设置的，而从服务器实例结构的name属性的值则是Sentinel根据从服务器的IP地址和端口号自动设置的。 



##### 获取从服务器信息

当Sentinel发现主服务器有新的从服务器出现时，Sentinel除了会为这个新的从服务器创建相应的实例结构之外，Sentinel还会创建连接到从服务器的命令连接和订阅连接。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100515485.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




在创建命令连接之后，Sentinel在默认情况下，会以每十秒一次的频率通过命令连接向从服务器发送INFO命令

```
# Server
...
run_id:32be0699dd27b410f7c90dada3a6fab17f97899f
...
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
slave_repl_offset:11887
slave_priority:100
# Other sections
...
```

根据INFO命令的回复，Sentinel会提取出以下信息：

* 从服务器的运行ID run_id。

* 从服务器的角色role。

* 主服务器的IP地址master_host，以及主服务器的端口号master_port。

* 主从服务器的连接状态master_link_status。

* 从服务器的优先级slave_priority。

* 从服务器的复制偏移量slave_repl_offset。

根据这些信息，Sentinel会对从服务器的实例结构进行更新

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100529813.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




##### 发送Sentinel监视信息

在默认情况下，Sentinel会以每两秒一次的频率，会**通过每个主服务器和从服务器所对应的命令连接，发送以下格式的消息发布命令**：

```
PUBLISH __sentinel__:hello "<s_ip>,<s_port>,<s_runid>,<s_epoch>,<m_name>,<m_ip>,<m_port>,<m_epoch>"
```

这条命令向服务器的\__sentinel__:hello频道发送了一条信息，信息的内容由多个参数组成：

* 其中以s_开头的参数记录的是Sentinel本身的信息

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100549440.png#pic_center)


* 而m_开头的参数记录的则是主服务器的信息。如果Sentinel正在监视的是主服务器，那么这些参数记录的就是主服务器的信息；如果Sentinel正在监视的是从服务器，那么这些参数记录的就是从服务器正在复制的主服务器的信息。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100601551.png#pic_center)


  



##### Sentinel接收频道信息

因为当Sentinel与一个主服务器或者从服务器建立起订阅连接之后，Sentinel就会**通过该服务器对应的订阅连接，向服务器发送消息订阅命令**：

```
SUBSCRIBE __sentinel__:hello
```

Sentinel 对\__sentinel__:hello 频道的订阅会一直持续到 Sentinel 与服务器的连接断开为止，即与该服务器对应的命令连接和订阅连接都断开了。



这也就是说

* 对于每个与Sentinel连接的服务器，Sentinel 既通过该服务器对应的命令连接向服务器的\_\_sentinel\_\_:hello 频道发送信息，又通过该服务器对应的订阅连接从服务器的\__sentinel\_\_:hello频道接收信息

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100614377.png#pic_center)




* 对于监视同一个服务器的多个Sentinel来说，一个Sentinel发送的信息会被其他Sentinel接收到，这些信息会被用于更新其他Sentinel对发送信息Sentinel的认知，也会被用于更新其他Sentinel对被监视服务器的认知。  

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100626884.png#pic_center)






当一个Sentinel从\__sentinel__:hello频道收到一条信息时，Sentinel会对这条信息进行分析，提取出信息中的Sentinel IP地址、Sentinel端口号、Sentinel运行ID等八个参数（就是上面发送监视信息提到的 8 个参数），并进行以下检查：

* 如果信息中记录的Sentinel运行ID和接收信息的Sentinel的运行ID相同，那么说明这条信息是Sentinel自己发送的，Sentinel将丢弃这条信息，不做进一步处理。

* 相反地，如果信息中记录的Sentinel运行ID和接收信息的Sentinel的运行ID不相同，那么说明这条信息是监视同一个服务器的其他Sentinel发来的，接收信息的Sentinel将根据信息中的各个参数，对相应主服务器的实例结构进行更新。



更新sentinels字典

* Sentinel为主服务器创建的实例结构中的sentinels字典保存了除Sentinel本身之外，所有同样监视这个主服务器的其他Sentinel的资料：

  * sentinels字典的键是其中一个Sentinel的名字，格式为ip:port，比如对于IP地址为127.0.0.1，端口号为26379的Sentinel来说，这个Sentinel在sentinels字典中的键就是"127.0.0.1:26379"。
  * sentinels字典的值则是键所对应Sentinel的实例结构，比如对于键"127.0.0.1:26379"来说，这个键在sentinels字典中的值就是IP为127.0.0.1，端口号为26379的Sentinel的实例结构。

* 当一个Sentinel接收到其他Sentinel发来的信息时（我们称呼发送信息的Sentinel为源Sentinel，接收信息的Sentinel为目标Sentinel），目标Sentinel会从信息中分析并提取出以下两方面参数：

  * 与Sentinel有关的参数：源Sentinel的IP地址、端口号、运行ID和配置纪元。

  * 与主服务器有关的参数：源Sentinel正在监视的主服务器的名字、IP地址、端口号和配置纪元。

* 根据信息中提取出的主服务器参数，目标Sentinel会在自己的Sentinel状态的masters字典中查找相应的主服务器实例结构，然后根据提取出的Sentinel参数，检查主服务器实例结构的sentinels字典中，源Sentinel的实例结构是否存在：

  * 如果源Sentinel的实例结构已经存在，那么对源Sentinel的实例结构进行更新。
  * 如果源Sentinel的实例结构不存在，那么说明源Sentinel是刚刚开始监视主服务器的新Sentinel，目标Sentinel会为源Sentinel创建一个新的实例结构，并将这个结构添加到sentinels字典里面。 

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100641352.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




创建连向其他Sentinel的命令连接

* 当Sentinel通过频道信息发现一个新的Sentinel时，它不仅会为新Sentinel在sentinels字典中创建相应的实例结构，还会创建一个连向新Sentinel的命令连接，而新Sentinel也同样会创建连向这个Sentinel的命令连接

* 最终监视同一主服务器的多个Sentinel将形成相互连接的网络：Sentinel A有连向Sentinel B的命令连接，而Sentinel B也有连向Sentinel A的命令连接。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110100654746.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


* 注意：Sentinel之间不会创建订阅连接

  * Sentinel在连接主服务器或者从服务器时，会同时创建命令连接和订阅连接，但是在连接其他Sentinel时，却只会创建命令连接，而不创建订阅连接。
  * 这是因为**Sentinel需要通过接收主服务器或者从服务器发来的频道信息来发现未知的新Sentinel**，所以才需要建立订阅连接，而相互已知的Sentinel只要使用命令连接来进行通信就足够了。

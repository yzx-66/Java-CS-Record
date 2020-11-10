### RedisServer

#### 请求处理 (文件事件)

* 命令请求执行过程：即 processFileEvents 里读事件处理器的主要操作



##### 发送命令请求

Redis服务器的命令请求来自Redis客户端，当用户在客户端中键入一个命令请求时，客户端会将这个命令请求转换成协议格式，然后通过连接到服务器的套接字，将协议格式的命令请求发送给服务器。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094031308.png#pic_center)




示例（步骤一）：

```
SET KEY VALUE

# 那么客户端会将这个命令转换成协议：
*3\r\n$3\r\nSET\r\n$3\r\nKEY\r\n$5\r\nVALUE\r\n
```



##### 读取命令请求

* 注意：不会出现要同时处理一个客户端多个请求的情况，因为redis是 单Reactor单线程 模式，所以一次只能处理一个请求，完了才能处理下一个

  

当客户端与服务器之间的连接套接字因为客户端的写入而变得可读时，服务器将调用命令请求处理器来执行以下操作：

* 读取套接字中协议格式的命令请求，并将其保存到客户端状态的输入缓冲区里面。
* 对输入缓冲区中的命令请求进行分析，提取出命令请求中包含的命令参数，以及命令参数的个数，然后分别将参数和参数个数保存到客户端状态的argv属性和argc属性里面。
* 调用命令执行器，执行客户端指定的命令。



示例（步骤二）：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094055392.png#pic_center)






##### 查找命令实现

* 命令执行器（1）：查找命令实现

命令执行器要做的第一件事就是根据客户端状态的argv[0]参数，在命令表（command table）中查找参数所指定的命令，并将找到的命令保存到客户端状态的cmd属性里面。



命令表是一个字典

* 字典的键是一个个命令名字，比如"set"、"get"、"del"等等；

* 而字典的值则是一个个redisCommand结构，每个redisCommand结构记录了一个Redis命令的实现信息

  redisCommand属性：![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094111233.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




redisCommand#sflags：![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094125348.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


* 演示：

  * SET命令的名字为"set"，实现函数为setCommand；命令的参数个数为-3，表示命令接受三个或以上数量的参数；命令的标识为"wm"，表示SET命令是一个写入命令，并且在执行这个命令之前，服务器应该对占用内存状况进行检查，因为这个命令可能会占用大量内存。
  * GET命令的名字为"get"，实现函数为getCommand函数；命令的参数个数为2，表示命令只接受两个参数；命令的标识为"r"，表示这是一个只读命令。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094141551.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




示例（步骤三）：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094154138.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




##### 执行预备操作

* 命令执行器（2）：执行预备操作

  

到目前为止，服务器已经将执行命令所需的命令实现函数（保存在客户端状态的cmd属性）、参数（保存在客户端状态的argv属性）、参数个数（保存在客户端状态的argc属性）都收集齐了。

但是在真正执行命令之前，程序还需要进行一些预备操作，从而确保命令可以正确、顺利地被执行，这些操作包括：

* 检查客户端状态的cmd指针是否指向NULL，如果是的话，那么说明用户输入的命令名字找不到相应的命令实现，服务器不再执行后续步骤，并向客户端返回一个错误。
* 根据客户端cmd属性指向的redisCommand结构的arity属性，检查命令请求所给定的参数个数是否正确，当参数个数不正确时，不再执行后续步骤，直接向客户端返回一个错误。比如说，如果redisCommand结构的arity属性的值为-3，那么用户输入的命令参数个数必须大于等于3个才行。
* 检查客户端是否已经通过了身份验证，未通过身份验证的客户端只能执行AUTH命令，如果未通过身份验证的客户端试图执行除AUTH命令之外的其他命令，那么服务器将向客户端返回一个错误。
* 如果服务器打开了maxmemory功能，那么在执行命令之前，先检查服务器的内存占用情况，并在有需要时进行内存回收，从而使得接下来的命令可以顺利执行。如果内存回收失败，那么不再执行后续步骤，向客户端返回一个错误。
* 如果服务器上一次执行BGSAVE命令时出错，并且服务器打开了stop-writes-on-bgsave-error功能，而且服务器即将要执行的命令是一个写命令，那么服务器将拒绝执行这个命令，并向客户端返回一个错误。
* 如果客户端当前正在用SUBSCRIBE命令订阅频道，或者正在用PSUBSCRIBE命令订阅模式，那么服务器只会执行客户端发来的SUBSCRIBE、PSUBSCRIBE、UNSUBSCRIBE、PUNSUBSCRIBE四个命令，其他命令都会被服务器拒绝。
* 如果服务器正在进行数据载入，那么客户端发送的命令必须带有l标识（比如INFO、SHUTDOWN、PUBLISH等等）才会被服务器执行，其他命令都会被服务器拒绝。
* 如果服务器因为执行Lua脚本而超时并进入阻塞状态，那么服务器只会执行客户端发来的SHUTDOWN nosave命令和SCRIPT KILL命令，其他命令都会被服务器拒绝。
* 如果客户端正在执行事务，那么服务器只会执行客户端发来的EXEC、DISCARD、MULTI、WATCH四个命令，其他命令都会被放进事务队列中。
* 如果服务器打开了监视器功能，那么服务器会将要执行的命令和参数等信息发送给监视器。当完成了以上预备操作之后，服务器就可以开始真正执行命令了。



##### 调用命令的实现函数

* 命令执行器（3）：调用命令的实现函数

在前面的操作中，服务器已经将要执行命令的实现保存到了客户端状态的cmd属性里面，并将命令的参数和参数个数分别保存到了客户端状态的argv属性和argv属性里面。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094207814.png#pic_center)




当服务器决定要执行命令时，它只要执行以下语句就可以了：

```c
// client
是指向客户端状态的指针
client->cmd->proc(client);
```



被调用的命令实现函数会执行指定的操作，并产生相应的命令回复，这些回复会被保存在客户端状态的输出缓冲区里面（buf属性和reply属性），之后实现函数还会为客户端的套接字关联命令回复处理器，这个处理器负责将命令回复返回给客户端。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094224385.png#pic_center)




##### 执行后续工作

* 命令执行器（4）：执行后续工作



在执行完实现函数之后，服务器还需要执行一些后续工作：

* 根据刚刚执行命令所耗费的时长，更新被执行命令的redisCommand结构的milliseconds属性，并将命令的redisCommand结构的calls计数器的值增一。

* 如果服务器开启了AOF持久化功能，那么AOF持久化模块会将刚刚执行的命令请求写入到AOF缓冲区里面。

* 如果服务器开启了慢查询日志功能，那么慢查询日志模块会检查是否需要为刚刚执行完的命令请求添加一条新的慢查询日志。

* 如果有其他从服务器正在复制当前这个服务器，那么服务器会将刚刚执行的命令传播给所有从服务器。



当以上操作都执行完了之后，服务器对于当前命令的执行到此就告一段落了，之后服务器就可以继续从文件事件处理器中取出并处理下一个命令请求了。



##### 回复给客户端

前面说过，命令实现函数会将命令回复保存到客户端的输出缓冲区里面，并为客户端的套接字关联命令回复处理器，当客户端套接字变为可写状态时，服务器就会执行命令回复处理器，将保存在客户端输出缓冲区中的命令回复发送给客户端。



当命令回复发送完毕之后，回复处理器会清空客户端状态的输出缓冲区，为处理下一个命令请求做好准备。



##### 客户端接收并打印

当客户端接收到协议格式的命令回复之后，它会将这些回复转换成人类可读的格式，并打印给用户观看

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094236545.png#pic_center)






#### serverCron (时间事件)

* 即 processTimeEvents 里处理的主要操作。Redis服务器中的serverCron函数默认每隔100毫秒执行一次，这个函数负责管理服务器的资源，并保持服务器自身的良好运转。



##### 更新时间缓存

Redis服务器中有不少功能需要获取系统的当前时间，而每次获取系统的当前时间都需要执行一次系统调用，为了减少系统调用的执行次数，服务器状态中的unixtime属性和mstime属性被用作当前时间的缓存：

```c
struct redisServer {
    // ...
    // 保存了秒级精度的系统当前UNIX时间戳
    time_t unixtime;
    // 保存了毫秒级精度的系统当前UNIX时间戳
    long long mstime;
    // ...
};
```



因为serverCron函数默认会以每100毫秒一次的频率更新unixtime属性和mstime属性，所以这两个属性记录的时间的精确度并不高：

* 服务器只会在打印日志、更新服务器的LRU时钟、决定是否执行持久化任务、计算服务器上线时间（uptime）这类对时间精确度要求不高的功能上。
* 对于为键设置过期时间、添加慢查询日志这种需要高精确度时间的功能来说，服务器还是会再次执行系统调用，从而获得最准确的系统当前时间。



##### 更新LRU时钟

服务器状态中的lruclock属性保存了服务器的LRU时钟，这个属性和上面介绍的unixtime属性、mstime属性一样，都是服务器时间缓存的一种：

```c
struct redisServer {
    // ...
    // 默认每10秒更新一次的时钟缓存，用于计算键的空转（idle）时长。
    unsigned lruclock:22;
    // ...
};
```

* lru 属性保存了服务器最后一次被命令访问的时间（每个 redisObject 也有自己的 lru 属性）



serverCron函数默认会以每10秒一次的频率更新lruclock属性的值，因为这个时钟不是实时的，所以根据这个属性计算出来的LRU时间实际上只是一个模糊的估算值。

lruclock时钟的当前值可以通过INFO server命令的lru_clock域查看



##### 更新每秒执行次数

serverCron函数中的trackOperationsPerSecond函数会以每100毫秒一次的频率执行，这个函数的功能是以抽样计算的方式，估算并记录服务器在最近一秒钟处理的命令请求数量，和服务器状态中四个ops_sec_开头的属性有关

```c
struct redisServer {
    // ...
    // 上一次进行抽样的时间
    long long ops_sec_last_sample_time;
    // 上一次抽样时，服务器已执行命令的数量
    long long ops_sec_last_sample_ops;
    // REDIS_OPS_SEC_SAMPLES大小（默认值为16）的环形数组，
    // 数组中的每个项都记录了一次抽样结果。
    long long ops_sec_samples[REDIS_OPS_SEC_SAMPLES];
    // ops_sec_samples数组的索引值，每次抽样后将值自增一，
    // 在值等于16时重置为0，让ops_sec_samples数组构成一个环形数组。
    int ops_sec_idx;
    // ...
};
```



trackOperationsPerSecond函数每次运行，都会根据ops_sec_last_sample_time记录的上一次抽样时间和服务器的当前时间，以及ops_sec_last_sample_ops记录的上一次抽样的已执行命令数量和服务器当前的已执行命令数量，计算出两次trackOperationsPerSecond调用之间，服务器平均每一毫秒处理了多少个命令请求，然后将这个平均值乘以1000，这就得到了服务器在一秒钟内能处理多少个命令请求的估计值，这个估计值会被作为一个新的数组项被放进ops_sec_samples环形数组里面。



当客户端执行INFO命令时，服务器就会调用getOperationsPerSecond函数，根据ops_sec_samples环形数组中的抽样结果，计算出instantaneous_ops_per_sec属性的值，以下是getOperationsPerSecond函数的实现代码：

```c
long long getOperationsPerSecond(void){
    int j;
    long long sum = 0;
    // 计算所有取样值的总和
    for (j = 0; j < REDIS_OPS_SEC_SAMPLES; j++)
    sum += server.ops_sec_samples[j];
    // 计算取样的平均值
    return sum / REDIS_OPS_SEC_SAMPLES;
}
```

根据getOperationsPerSecond函数的定义可以看出，instantaneous_ops_per_sec属性的值是通过计算最近REDIS_OPS_SEC_SAMPLES次取样的平均值来计算得出的，它只是一个估算值。



##### 更新内存峰值记录

服务器状态中的stat_peak_memory属性记录了服务器的内存峰值大小：

```c
struct redisServer {
    // ...
    // 已使用内存峰值
    size_t stat_peak_memory;
    // ...
};
```



每次serverCron函数执行时，程序都会查看服务器当前使用的内存数量，并与stat_peak_memory保存的数值进行比较，如果当前使用的内存数量比stat_peak_memory属性记录的值要大，那么程序就将当前使用的内存数量记录到stat_peak_memory属性里面。



INFO memory命令的used_memory_peak和used_memory_peak_human两个域分别以两种格式记录了服务器的内存峰值。



##### 增加cronloops计数器

服务器状态的cronloops属性记录了serverCron函数执行的次数：

```c
struct redisServer {
    // ...
    // serverCron函数的运行次数计数器
    // serverCron函数每执行一次，这个属性的值就增一。
    int cronloops;
    // ...
};
```



cronloops属性目前在服务器中的唯一作用，就是在复制模块中实现“每执行serverCron函数N次就执行一次指定代码”的功能，方法如以下伪代码所示：

```
if cronloops % N == 0: 
    # 执行指定代码... 
```





##### 处理SIGTERM信号

每次serverCron函数运行时，程序都会对服务器状态的shutdown_asap属性进行检查，并根据属性的值决定是否关闭服务器：

```c
struct redisServer {
    // ...
    // 关闭服务器的标识：
    // 值为1时，关闭服务器，
    // 值为0时，不做动作。
    int shutdown_asap;
    // ...
};
```

* 服务器在关闭自身之前会进行RDB持久化操作，这也是服务器拦截SIGTERM信号的原因，如果服务器一接到SIGTERM信号就立即关闭，那么它就没办法执行持久化操作了。



在启动服务器时，Redis会为服务器进程的SIGTERM信号关联处理器sigtermHandler函数，这个信号处理器负责在服务器接到SIGTERM信号时，打开服务器状态的shutdown_asap标识：

```c
// SIGTERM 信号的处理器
static void sigtermHandler(int sig) {
    // 打印日志
    redisLogFromHandler(REDIS_WARNING,"Received SIGTERM, scheduling shutdown...");
    // 打开关闭标识
    server.shutdown_asap = 1;
}
```



##### 管理数据库资源

serverCron函数每次执行都会调用databasesCron函数，这个函数会对服务器中的一部分数据库进行检查，删除其中的过期键，并在有需要时，对字典进行收缩操作。



##### 管理客户端资源

serverCron函数每次执行都会调用clientsCron函数，clientsCron函数会对一定数量的客户端进行以下两个检查：

* 如果客户端与服务器之间的连接已经超时（很长一段时间里客户端和服务器都没有互动），那么程序释放这个客户端。

* 如果客户端在上一次执行命令请求之后，输入缓冲区的大小超过了一定的长度，那么程序会释放客户端当前的输入缓冲区，并重新创建一个默认大小的输入缓冲区，从而防止客户端的输入缓冲区耗费了过多的内存。



##### 关闭异步客户端

在这一步，服务器会关闭那些输出缓冲区大小超出限制的客户端





##### 被延迟BGREWRITEAOF

在服务器执行BGSAVE命令的期间，如果客户端向服务器发来BGREWRITEAOF命令，那么服务器会将BGREWRITEAOF命令的执行时间延迟到BGSAVE命令执行完毕之后。

服务器的aof_rewrite_scheduled标识记录了服务器是否延迟了BGREWRITEAOF命令：

```c
struct redisServer {
    // ...
    // 如果值为1，那么表示有 BGREWRITEAOF命令被延迟了。
    int aof_rewrite_scheduled;
    // ...
};
```



每次serverCron函数执行时，函数都会检查BGSAVE命令或者BGREWRITEAOF命令是否正在执行，如果这两个命令都没在执行，并且aof_rewrite_scheduled属性的值为1，那么服务器就会执行之前被推延的BGREWRITEAOF命令。



##### 检查持久化操作

服务器状态使用rdb_child_pid属性和aof_child_pid属性记录执行BGSAVE命令和BGREWRITEAOF命令的子进程的ID，这两个属性也可以用于检查BGSAVE命令或者BGREWRITEAOF命令是否正在执行：

```c
struct redisServer {
    // ...
    // 记录执行BGSAVE命令的子进程的ID：
    // 如果服务器没有在执行BGSAVE，那么这个属性的值为-1。
    pid_t rdb_child_pid;                /* PID of RDB saving child */
    // 记录执行BGREWRITEAOF命令的子进程的ID：
    // 如果服务器没有在执行BGREWRITEAOF，那么这个属性的值为-1。
    pid_t aof_child_pid;                /* PID if rewriting process */
    // ...
};
```



每次serverCron函数执行时，程序都会检查rdb_child_pid和aof_child_pid两个属性的值，只要其中一个属性的值不为-1，程序就会执行一次wait3函数，检查子进程是否有信号发来服务器进程：

* 如果有信号到达，那么表示新的RDB文件已经生成完毕（对于BGSAVE命令来说），或者AOF文件已经重写完毕（对于BGREWRITEAOF命令来说），服务器需要进行相应命令的后续操作，比如用新的RDB文件替换现有的RDB文件，或者用重写后的AOF文件替换现有的AOF文件。

* 如果没有信号到达，那么表示持久化操作未完成，程序不做动作。



另一方面，如果rdb_child_pid和aof_child_pid两个属性的值都为-1，那么表示服务器没有在进行持久化操作，在这种情况下，程序执行以下三个检查：

* 查看是否有BGREWRITEAOF被延迟了，如果有的话，那么开始一次新的BGREWRITEAOF操作（这就是上条说到的检查）。

* 检查服务器的自动保存条件是否已经被满足，如果条件满足，并且服务器没有在执行其他持久化操作，那么服务器开始一次新的BGSAVE操作（因为条件1可能会引发一次BGREWRITEAOF，所以在这个检查中，程序会再次确认服务器是否已经在执行持久化操作了）。

* 检查服务器设置的AOF重写条件是否满足，如果条件满足，并且服务器没有在执行其他持久化操作，那么服务器将开始一次新的BGREWRITEAOF操作（因为条件1和条件2都可能会引起新的持久化操作，所以在这个检查中，我们要再次确认服务器是否已经在执行持久化操作了）。

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094250434.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




##### 将AOF缓冲区写入文件

如果服务器开启了AOF持久化功能，并且AOF缓冲区里面还有待写入的数据，那么serverCron函数会调用相应的程序，将AOF缓冲区中的内容写入到AOF文件里面










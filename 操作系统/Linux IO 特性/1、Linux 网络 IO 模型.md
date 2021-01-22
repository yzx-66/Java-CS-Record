# 五种 IO 模型

5 种（前4种IO都可以归类为synchronous IO - 同步IO）

* blocking IO - 阻塞IO 
* nonblocking IO - 非阻塞IO 
* IO multiplexing - IO多路复用 
* signal driven IO - 信号驱动IO （使用较少，不介绍）
* asynchronous IO - 异步IO



IO模型的异同点就是区分在两个系统对象、两个处理阶段的不同上

* 两个系统对象：
  * (1) 用户进程(线程)Process；
  * (2)内核对象kernel
* 两个处理阶段：
  * [1] Waiting for the data to be ready - 等待数据准备好
  * [2] Copying the data from the kernel to the process - 将数据从内核空间的buffer拷贝到用户空间进程的buffer





1、同步IO 之 Blocking IO

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210122011657164.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)


* 用户进程process在Blocking IO读recvfrom操作的两个阶段都是等待的。在数据没准备好的时候，process原地等待kernel准备数据。
* kernel准备好数据后，process继续等待kernel将数据copy到自己的buffer。在kernel完成数据的copy后process才会从recvfrom系统调用中返回。



2、同步IO 之 NonBlocking IO

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210122011712476.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)


* process在NonBlocking IO读recvfrom操作的第一个阶段是不会block等待的，如果kernel数据还没准备好，那么recvfrom会立刻返回一个EWOULDBLOCK错误。
* 当kernel准备好数据后，进入处理的第二阶段的时候，process会等待kernel将数据copy到自己的buffer，在kernel完成数据的copy后process才会从recvfrom系统调用中返回。



3、同步IO 之 IO multiplexing

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210122011725584.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)


* IO多路复用，就是select、poll、epoll模型。
* 在IO多路复用的时候，process在两个处理阶段都是block住等待的。select、poll、epoll的优势在于可以**以较少的代价来同时监听处理多个IO**。



4、异步IO

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210122011739532.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)


* 异步IO要求process在recvfrom操作的两个处理阶段上都不能等待，也就是process调用recvfrom后立刻返回，kernel自行去准备好数据并将数据从kernel的buffer中copy到process的buffer在通知process读操作完成了，然后process在去处理。
* 遗憾的是，linux的网络IO中是不存在异步IO的，linux的网络IO处理的第二阶段总是阻塞等待数据copy完成的。真正意义上的网络异步IO是Windows下的IOCP（IO完成端口）模型。



对比

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021012201175384.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)


* 很多时候，比较容易混淆non-blocking IO和asynchronous IO
* non-blocking IO和asynchronous IO的区别还是很明显的，non-blocking IO仅仅要求处理的第一阶段不block即可，而asynchronous IO要求两个阶段都不能block住。



# IO 多路复用

* select、poll、epoll 都是IO多路复用的机制，可以监视多个描述符的读/写等事件，一旦某个描述符就绪（一般是读或者写事件发生了），就能够将发生的事件通知给关心的应用程序去处理该事件。



出现原因

*  在对 socket 对于read 和 write 之前应该知道是否可读或可写，而不应该直接调用，然后睡眠
   * 例如，对于 read 应该期待”可读”事件的通知，而不是盲目地对每个socket调用recv/recvfrom来尝试接收数据。
*  由上面的需求，我们不知道什么时候，哪个socket会有读事件发生，就有了下面的  wakeup、 callback机制，
   * process 需要同时插入到这 sleep_list 上等待其关心的任意一个 socket 可读事件发生而被唤醒，当时 process 被唤醒的时候，其 callback 里面应该有个逻辑去检查具体那些 socket 可读了。





socket 事件的 wakeup、 callback机制

* linux(2.6+)内核的事件wakeup callback机制，是IO多路复用机制存在的本质。
  * Linux通过**socket睡眠队列**来管理所有等待socket的某个事件的process
  * 同时通过**wakeup机制来异步唤醒**整个睡眠队列上等待事件的process，通知process相关事件发生。
  * 通常情况，socket的事件发生的时候，其会**顺序遍历socket睡眠队列上的每个process节点，调用每个process节点挂载的callback函数**。
  * 在遍历的过程中，如果遇到某个节点是排他的，那么就终止遍历，总体上会涉及两大逻辑：（1）睡眠等待逻辑；（2）唤醒逻辑。

* （1）睡眠等待逻辑
  * select、poll、epoll_wait陷入内核，判断监控的socket是否有关心的事件发生了，如果没，则为当前process构建一个wait_entry节点，然后插入到监控socket的sleep_list
  * 进入循环的schedule直到关心的事件发生了
  * 关心的事件发生后，将当前process的wait_entry节点从socket的sleep_list中删除。

* （2）唤醒逻辑：
  * socket的事件发生了，然后socket顺序遍历其睡眠队列，依次调用每个wait_entry节点的callback函数
  * 直到完成队列的遍历或遇到某个wait_entry节点是排他的才停止。
  * 一般情况下callback包含两个逻辑：
    * 1、wait_entry自定义的私有逻辑；
    * 2、唤醒的公共逻辑，主要用于将该wait_entry的process放入CPU的就绪队列，让CPU随后可以调度其执行。





poll 函数

* 每个 socket 在加入等待队列前都会调用 poll 函数
  * 对于socket，这个poll方法是sock_poll，sock_poll根据情况会调用到tcp_poll,udp_poll或者datagram_poll
  * 主要用来收集socket发生的事件
* 对于发生了可读事件来说，简单伪码如下： 

```javascript
poll()
{
    //其他逻辑
    if (recieve queque is not empty)
    {
        // 如果是收到数据，那么设置可读标志位
        sk_event |= POLL_IN；
    }
   //其他逻辑
}
```



## Select

select

```javascript
// readfds、writefds、errorfds 是三个文件描述符集合（使用的位图）
// select 会遍历每个集合的前 nfds 个描述符，分别找到可以读取、可以写入、发生错误的描述符，统称为“就绪”的描述符
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

* 1、当用户process调用select的时候，select会将需要监控的readfds集合拷贝到内核空间（假设监控的仅仅是socket可读）

* 2、然后遍历自己监控的socket sk，挨个调用sk的poll逻辑以便检查该sk是否有可读事件，遍历完所有的sk后，如果没有任何一个sk可读，那么select会调用schedule_timeout进入schedule循环，使得process进入睡眠。

* 3、如果在timeout时间内某个sk上有数据可读了，或者等待timeout了，则调用select的process会被唤醒

* 4、接下来select就是遍历监控的sk集合，挨个收集可读事件并返回给用户了，相应的伪码如下：

  ```
  for (sk in readfds)
  {
  	// 挨个查看每个 socket 此时的状态
      sk_event.evt = sk.poll();
      sk_event.sk = sk;
  }
  // 把 readfds、writefds、errorfds 中，是期望状态的 socket 的位图设置为 1，然后返回
  ret_event_for_process;
  ```



select 示例

* 操作 fd_set 位图的函数

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210122011626568.png)


* 示例

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210122011642286.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)






select存在两个问题：

* 被监控的fds需要从用户空间拷贝到内核空间
  * 为了减少数据拷贝带来的性能损坏，内核对被监控的fds集合大小做了限制，并且这个是通过宏控制的，大小不可改变(限制为1024)。
* 被监控的fds集合中，只要有一个有数据可读，整个socket集合就会被遍历一次调用sk的poll函数收集可读事件



有三个问题需要解决：

* 被监控的fds集合限制为1024，1024太小了，我们希望能够有个比较大的可监控fds集合 
* fds集合每次需要从用户空间拷贝到内核空间的问题，我们希望不需要拷贝 
* 当被监控的fds中某些有数据可读的时候，我们希望通知更加精细一点（即：希望能够从通知中得到有可读事件的fds列表，而不是需要遍历整个fds来收集）



## poll

poll和select非常相似

* select遗留的三个问题中，问题(1)是用法限制问题，问题(2)和(3)则是性能问题。

* poll并没着手解决性能问题，poll只是解决了select的问题(1)fds集合大小1024限制问题。

  




poll的函数原型

```c
/**
 * struct pollfd {
 *   int fd;         // file descriptor 
 *   short events;   // requested events to watch 
 *   short revents;  // returned events witnessed 
 * };
 */
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

* poll改变了**fds数组**的描述方式，使用了pollfd结构而不是select的fd_set结构，使得poll支持的fds集合限制远大于select的1024。




poll虽然解决了fds集合大小1024的限制问题，但是poll 仍然随着监控的socket集合的增加性能线性下降，并不适合用于大并发场景

* 并没改变大量描述符数组被整体复制于用户态和内核态的地址空间之间
* 以及个别描述符就绪触发整体描述符集合的遍历的低效问题。


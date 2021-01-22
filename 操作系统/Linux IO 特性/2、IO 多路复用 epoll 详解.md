## epoll 详解

epoll_create

```c
int epoll_create(int size);
```

* `epoll_create` 会创建一个 `epoll` 实例，同时返回一个引用该实例的文件描述符。
  * 返回的文件描述符仅仅指向对应的 `epoll` 实例，并不表示真实的磁盘文件节点。
  * 其他 API 如 `epoll_ctl`、`epoll_wait` 会使用这个文件描述符来操作相应的 `epoll` 实例。
* 当创建好 epoll 句柄后，它会占用一个 fd 值，在 linux 下查看 `/proc/进程id/fd/`，就能够看到这个 fd。
  * 所以在使用完 epoll 后，必须调用 `close(epfd)` 关闭对应的文件描述符，否则可能导致 fd 被耗尽。
  * 当指向同一个 `epoll` 实例的所有文件描述符都被关闭后，操作系统会销毁这个 `epoll` 实例。



epoll_ctl

```c
/**
 * epfd 即 epoll_create 返回的文件描述符，指向一个 epoll 实例
 * fd 表示要监听的目标文件描述
 *
 * event 表示要监听的事件（可读、可写、发送错误…）
 * 		struct epoll_event {
 * 			__uint32_t events;  // Epoll events 
 * 			epoll_data_t data;  // User data variable 
 *		};
 *
 * op 表示要对 fd 执行的操作，有以下几种：
 *	   EPOLL_CTL_ADD：为 fd 添加一个监听事件 event
 *     EPOLL_CTL_MOD：改变 fd 的监听事件 
 *	   EPOLL_CTL_DEL：删除 fd 的所有监听事件，这种情况下 event 参数没用
 */
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

* 监听文件描述符 `fd` 上发生的 `event` 事件。
* 返回值 0 或 -1，表示上述操作成功与否。



epoll_wait

```c
/**
 * epfd 即 epoll_create 返回的文件描述符，指向内核中一个 epoll 实例
 * events 是一个数组，保存就绪状态的文件描述符，其空间由调用者负责申请
 * maxevents 指定 events 的大小
 * timeout 类似于 select 中的 timeout。
 *     如果没有文件描述符就绪，即就绪队列为空，则 epoll_wait 会阻塞 timeout 毫秒。
 *     如果 timeout 设为 -1，则 epoll_wait 会一直阻塞，直到有文件描述符就绪；
 *     如果 timeout 设为 0，则 epoll_wait 会立即返回
 */
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);
```

* 这是 epoll 模型的主要函数，功能相当于 `select`。
* 返回值表示 `events` 中存储的就绪描述符个数，最大不超过 `maxevents`。





### 解决 fds 拷贝

* 需要监控的 fds 集合变化频率很低，没必要每次都重新准备整个fds集合
  * 只用增加一个改变已有 fds 的方法即可

* epoll 模型使用三个函数：`epoll_create`、`epoll_ctl` 和 `epoll_wait`。



epoll_ctl

* 引入了epoll_ctl系统调用，将高频调用的epoll_wait和低频的epoll_ctl隔离开。
  * 通过(EPOLL_CTL_ADD、EPOLL_CTL_MOD、EPOLL_CTL_DEL)三种操作来分散对需要监控的fds集合的修改，做到有变化才变更
  * 避免了select或poll高频、大块内存拷贝，变成epoll_ctl的低频、小块内存的拷贝。
* 同时，对于就绪的 fd集合返回的拷贝问题，epoll 通过内核与用户空间mmap(内存映射)同一块内存来解决。



epoll 在内核中组织 fd 的结构

* epoll通过epoll_ctl来对监控的fds集合来进行增、删、改，那么必须涉及到fd的快速查找问题，于是，一个低时间复杂度的增、删、改、查的数据结构来组织被监控的fds集合是必不可少的了。
* 在linux 2.6.8之前的内核，epoll使用hash来组织fds集合，于是在创建epoll fd的时候，epoll需要初始化hash的大小。于是epoll_create(int size)有一个参数size，以便内核根据size的大小来分配hash的大小。
* 在linux 2.6.8以后的内核中，**epoll使用红黑树来组织监控的fds集合**，于是epoll_create(int size)的参数size实际上已经没有意义了。





### 解决 fds 遍历 

epoll引入了一个中间层

* 通过上面的socket的睡眠队列唤醒逻辑，socket唤醒睡眠在其睡眠队列的wait_entry(process)的时候会调用wait_entry的回调函数callback。
  * 所以可以在callback中做任何事情。
* 为了做到只遍历就绪的fd，需要有个地方来组织那些已经就绪的fd
  * **一个双向链表(ready_list) 来组织就绪的 fd**
* 还有与select或poll不同的是，epoll的process不需要同时插入到多路复用的socket集合的所有睡眠队列中
  * **一个单独的睡眠队列(single_epoll_wait_list)**
  * process只是插入到中间层的epoll的单独睡眠队列中，process睡眠在epoll的单独队列上，等待事件的发生。
* 同时，引入**一个中间的wait_entry_sk**，睡眠在真正的 socket 睡眠队列上，它与某个socket sk密切相关
  * 其callback函数逻辑是将当前sk排入到epoll的ready_list中，并唤醒epoll的single_epoll_wait_list 上对应的 process
  * single_epoll_wait_list上睡眠的process的回调函数：遍历ready_list上的所有sk，挨个调用sk的poll函数收集事件，然后唤醒process从epoll_wait返回（整体逻辑如下）。 



1、epoll_ctl （示例添加事件 EPOLL_CTL_ADD）逻辑

* (1) 构建睡眠实体 wait_entry_sk，将当前socket sk关联给 wait_entry_sk，并设置wait_entry_sk的回调函数为epoll_callback_sk

* (2) 将wait_entry_sk加入内核中真正的 socket 睡眠队列上

* 回调函数epoll_callback_sk的逻辑如下：

  * 将之前关联的sk排入epoll的ready_list
  * 然后唤醒epoll的单独睡眠队列single_epoll_wait_list

  



2、epoll_wait 逻辑

* (1) 构建 epoll 睡眠队列上（single_epoll_wait_list）睡眠实体wait_entry_proc，将当前process关联给wait_entry_proc，并设置回调函数为epoll_callback_proc
* (2) 判断epoll的ready_list是否为空，如果为空，则将wait_entry_proc排入epoll的single_epoll_wait_list中，随后进入schedule循环（切换到其他内核线程，因为已经自己把当前内核线程设置为 sleep 了，所以不会再被调度，然后等待唤醒）。
* (3) wait_entry_proc被事件唤醒或超时醒来，wait_entry_proc将被从single_epoll_wait_list移除掉，然后wait_entry_proc执行回调函数epoll_callback_proc
* 回调函数epoll_callback_proc的逻辑如下：
  * 遍历epoll的ready_list（ready list 上的 fd 都是发生事件的），挨个调用每个sk的poll逻辑收集发生的事件（确定每个 fd 发生的哪种事件）
  * 将每个sk收集到的事件，通过epoll_wait传入的events数组回传并唤醒相应的process。





epoll 整个唤醒逻辑如下(对于可读事件而言)：

* (1) 协议数据包到达网卡并被排入socket sk的接收队列
* (2) 睡眠在sk的睡眠队列wait_entry被唤醒，wait_entry_sk的回调函数epoll_callback_sk被执行
* (3) epoll_callback_sk将当前sk插入epoll的ready_list中
* (4) 唤醒睡眠在epoll的单独睡眠队列single_epoll_wait_list的对应 wait_entry，wait_entry_proc被唤醒执行回调函数epoll_callback_proc
* (5) 遍历epoll的ready_list，挨个调用每个sk的poll逻辑收集发生的事件
* (6) 将每个sk收集到的事件，通过epoll_wait传入的events数组回传并唤醒相应的process。



epoll 巧妙的引入一个中间层解决了大量监控socket的无效遍历问题。

* 通过在中间层上为每个监控的socket准备了一个单独的回调函数epoll_callback_sk，而对于select/poll，所有的socket都公用一个相同的回调函数。
  * 正是这个单独的回调epoll_callback_sk使得每个socket都能单独处理自身，当自己就绪的时候将自身socket挂入epoll的ready_list。
* 同时，epoll引入了一个睡眠队列single_epoll_wait_list，分割了两类睡眠等待。
  * process不再睡眠在所有的socket的睡眠队列上，而是睡眠在epoll的睡眠队列上，在等待”任意一个socket可读就绪”事件。
  * 而中间wait_entry_sk则代替process睡眠在具体的socket上，当socket就绪的时候，它就可以处理自身了。



### ET 与 LT

ET vs LT - 概念

- Edge Triggered (ET) 边沿触发
  - socket的接收缓冲区状态变化时触发读事件，即空的接收缓冲区刚接收到数据时触发读事件
  - socket的发送缓冲区状态变化时触发写事件，即满的缓冲区刚空出空间时触发读事件
  - 即：仅在缓冲区状态变化时触发事件，比如数据缓冲去从无到有的时候(不可读-可读)

- Level Triggered (LT) 水平触发
  - socket接收缓冲区不为空，有数据可读，则读事件一直触发
  - socket发送缓冲区不满可以继续写入数据，则写事件一直触发
  - 即：符合思维习惯，epoll_wait返回的事件就是socket的状态




两种模式的本质：

* epoll_wait 逻辑里，当因为某个 socket  唤醒的时候，会遍历当前的 ready_list，决定上次的 sk 怎么处理
* 对于Edge Triggered (ET) 边沿触发（直接移除原先所有的 sk）：
  * 遍历epoll的ready_list，将sk从ready_list中移除，然后调用该sk的poll逻辑收集发生的事件

* 对于Level Triggered (LT) 水平触发（如果原先 ready list 中的某个 sk 关心的事件相同，重新加入）： 
  * 遍历epoll的ready_list，将sk从ready_list中移除，然后调用该sk的poll逻辑收集发生的事件
  * 如果该sk的poll函数返回了关心的事件(对于可读事件来说，就是POLL_IN事件)，那么该sk被重新加入到epoll的ready_list中。



即：

* 对于可读事件而言，在ET模式下，只有某个socket有新的数据到达，那么该sk才会被排入epoll的ready_list，从而epoll_wait 能收到可读事件的通知（对于边缘触发通常理解的缓冲区状态变化(从无到有)的理解是不准确的，准确的理解应该是是否有新的数据达到缓冲区）
* 在LT模式下，某个sk被探测到只要有数据可读，那么该sk会被重新加入到read_list，所以在该sk的数据被全部取走前，下次调用epoll_wait就一定能够收到该sk的可读事件(调用sk的poll逻辑一定能收集到可读事件)，从而epoll_wait就能返回。



ET vs LT - 性能

* 对于可读事件而言，LT比ET多了两个操作：
  * (1) 对ready_list的遍历的时候，对于收集到可读事件的sk会重新放入ready_list；
  * (2)下次epoll_wait的时候会再次遍历上次重新放入的sk，如果sk本身没有数据可读了，那么这次遍历就变得多余了。 
* 在服务端有海量活跃socket的时候，LT模式下，epoll_wait返回的时候，会有海量的socket sk重新放入ready_list。
  * 如果，用户在第一次epoll_wait返回的时候，将有数据的socket都处理掉了，那么下次epoll_wait的时候，上次epoll_wait重新入ready_list的sk被再次遍历就有点多余，这个时候LT确实会带来一些性能损失。
* 但事实是目前还没有实际应用场合的测试表面ET比LT性能更好。
  * 先不说第一次epoll_wait返回的时候，用户进程能否都将有数据返回的socket处理掉。
  * 在用户处理的过程中，如果该socket有新的数据上来，那么协议栈发现sk已经在ready_list中了，那么就不需要再次放入ready_list，也就是在LT模式下，对该sk的再次遍历不是多余的，是有效的。
  * 同时，我们回归epoll高效的场景在于，服务器有海量socket，但是活跃socket较少的情况下才会体现出epoll的高效、高性能。因此，在实际的应用场合，绝大多数情况下，ET模式在性能上并不会比LT模式具有压倒性的优势。 





ET vs LT - 对于 socket 是否阻塞的要求

* 在LT模式下，如果socket_fd还有数据可读，那么epoll_wait就一定能够返回，接着，我们就可以对该socket_fd调用recv或read读取数据。
* 然而，在ET模式下，尽管socket_fd还是数据可读，但是如果没有新的数据上来，那么epoll_wait是不会通知可读事件，除非有新的数据来了在处理（、
  * 因为上面说了每次从被某个 socket 唤醒后会从 ready_list 删除原先的 sk，然而对于 tcp 这种请求，都是每次收到一个完整报文时才去 sleep_list 唤醒对应的 entry



ET强制需要在 socket 的非阻塞模式下使用

* 即 epoll_wait返回socket_fd有数据可读，必须要读完所有数据才能离开。因为，如果不读完，epoll不会再通知
  * 如果有新的数据到来的时候，会再次通知，但是我们并不知道新数据会不会来，以及什么时候会来。
* 由于在阻塞模式下，无法通过recv/read来探测空数据事件，所以必须采用非阻塞模式，一直read直到EAGAIN。



ET 模式下死锁和socket饿死现象

* epoll_wait原本的语意是：监控并探测socket是否有数据可读(对于读事件而言)。
  * LT模式保留了其原本的语意，只要socket还有数据可读，它就能不断反馈，所以想什么时候读取处理都可以，只要再次poll的时候探测到有数据可以处理就会再次放入 ready_list。这样带来了编程上的很大方便，不容易死锁造成某些socket饿死。
  * 相反，ET模式修改了epoll_wait原本的语意，变成了“监控并探测socket是否有新的数据可读”。
* ET 模式下在epoll_wait返回socket_fd可读的时候，要小心处理，要不然会造成死锁和socket饿死现象。
  * 典型如listen_fd返回可读的时候，需要不断的accept直到EAGAIN。
  * 假设同时有三个请求到达，epoll_wait返回listen_fd可读，这个时候，如果仅仅accept一次拿走一个请求去处理，那么就会留下两个请求，如果这个时候一直没有新的请求到达，那么再次调用epoll_wait是不会通知listen_fd可读的，于是epoll_wait只能睡眠到超时才返回，遗留下来的两个请求一直得不到处理，处于饿死状态。





ET vs LT - 总结

- ET - 对于读操作
  - 当接收缓冲buffer内待读数据增加的时候时候(由空变为不空的时候、或者有新的数据进入缓冲buffer) ，此时会把 socket 放入 ready_list
  - 添加这种状态监听：调用 `epoll_ctl(EPOLL_CTL_MOD)` 来改变 socket_fd 的监控事件，也就是重新mod socket_fd 的 EPOLLIN事件，并且接收缓冲buffer内还有数据没读取
    - 不能是EPOLL_CTL_ADD 改变状态的原因是，epoll不允许重复ADD的，除非先DEL了，再ADD
    - 因为 epoll_ctl(ADD或MOD) 会调用sk的 poll逻辑来检查是否有关心的事件，如果有，就会将该sk加入到epoll的ready_list中，下次调用epoll_wait的时候，就会遍历到该sk，然后会重新收集到关心的事件返回。

- ET - 对于写操作
  - 发送缓冲buffer内待发送的数据减少的时候(由满状态变为不满状态的时候、或者有部分数据被发出去的时候) ，此时会把 socket 放入 ready_list
  - 添加这种状态监听：调用 epoll_ctl(EPOLL_CTL_MOD) 来改变socket_fd的监控事件，也就是重新mod socket_fd的EPOLLOUT事件，并且发送缓冲buffer还没满的时候。
- LT

  - LT - 对于读操作 LT就简单多了，唯一的条件就是，接收缓冲buffer内有可读数据的时候
  - LT - 对于写操作 LT就简单多了，唯一的条件就是，发送缓冲buffer还没满的时候

* 在绝大多少情况下，ET模式并不会比LT模式更为高效，同时，ET模式带来了不好理解的语意，这样容易造成编程上面的复杂逻辑和坑点。因此，建议还是采用LT模式来编程更为方便。

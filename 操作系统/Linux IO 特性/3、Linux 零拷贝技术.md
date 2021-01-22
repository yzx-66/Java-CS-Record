# 零拷贝
使用场景
* 在写一个服务端程序时（Web Server或者文件服务器），**文件下载**是一个基本功能。
* 这时候服务端的任务是：将服务端主机磁盘中的文件不做修改地从已连接的socket发出去，通常用下面的代码完成
	```c
	// 循环的从磁盘读入文件内容到缓冲区，再将缓冲区的内容发送到socket。 
	while((n = read(diskfd, buf, BUF_SIZE)) > 0)
	    write(sockfd, buf , n);
	```
	* 但是由于Linux的I/O操作默认是缓冲I/O。
	* 这里面主要使用的也就是read和write两个系统调用，在以上I/O操作中，发生了多次的数据拷贝。

Linux 传输文件的步骤
* 当访问某个文件时，会先拿到其 innodb，然后通过要读写的文件内偏移，算出逻辑盘块，然后在 innode 中得到物理盘块号，然后判断这个物理盘块是否有对应的 bffer_head 缓冲
	* 如果有，操作系统则直接根据 read 系统调用提供的 buf 地址，将内核缓冲区的内容拷贝到buf所指定的用户空间缓冲区中去。
	* 如果不是，操作系统则创建一个该盘块的 buffer_head，然后把对应的盘块读取这个 buffer_head 对应的内核缓冲。这一步目前主要**依靠DMA来传输**，然后再把内核缓冲区上的内容拷贝到用户缓冲区中。
* 接下来，write系统调用再把用户缓冲区的内容拷贝到网络堆栈相关的内核缓冲区中，最后socket再把内核缓冲区的内容发送到网卡上。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210122161807818.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)

上述过程存在的问题

* 共产生了四次数据拷贝，即使使用了DMA来处理了与硬件的通讯，CPU仍然需要处理两次数据拷贝
* 与此同时，在用户态与内核态也发生了多次上下文切换，无疑也加重了CPU负担。
* 在此过程中，我们没有对文件内容做任何修改，那么在内核空间和用户空间来回拷贝数据无疑就是一种浪费，而零拷贝主要就是为了解决这种低效性。

什么是零拷贝技术（zero-copy）？
* 零拷贝主要的任务就是**避免 CPU 将数据从一块存储拷贝到另外一块存储**
* 主要指的是内核到用户态间的拷贝，因为毕竟都是同一个物理内存


# mmap
内核 buffer 的起始地址和用户态的 buffer 的起始地址映射到同一物理页上
![在这里插入图片描述](https://img-blog.csdnimg.cn/202101221635415.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)


调用mmap()来代替read调用：
```c
buf = mmap(diskfd, len);
write(sockfd, buf, len);
```
* 整个拷贝过程会发生 4 次上下文切换，1 次 CPU 拷贝和 2 次 DMA 拷贝

用户程序读写数据的流程如下：
* 用户进程通过 mmap() 函数向内核（kernel）发起系统调用，上下文从用户态（user space）切换为内核态（kernel space）。
* 将用户进程的内核空间的读缓冲区（read buffer）与用户空间的缓存区（user buffer）进行内存地址映射。
* CPU利用DMA控制器将数据从主存或硬盘拷贝到内核空间（kernel space）的读缓冲区（read buffer）。
* 上下文从内核态（kernel space）切换回用户态（user space），mmap 系统调用执行返回。
* 用户进程通过 write() 函数向内核（kernel）发起系统调用，上下文从用户态（user space）切换为内核态（kernel space）。
* CPU将读缓冲区（read buffer）中的数据拷贝到的网络缓冲区（socket buffer）。
* CPU利用DMA控制器将数据从网络缓冲区（socket buffer）拷贝到网卡进行数据传输。
* 上下文从内核态（kernel space）切换回用户态（user space），write 系统调用执行返回。

mmap 存在的问题
* mmap 主要的用处是提高 I/O 性能，特别是针对大文件。对于小文件，内存映射文件反而会导致碎片空间的浪费，因为内存映射总是要对齐页边界，最小单位是 4 KB，一个 5 KB 的文件将会映射占用 8 KB 内存，也就会浪费 3 KB 内存。
* mmap 的拷贝虽然减少了 1 次拷贝，提升了效率，但也存在一些隐藏的问题。当 mmap 一个文件时，如果这个文件被另一个进程所截获，那么 write 系统调用会因为访问非法地址被 SIGBUS 信号终止，SIGBUS 默认会杀死进程并产生一个 coredump，服务器可能因此被终止。

其他进程截获的解决方案：
* 为SIGBUS信号建立信号处理程序
	* 当遇到SIGBUS信号时，信号处理程序简单地返回，write系统调用在被中断之前会返回已经写入的字节数，并且errno会被设置成success
	* 但是这是一种糟糕的处理办法，因为并没有解决问题的实质核心。
* 使用文件租借锁
	* 通常我们使用这种方法，在文件描述符上使用租借锁，我们为文件向内核申请一个租借锁，当其它进程想要截断这个文件时，内核会向我们发送一个实时的RT_SIGNAL_LEASE信号，告诉我们内核正在破坏你加持在文件上的读写锁。
	* 这样在程序访问非法内存并且被SIGBUS杀死之前，你的write系统调用会被中断。write会返回已经写入的字节数，并且置errno为success。
	* 在mmap文件之前加锁，并且在操作完文件后解锁：
	```c
	if(fcntl(diskfd, F_SETSIG, RT_SIGNAL_LEASE) == -1) {
	    perror("kernel lease set signal");
	    return -1;
	}
	/* l_type can be F_RDLCK F_WRLCK  加锁*/
	/* l_type can be  F_UNLCK 解锁*/
	if(fcntl(diskfd, F_SETLEASE, l_type)){
	    perror("kernel lease set type");
	    return -1;
	}
	```
# sendfile
从2.1版内核开始，Linux引入了sendfile来简化操作，让数据传输不需要经过user space
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210122164338893.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)
调用sendfile() 直接发送
```c
#include<sys/sendfile.h>
// in_fd 代表要发送文件的描述符
// out_fd 代表目标 socket 的描述符
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```
* 整个拷贝过程会发生 2 次上下文切换，1 次 CPU 拷贝和 2 次 DMA 拷贝

用户程序读写数据的流程如下：
* 用户进程通过 sendfile() 函数向内核（kernel）发起系统调用，上下文从用户态（user space）切换为内核态（kernel space）。
* CPU 利用 DMA 控制器将数据从主存或硬盘拷贝到内核空间（kernel space）的读缓冲区（read buffer）。
* CPU 将读缓冲区（read buffer）中的数据拷贝到的网络缓冲区（socket buffer）。
* CPU 利用 DMA 控制器将数据从网络缓冲区（socket buffer）拷贝到网卡进行数据传输。
* 上下文从内核态（kernel space）切换回用户态（user space），sendfile 系统调用执行返回。


sendfile 的比较
* sendfile 不仅减少了数据拷贝的次数，还减少了上下文切换，数据传送始终只发生在kernel space。
* 但 sendfile **只能将数据从文件传递到 socket 套接字上**，反之则不行。

sendfile 的文件截断问题
* 在我们调用sendfile时，如果有其它进程截断了文件会发生什么呢？假设我们没有设置任何信号处理程序，sendfile调用仅仅返回它在被中断之前已经传输的字节数，errno会被置为success。
* 如果我们在调用sendfile之前给文件加了锁，sendfile的行为仍然和之前相同，我们还会收到RT_SIGNAL_LEASE的信号。

sendfile 的改进
* 目前为止，已经减少了数据拷贝的次数了，但是仍然存在一次拷贝，就是**文件缓冲到 socket 缓冲的拷贝**。那么能不能把这个拷贝也省略呢？
* 借助于硬件上的帮助，可以**把文件缓冲的数据直接拷贝到网卡 DMA 接口的缓冲中，而不经过 socket 缓冲**
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20210122165327108.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)

* 仍然存在的问题：是需要硬件以及驱动程序支持的。

# splice
sendfile只适用于将数据从文件拷贝到套接字上，限定了它的使用范围。
* Linux在2.6.17版本引入splice系统调用，用于在**两个文件描述符中移动数据**：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210122165618933.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)
```c
#define _GNU_SOURCE         /* See feature_test_macros(7) */
#include <fcntl.h>
ssize_t splice(int fd_in, loff_t *off_in, int fd_out, loff_t *off_out, size_t len, unsigned int flags);
```
* 整个拷贝过程会发生 2 次上下文切换，0 次 CPU 拷贝以及 2 次 DMA 拷贝

用户程序读写数据的流程如下：
* 用户进程通过 splice() 函数向内核（kernel）发起系统调用，上下文从用户态（user space）切换为内核态（kernel space）。
* CPU 利用 DMA 控制器将数据从主存或硬盘拷贝到内核空间（kernel space）的读缓冲区（read buffer）。
* CPU 在内核空间的读缓冲区（read buffer）和网络缓冲区（socket buffer）之间建立管道（pipeline）。
* CPU 利用 DMA 控制器将数据从网络缓冲区（socket buffer）拷贝到网卡进行数据传输。
* 上下文从内核态（kernel space）切换回用户态（user space），splice 系统调用执行返回。


splice 的问题
* fd_in 或 fd_out，有一方必须是管道设备
* flags参数有以下几种取值：
	* SPLICE_F_MOVE ：尝试去移动数据而不是拷贝数据。这仅仅是对内核的一个小提示：如果内核不能从pipe移动数据或者pipe的缓存不是一个整页面，仍然需要拷贝数据。Linux最初的实现有些问题，所以从2.6.21开始这个选项不起作用，后面的Linux版本应该会实现
	* SPLICE_F_NONBLOCK：splice 操作不会被阻塞。然而，如果文件描述符没有被设置为不可被阻塞方式的 I/O ，那么调用 splice 有可能仍然被阻塞。
	* SPLICE_F_MORE： 后面的splice调用会有更多的数据。

* splice调用利用了Linux提出的管道缓冲区机制， 所以至少一个描述符要为管道。

# 写时复制
上面几种零拷贝技术都是减少数据在用户空间和内核空间拷贝技术实现的
* 但是有些时候，数据必须在用户空间和内核空间之间拷贝。这时只能针对数据在用户空间和内核空间拷贝的时机上下功夫了。
* Linux通常利用写时复制(copy on write)来减少系统开销，这个技术又时常称作COW。

思想
* 写时复制指的是当多个进程共享同一块数据时，如果其中一个进程需要对这份数据进行修改，那么就需要将其拷贝到自己的进程地址空间中。这样做并不影响其他进程对这块数据的操作，每个进程要修改的时候才会进行拷贝，所以叫写时拷贝。
* 这种方法在某种程度上能够降低系统开销，如果某个进程永远不会对所访问的数据进行更改，那么也就永远不需要拷贝。
* 可以参考我下面的博客
	* 写时复制发生的 fork 的 copy_mem 时： https://yzx66.blog.csdn.net/article/details/112913564
	* 写时会发生缺页中断：https://yzx66.blog.csdn.net/article/details/112913576



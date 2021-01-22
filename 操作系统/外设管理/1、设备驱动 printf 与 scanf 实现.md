# 驱动的基本原理

外设的工作原理

* 计算机外设的工作原理，即 CPU 对外设的使用主要由如下两条主线构成：

  * 第一条主线是从 CPU 开始，CPU 发送命令给外部设备，最终表现为 CPU 执行指令“out ax, 端口号”；
  * 第二条主线是从外设开始，外设在完成工作后或出现状态变化时通过中断通知 CPU，CPU 通过中断处理程序完成后续工作。

* 第一条主线的主题词是“发出命令”，第二条主线的主题词是“中断处理”

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/2021012100481910.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)






文件视图

* 让应用程序员通过命令来直接操作计算机外部设备的想法几乎不可行，这就引出了文件视图，不管是什么样的外设，操作系统都将其统一抽象成一个文件，程序员通过文件接口 open、read、write 来使用这些外设。

  * 例如向“显示器文件”里 write 了一个“Hello World!”字符串，那么就会在显示器出显示出“Hello World!”。

* 文件视图下上层用户使用外部设备的基本结构：

  * 采用了这样的统一结构以后，在上层用户眼里，对外部设备的操作和对文件的操作是完全一样的，上层用户可以完全忽略诸如外部设备端口号、设备指令格式等诸多细节。

  ```c
  main()
  {
      int fd = open(“/dev/xxx”);
      for (int i = 0; i < 10; i++)
      {
      	write(fd,i,sizeof(int));
      }
      close(fd);
  }
  ```

  

## printf---显示器驱动

从 printf 开始

* printf 是一个库函数，该库函数会将%d,%c等内容统一处理为字符串，然后以该字符串所在的内存地址 buf 和字符串长度count 为参数调用系统调用 write(1, buf, count)。

* write 的内核实现是 sys_write，所以 printf 的下一步就是 sys_write



sys_write

* 首先要做的事就是找到所写文件的属性，即到底是普通文件还是设备文件，如果是设备文件，sys_write 要根据设备
  文件中存放的设备属性信息具体分支到相应的操作命令中。
* 设备信息存放在描述文件本身（非文件内容）的数据结构中，这个数据结构就是著名的文件控制块（FCB，File Control Block）。

```c
// 为找到“文件”FCB，首先要做的工作就是从当前进程 PCB 中找到打开文件的句柄标识，即 fd（file descriptor），
// 对于显示器而言，这个 fd = 1，然后根据这个 fd 可以找到文件 FCB，即代码中的 inode
int sys_write(unsigned int fd, char *buf, int count)
{
    struct file* file;
    // current->filp 数据中存放当前进程打开的文件
    /**
     * 如果一个文件不是当前进程打开的，那么就一定是其父进程打开后再由子进程继承来的
     * 因为在 fork 的核心实现 copy_process(···) 中有这样的资源拷贝
     *       int copy_process(···)
     *       {
     *           *p = *current;
     *           for (i=0; i<NR_OPEN;i++)
     *           	if ((f=p->filp[i])) f->f_count++;	
     *
     * fd = 1 的文件对应标准输出，因为每个进程都可能用到标准输出，所以每个进程都会打开这个文件。
     * 既然所有进程都要打开这个设备文件，操作系统初始化时的 1 号进程会首次打开这个设备文件，然后其他进程继承这个文件句柄。
     *    	 void main(void) { if(!fork()){ init(); }
     *       void init(void)
     *       {
     *           // 由于这是该进程打开的第一个文件，所以对应的文件句柄 fd = 0
     *           // 显示器对应的设备文件就是文件“/dev/tty0”，其属性信息也已经存放在 sys_write 函数中的 inode 变量中了
     *           open(“/dev/tty0”,O_RDWR,0);
     *           // 使用了两次 dup，使得 fd = 1，fd = 2 也都指向了“/dev/tty0”的 FCB
     *           dup(0);
     *           dup(0);
     *           execve(”/bin/sh”,argv,envp);	
     */
    file = current->filp[fd];
    /**
     * 说明：FCB 是 innode，每个物理文件对应一个，是全局唯一的，放在 FCB 数组（磁盘起始扇区位置）
     *      file 是每次打开都会产生一个，保存这次打开一些操作结果的，例如这次打开操作到哪个位置
     *      即：一个 FCB（innode） 可以对应多个 file
     */
    inode = file->f_inode;
```



沿着 sys_write 继续向下

```c
int sys_write(unsigned int fd, char *buf,int cnt)
{
    // 根据 inode 中的信息判断该文件对应的设备是否是一个字符设备
    if(S_ISCHR(inode->i_mode))
        // 分支到函数 rw_char(WRITE,inode->i_zone[0], buf, cnt) 中去执行
        //  inode->i_zone[0] 中存放的就是该设备的主设备号和次设备号。
    	return rw_char(WRITE,inode->i_zone[0], buf, cnt);
```



rw_char

```c
int rw_char(int rw, int dev, char *buf, int cnt)
{
    // rw_char 中以主设备号（MAJOR(dev)）为索引从一个函数表crw_table 中要找到和终端设备对应的读写函数 rw_ttyx，
    // 然后调用这个函数。
    crw_ptr call_addr = crw_table[MAJOR(dev)];
    call_addr(rw, dev, buf, cnt);
    
static crw_ptr crw_table[] = {···, rw_ttyx, ··· };

static int rw_ttyx(int rw, unsigned minor, char *buf, int count)
{
    // 函数 rw_ttyx 中根据是设备读操作还是设备写操作继续分支。
    // 显示器和键盘合在一起构成了终端设备 tty，显示器只写，键盘只读。
	return ((rw==READ)? tty_read(minor,buf): tty_write(minor,buf));
}
```



tty_write

````c
int tty_write(unsigned channel,char *buf,int nr)
{
    struct tty_struct *tty;
    // tty_write 首先获得一个结构体 tty_struct，主要目的是在这个结构体中找到队列 tty->write_q
    // 站在用户的角度，输出到显示器就是输出到这个队列中。最终要等到合适的时候，由操作系统统一将队列中的内容输出到显示器
	// 上，这就是著名的缓冲机制。
    //   缓冲是指两个速度存在差异的设备之间通过缓冲队列来弥补这种差异的一种方式，
    //   具体而言，就是高速设备将数据存到缓冲队列中，然后高速设备去做其他事，低速设备在合适的时候从缓冲队列中取走内容进行输出，
    //   从而高速设备不用一直同步等待低速设备，提高系统的整体效率
    tty = channel+tty_table;
    
    // 在写显示队列之前，需要判断显示队列是否已满
    // tty_write 是生产者，用 sleep_if_full(&tty->write_q) 进行“P 操作”，如果发现队列已满，就睡眠等待
    sleep_if_full(&tty->write_q);
    
    char c, *b=buf;
    while(nr>0 && !FULL(tty->write_q))
    {
        // 如果 tty->write_q 队列没有满，那么就从用户态内存中逐个取出字符 c，即执行 c = get_fs_byte(b)
        c = get_fs_byte(b);
        // 获得 c 以后要进行一些判断，如果是换行，即 if(c==‘’̊)，则将 13 放入 tty->write_q 中；
        // 如果设置了大写标志，则将 c 变成大写字母后再放入tty->write_q 中，等等。
        if(c==‘’̊){PUTCH(13,tty->write_q); continue; }
        if(O_LCUC(tty)) c = toupper(c);
        PUTCH(c,tty->write_q);
        b++; nr--;
    } //输出完事或写队列满
    
    // tty 结构体中 write 函数指针来进行真实的显示器输出
    tty->write(tty);
}
````



con_write（tty->write 调用的函数）

```c
struct tty_struct tty_table[] = {{con_write,{0,0,0,0,””},{0,0,0,0,””}},{},···};
void con_write(struct tty_struct *tty)
{
    GETCH(tty->write_q,c);
    if(c>31 && c<127)
    {
        /**
         * 核心代码是一个嵌入式汇编，具体完成的工作是：
         *   mov c, al      # 将要输出的字符放在寄存器 ax 的低 8 位，
         *   mov attr, ah   # 将显示属性 attr 放到 ax 的高 8 位，
         *   mov ax, [pos]  # 然后将 ax 输出到地址 pos 处。
         *                     不是要“out”呢，怎么变成了 mov？实际上这里的 mov 和 out 没有本质区别，
         *					   计算机硬件原理告诉我们，外设可以独立编址，也可以和内存统一编址。
         *					   如果是独立编址就用“out”指令，如果统一编址就用“mov”指令。
         */
        __asm__(“movb attr, %%ah”
        “movw %%ax,%1”::”a”(c),”m”(*(short*)pos):”ax”);
        // con_write 中每输出一个 ax 都让 pos 加 2，这是必然的，因为 ax 就是两个字节。
        /**
         *  pos 的初始值：在初始化函数 con_init 中，调用函数 gotoxy 将 pos 的值初始化为 
         *    origin+[0x90001]*video_size_row +([0x90000]«1)。
         *  90000 这个数字不陌生吧，在系统启动的 setup.s 时利用 BIOS 中断将当前光标的行、列位置取出来
         *  放到了 0x90000 和 0x90001 处。而 origin 是显存在内存中的初始位置。
         *  因此初始化以后 pos 就是开机以后当前光标所在的显存位置。
         */
        pos+=2;
    }
}
```



小结

* printf → write → sys_write → rw_char → rw_ttyx → tty_write →write_q → con_write → “mov ax, [pos]”
* 这条从 printf 到“mov ax,[pos]”的文件视图路线全部有了。



## scanf---键盘驱动

从键盘中断开始

* 硬件手册告诉我们，按下键盘会产生 0x21 号中断，所以整个故事要从设置 0x21 号中断的中断处理函数开始。

```assembly
void con_init(void) { set_trap_gate(0x21, &keyboard_interrupt); }

keyboard_interrupt:
	# 从键盘的 0x60 端口上获得按键扫描码
    inb $0x60,%al			 
    # 根据这个扫描码调用不同的处理函数来处理各个按键，即调用 call key_table(,%eax,4)。
    # key_table:
    #    .long none,do_self,do_self,do_self  //对应扫描码 00-03
    #    .long do_self, ···, func, scroll, cursor // 绝大多数按键（如字母键、数字键等）都用 do_self 函数来处理
    call key_table(,%eax,4)  
    ······
    push $0
    call do_tty_interrupt
```



do_self

```assembly
do_self:
	# 从键盘对应的 ASCII 码表（key_map）中以当前按键的扫描码（存在寄存器 EAX 中）为索引找到当前按键的 ASCII 码
    lea key_map, %ebx
    # 将按键对应的 ASCII 码放入 al
    movb (%ebx, %eax), %al  
    # 找到 tty 结构体中的 read_q 队列
    # 键盘和显示器使用了同一个 tty 结构体 tty_table[0]，只是键盘使用的读队列，而显示器使用的写队列。
    movl table_list, %edx
    movl head(%edx),%ecx
    # 将 ASCII 码放到缓冲队列 read_q 
    movb %al,buf(%edx,%ecx)
key_map: .byte 0,27 .ascii “1234567890-=“···
```



do_tty_interrupt

```c
void do_tty_interrupt(int tty)
{
    // 调用 copy_to_cooked(tty_table[0])，来处理键盘 ASCII 码所存放的那个队列，即 tty->read_q。
	copy_to_cooked(tty_table+tty);
}

void copy_to_cooked(struct tty_struct *tty)
{
    // 从read_q队列中取出字符,并将该字符放在 tty->secondary 队列中
    GETCH(tty->read_q,c);
    PUTCH(c,tty->secondary);
    ······
    // 同时唤醒等待这个队列上的进程
    wake_up(&tty->secondary.proc_list);
```



两条线（用户和设备）将同步机制连接在一起

* 文件操作是由用户发起的，即用户启动了一个进程调用“read”来发起设备读操作
  * 该进程会在文件视图路线中阻塞，因为这个时候设备还没有将进程要读的东西准备好。
* 设备中断是由设备动作发起的，由操作系统的中断处理函数负责处理，两条线之间通过上述同步机制连接在一起。
  * 设备开始工作，工作完成以后会中断 CPU，操作系统在设备中断处理时，会将设备上的内容放入到内存缓冲中，并唤醒阻塞等待的进程，醒来的那个进程从缓冲取出内容进行处理。



用户发起的文件操作是scanf 开始

* 而 scanf 调用的是 sys_read(0,buf,count)，其中 fd = 0 表示标准输入。
* 找到设备文件的 FCB（即 innode 物理盘块索引） 以后可以发现这个设备文件仍然是/dev/tty0，根据 printf的文件视图路线不难类推，通过一系列分支后最后会执行 tty_read。
* 而 tty_read的核心就是要和键盘中断处理程序中的 wake_up 接上，所以这里面有句非常重要的语句：sleep_if_empty(&tty->secondary)。

```c
int tty_read(unsigned channel, char * buf, int nr)
{
    sleep_if_empty(&tty->secondary);
    // 一旦 tty->secondary 中有内容了，即键盘中断处理程序中将按键的 ASCII 码放进去
    do {
        // 将 tty->secondary中的字符逐个取出来
        GETCH(tty->secondary,c);
        tty->secondary.data--;
        // 拷贝回用户内存 buf 中（put_fs_byte(c,b++)）。
        // 到现在，按键对应的 ASCII 字符已经被逐个放到用户缓存区 buf 中了
        put_fs_byte(c,b++);
    } while (nr>0 && !EMPTY(tty->secondary));
```



小结

* 现在，keyboard_interrupt → inb0x60,al → do_self → read_q → copy_to_cooked → secondary → wake_up → tty_read → rw_ttyx → rw_char → sys_read → read →scanf

* 这条从键盘中断到“inb 0x60,al”最后到 scanf 的路线也有了。


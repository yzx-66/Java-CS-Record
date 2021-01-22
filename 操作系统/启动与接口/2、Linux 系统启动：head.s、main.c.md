## 3、head.s

* 这是进入 32 位保护模式以后要执行的第一段代码 head.s

  

对操作系统管理资源的关键数据结构进行初始化，但初始化之前准备工作还没有做完：

  * （1）设置中断表 IDT，因为从现在开始操作系统不再使用 BIOS 中断了，实际上将 system 从 0x10000 挪到 0x0地址处，BIOS 中断就没法使用了，因为 BIOS 中断向量表就放置在 0 地址处；另一方面，接管中断是操作系统必须要做的事，因为不同的操作系统遇到同一中断（如时钟中断）要做的事情不会相同。

  * （2）设置 GDT 表，虽然在 setup 中建立了 GDT 表，但那是为了执行“jmpi 0,8”而临时建立的，现在进入了 system模块，需要重新建立。

    * 同时这个 GDT 表也变得更长了，这是要为后面的 LDT 留下空间（GDT 是全局的，LDT 是每个进程的，因为每个进程的视角下都是规整的从 0 开始连续的内存，所以 CS 会重复，所以要为每个进程设置单独的 LDT，即进程 LDT 表的入口放在 GDT 表中）

      ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210121000414430.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)

  * （3）设置页表，实际上，进入 32 位保护模式以后，寻址方式要更加复杂，即“GDT[CS]+EIP”算出来的地址仍然不直接打到地址总线上，通常还要用这个地址再去查一次页表才得到真正的“物理地址”打到地址总线上。

    * 因为 GDT 得出来的是一个 32 位的段基址，但是如果计算实际地址时直接用该段基址 + EIP，那么就必须给该段开辟一个连续空间，那么空间利用率太低，所以把一个段又分为了又分为了许多页，每页 4KB。

    * 线性地址（32位） =  该 32 位基址 + EIP，然后用该 32 位的线性地址再去查页表（页目录的起始地址放在 CR3 寄存器）

      ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210121000439786.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)






设置 GDT 和 IDT

* 和 GDT 表相似，并且和 setup.s 中的设置没有太大区别，就是做出两段包含连续 8 个字节的内存数组，每个数组项占 8 个字节，分别作为 IDT表和 GDT 表。再用 lidt/lgdt 指令来将这个表的起始地址和长度限制放到关键的寄存器 IDTR/GDTR 中

* IDT 表项全部初始化为全 0，即现在所有的中断都不好使，要等到后面给各个模块（如时钟）初始化时，会设置相应的中断处理程序入口地址到各个表项中，所以 idt 要被设置为“全局变量”

  ```assembly
  .global idt
  mov $0x10, %eax
  mov %ax,%ds
  lidt idt_desc
  
  idt_desc:
  .word 256*8-1
  .long idt
  .align 3
  idt: .fill 256,8,0
  
  .global gdt
  lgdt gdt_desc
  
  gdt_desc:
  .word 256*8-1
  .long gdt
  idt: .fill 256,8,0
  gdt: .quad 0x0000000000000000
  .quad 0x00c09a0000000fff
  .quad 0x00c0920000000fff
  .quad 0x0000000000000000
  .fill 252, 8, 0
  ```




设置页表 

* 满足下面的条件

  * system 模块的页表被建立，且存放在从内存 0 地址开始处的 5 个长度为 0x1000（4K）的内存上，即 CR3 寄存器里存放的 0x0。

  * system 模块页表的查找规则为 PageTable[X] = X，即在经过 GDT 后，得到的 32 位线性地址里， 21~12 位页表索引项对应的真实物理页的起始位置为 0，并且后 14 位要为 0（即等于 EIP）

  * 即：对于操作系统的 system 模块而言，程序中使用偏移地址和实际打到物理内存上的物理地址实际上是一样的，这样做可以省去很多麻烦（尽管两个地址是一样的，但是每次取出指令、执行指令时取内存中的操作数等时，都要完成整个地址映射过程）。

    ```
    PageTable[GDT[CS/DS] + 程序中的偏移地址]
    = GDT[CS/DS] + 程序中的偏移地址
    = 0 + 程序中的偏移地址
    = 程序中的偏移地址
    ```

* 填写页表

  * 操作：填写那 5 个长度为 4K 的页表、设置页表寄存器 CR3、以及启动页表电路这三个部分。

  ```assembly
  pg_dir: ; 这里是 head.s 的头
  .org 0x1000
  .org 0x2000
  .org 0x3000
  .org 0x4000
  .org 0x5000
  
  setup_paging:
  ······ ;填写页表
  
  ; 设置页表寄存器 CR3（值为 0x0）
  xor %eax, %eax
  movl %eax, %cr3
  
  ; 启动页机制
  ; 仍然是那个 CR0 寄存器，启动 32 位保护模式是将 CR0 的最后一位置成 1，启动分页机制是需要将第一位设置为 1
  movl %cr0, %eax
  orl $0x8000000, %eax
  mov %eax, %cr0
  ```

* 此时内存布局

  * 实际上，操作系统启动这里的所有工作都是为了形成这张内存图。现在这张内存图有了，操作系统的准备就算都完成了，现在操作系统可以开始初始化
    了。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210121000500161.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)





head.s 的最后一段代码

* 负责跳到操作系统的初始化代码

  ```assembly
  .org 0x5000
  ; lss 命令用来设置栈，这条指令将一个 6 字节内存中的前 4 个字节赋给 ESP 寄存器，后 2 个字节赋给 SS 寄存器。
  ; 栈是函数跳转的基础，所以首先要设置栈
  lss stack_start, %esp
  
  ; 将标号 L6 压入，在 main() 函数返回以后会跳转到 L6 执行，这是一个死循环
  ; 所以 main 应该是一个永远都不能退出的函数，否则计算机就要“死机”了。
  pushl $L6
  
  ; 要从汇编程序中跳到 C 函数 main(){···} 去执行，即执行 “jmp $main”
  jmp $main
  
  L6: jmp L6
  ```



## 4、main.c

* main() 的主要功能就是初始化各种管理软、硬件资源的数据结构



初始化

* 可以调用各种初始化函数来完成相应的初始化，比如调用 mem_init() 用来初始化内存，调用 hd_init() 用来初始化硬盘，其他需要初始化的内容都可以在这里加上相应的 init 函数

  ```c
  #define EXT_MEM_K = (*(unsigned short *)0x90000)
  
  void main(void)
  {
      // 结束地址是由0x90000 处取出的内容决定，0x90000 这个地址是不是很熟悉啊？
      long memory_end = 1«20 + EXT_MEM_K«10;
      // 而一页的大小操作系统就设为 4K，当不足 1 页的内存就丢弃。
      memory_end &= 0xfffff000;
      // 此处的起始地址为 4M，这是因为 0 - 1M 交给了系统内核 system，1M - 4M 将交给磁盘高速缓存，所以 4M 以后的内存才是让用户应用程序使用的
      long memory_start = 4*1024*1024;
      // mem_init() 的两个参数是要管理的起始内存地址和结束内存地址
      mem_init(memory_start,memory_end);
      
      hd_init();
      ······
  }
  ```

* mem_init介绍

  * mem_init(start_mem, end_mem) 函数实现在内存管理的主要文件 memory.c中。具体工作就是用一个数组来表示对应的页是否空闲，初始化为全部空闲，即为全 0。这个数组放在一个全局数组变量 mem_map 中。
  * i = 0; length »= 12; while (length-- > 0) { mem_map[i++]=0; }。



让操作系统开始运转

* main() 函数中的最后四句话

  ```c
  void main(void)
  {
      ······
      sti()
      move_to_user_mode();
      // init() 会启动一个 shell，就是命令窗口
      if (!fork()) { init();}
      for(;;) { pause();}
  }
  ```

  



总结：

* 操作系统启动的主要工作就三项：系统准备、系统初始化和系统运转进入 shell。
* 系统准备又包括读入内核、启动保护模式、设置各种表等

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210121000519240.png)



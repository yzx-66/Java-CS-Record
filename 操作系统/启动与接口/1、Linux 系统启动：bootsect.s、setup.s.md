# 系统启动

总体启动过程：

* （1）BIOS 读入了操作系统的第一个扇区 bootsect.s 文件，然后执行 bootsect.s 执行；
* （2）bootsect.s 又读入了操作系统的 setup.s 文件，将来交给 setup.s 执行，在其中会完成一些操作系统设置工作；
* （3）接下来 bootsect.s 还要读入操作系统的主体模块 system 模块，在setup.s 执行完成以后到 system 执行中。



在计算机加电以后，硬件电路会初始化设置PC 寄存器的值。

* 对于 IBM PC 而言，这个寄存器的初始设置为 0xFFFF0。由于 IBM PC 的寻址方式，PC=0xFFFF0 的物理实现是设置 CS 和 IP 这两个寄存器的值，即CS=0xFFFF，IP=0x0000。
* 在 IBM PC 刚一启动时，计算机会首先工作在实模式下，实模式下取出指令的具体物理实现是首先将段寄存器 CS 中的数值左移 4位再和段内偏移寄存器 IP 中的值相加后形成一个地址，然后将这个地址放到地址总线上去取出内存中存放的指令，即 PC=CS≪4+IP，此处正是 0xFFFF0。



 0xFFFF0 中取出来的指令是什么

* 内存是RAM，属于易失性存储器，没有电时 RAM 中不会存放任何内容，因此刚一上电时 RAM 中不可能有任何实际内容。所以是计算机硬件厂商提供一段 ROM 存储，IBM PC 的0xFFFF0 就指向这样一段 ROM。
  * 在 IBM PC 中，这段 ROM 被称为 BIOS（BasicInput Output System，基本输入输出系统），其中放置的代码是对基本硬件的测试代码，如对主板、内存等硬件的测试，同时还提供了一些让用户调用硬件基本输入/输出功能的子程序
* CPU 从这段 ROM 中取出的指令要完成：测试各种硬件是否正常工作，如果测试出现了异常，则停止启动
  * 如果硬件测试正常，就利用 BIOS 的输入功能将启动磁盘的启动扇区内容读入到内存的 0x7C00 地址处，并设置寄存器 CS=0x07C0，IP=0x0000。



## 1、bootsect.s

* 引导扇区代码（bootsect.s）



操作系统第一个要编写的文件就是这个引导扇区要存放的程序代码，我们通常将这个程序文件命名为 bootsect.s。

* 这是一个汇编文件，之所以要使用汇编语言来编写是因为要让开机过程严格地遵照我们的要求进行，用 C 语言写程序虽然要简单的多，那 C 程序经过编译以后很多细节是不由我们控制的。
* 通常 bootsect.s 会包括如下一些核心代码



将 07C00 的 bootsect 程序移动到 90000 处，并跳转

```assembly
BOOTSEG = 0x07C0
INITSEG = 0x9000
mov ax,#BOOTSEG
mov ds,ax
mov ax,#INITSEG
mov es,ax

; 将内存中开始地址为 DS:SI 的 256个字移动到开始地址为 ES:DI 的地方
;   这个移动是解释执行“rep, movw”指令的结果，其中的 w 就是要移动一个字 w，对应两个字节，所以整个移动 512 个字节，正好是一个扇区的大小
; 将内存 0x7C00 处的 512 个字节（正好就是 bootsect.s 的全部程序）移动到从内存地址 0x90000 开始的一段内存中。
mov cx,#256
sub si,si
sub di,di
rep
movw

; 设置 CS 寄存器为 INITSEG，IP 寄存器为标号 go，显然就是设置 PC=0x90000+go
jmpi go,INITSEG
```



读入剩下的引导程序 setup.s

```assembly
	SETUPLEN = 4
go :mov ax,#INITSEG
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, #0xFF00
   
    ; 寄存器 DH=0x00 表示要读的磁盘扇区所在的磁头号为 0，寄存器 DL=0x00 表示要读扇区所在的驱动器号为 0
    ; 寄存器 寄存器 CH=0 表示要读的磁盘扇区所在的磁道号为 0，CL=0x2，表示要读的磁盘扇区从 2 号扇区开始
    ; 寄存器 ES:BX 和在一起形成的内存地址表示从磁盘读入的内容要放到的内存起始地址，此处是 0x90200。正好是移动后的 bootsect.s 的后面。
    ; 寄存器 AH=0x02 表示要读磁盘内容到内存，寄存器 AL=0x04 表示要读如 4 个扇区
    mov dx,#0x0000
    mov cx,#0x0002
    mov bx,#0x0200
    mov ax,#0x0200+SETUPLEN

    ; 这是一个 BIOS 中断,是一个读写磁盘的中断调用。
    int 0x13
```





显示器输出启动图像

```assembly
    ; AH 中的值通常被称为 BIOS 调用功能号, AH=0x03 是调用 BIOS 中断 int 0x10 时，取出当前光标的位置
    mov ah,#0x03
    xor bh,bh
    ;  BIOS 中断 int 0x10，该中断的作用是在屏幕上输出信息。
    int 0x10
    
    ; “mov cx,#24” 语句表示要显示 24 个字符
    ; DH 寄存器会存放光标所在的行，而 DL 存放光标所在的的列
    ; BL=0x07 用来设置显示字符的属性，其中 7 表示显示正常的黑底白字，当然可以通过修改这个值来显示出红字、闪烁等。
    ; ES:BP 用来说明输出字符串所在的内存地址
    ; 寄存器 AH=0x13 这个功能号调用 BIOS 中断 int 0x10时，会在屏幕上输出信息
    mov cx,#24
    mov bx,#0x0007
    mov bp,#msg
    mov ax,#0x1301
    int 0x10
    
	······
	
	  ; 13表示回车、10 表示换行
msg: .byte 13,10  
    .ascii ”Loading System ...”
    .byte 13,10,13,10
```





将所有的操作系统代码从磁盘读入到内存

* 准备工作
  * 初始化磁盘一些信息（磁道上扇区的个数），还有读取的起始位置（磁道、磁头、扇区起始位置、读到内存哪个位置）

```assembly
    ; 通过功能号 AH=0x08 调用 0x13 号BIOS 中断,可以获得每个磁道的扇区个数
    ; DL=0 表示要读取 0 号驱动器
    SYSSEG = 0x1000
    mov dl, #0x00
    mov ah, #0x08
    int 0x13
    
    ; 每个磁道的扇区个数会放置在寄存器 CL 的低 6 位中
    ; 获得每个磁道的扇区个数是为了将来读入系统模块时读取整个磁道作准备的，因为系统模块的尺寸通常都要大于一个磁道的容量。
    mov ch, #0x00
    and cl, #0x3F
    mov sectors, cx
    
    ; 设置 ES:BX，要读取磁盘信息，必须要设置存放读出内容的内存起始位置，此处将这个开始地址设置为 0x10000
    mov ax,#SYSSEG
    mov es,ax
    xor bx, bx
    
; 一个磁道的扇区总数
sectors: .word 0
; 已经读取某个磁道的扇区个数
sread: .word 1+SETUPLEN
; head = 0 表示使用上面的那个磁头（给上面的磁头的上电），head = 1 使用下面的磁头（应该用的软盘，把系统模块代码应该都放到了同一个盘面上）。
head: .byte 0
; 用来标识是哪个磁道
track: .word 0
```

* 真正的读入系统模块
  * 用一个循环实现一个磁道一个磁道的读入，同时随着磁道的读入地址 ES:BX 跟着不断往前移动（因为读的每个字节都会放到对应的 EX:BX 位置），直到系统模块被全部读入
  * 循环的思想是：先确定本次要读入的扇区个数（根据当前磁道剩余未读的磁道个数，和 BX 剩余能表示的偏移个数） -> 送 磁道、磁头、起始扇区、要读入的扇区个数、读进的内存地址 到寄存器，然后调用中断 -> 为下次读取做准备，根据本次读取的扇区个数+已经读取的该磁道的扇区个数，而后判断这个磁道是否读完，从而更改 trace 和 hand；之后更改 bx、es 然后接着读取

```assembly
        ; 系统模块大小，占用多少扇区，可以在编译操作系统时计算出这个数值
        SYSSIZE = 0x3000
        ENDSEG = SYSSEG + SYSSIZE
		
		 ; 循环条件是判断 AX 是否大于 ENDSEG=SYSSEG + SYSSIZE
		 ; 由于 AX 初始化为 SYSSEG，所以如果 AX 增加了 SYSSIZE，即系统模块尺寸以后，磁盘读取工作就结束了
rp_read: cmp ax, #ENDSEG
        ja end_read
        
        ; CX 表示当前磁道剩下要读取的字节个数，被赋值为 sectors - sread
        ; 	sread 被初始化为 1+SETUPLEN，SETUPLEN是 setup.s 的长度，此处占 4 个扇区
        ; 	sectors 中存放的是一个磁道的扇区总数，那么 sread 中存放的就是当前磁道中已经读入的扇区数量
        mov ax, sectors
        sub ax, sread
        mov cx, ax
        
        ; system 模块从第 5 个扇区开始。
        ; 接下来将 CX 左移 9 位，相当于乘了 512，一个扇区 512 个字节,现在 CX 正好是当前磁道剩下要读的字节数量。
        shl cx, #9
        
        ; BX 表示的读入目标内存段的段内偏移，每读完一个磁道，会把 CX 的值加到 BX 上
        ; 因为 BX 表示的读入目标内存段的段内偏移，所以读完一个磁道以后需要将当前磁道读入的字节数量加到现有偏移上形成新的偏移。
        add cx, bx
        
        ; 这里会出现两种情况：
        ;   （1）如果累加以后的段内偏移仍然没有超过 2^16 =64KB，此时 BX（因为 BX 是一个 16 位寄存器）仍然能表示这个新的偏移，
        ;       此时开始读取这个磁道的剩余扇区即可，对应的是跳转到 ok_read，其中 jnc, je 表示执行语句“add cx,bx”没有发生溢出；
        ;   （2）如果读入以后超过了 64KB，由于 BX 的 16 位限制，此时有些内容读入以后 ES:BX 会不知道该往哪里存放，
        ;       所以只能读到和当前 BX 加在一起等于 64KB 的那一部分，剩下的内容就只能等到下一次 rp_read 循环再读，
        ;       那时 ES 会前进64KB 到下一个段，BX 也变成 0 了。
        jnc ok_read
        je ok_read
        
        ; 出现情况（2）时，需要用 64KB 减去当前偏移量 BX 来计算在当前磁道上要读入多少内容，用 AX = 0 减去 BX 就是 64KB-BX。
        ; 而此时 AX 中存放就是此次要读满 64KB 需要读取的字节数，这个值右移 9 位（即除以 512 字节）算出的就是此次要读的扇区数
        xor ax, ax
        sub ax, bx
        shr ax, #9

		; 读取用函数 read_track 实现。
ok_read:call read_track
```

```assembly
; 读取哪个磁道/哪个磁头？ 信息放在 track/head中；
; 起始扇区信息，是什么？ 即 sread+1；
; 要读多少个扇区？ 两种情况各自算出了一个数，但都存放在了寄存器 AX中；
; 还有就是要读到的内存位置？具体信息存放在了 ES:BX 中。
read_track:
        push ax
        push bx
        push cx
        push dx
        
        ; 寄存器 DL 放置驱动器号 0
        ; 寄存器 CH 放置磁道号
        ; 寄存器 CL 放置起始扇区号 sread+1
        ; 寄存器 AL 中存放的是要读的扇区数
        ; ES:BX 是目标内存地址
        mov dx, track
        mov cx, sread
        inc cx
        mov ch, dl
        mov dh, head
        mov dl, #0
        ; AH=2 是功能号
        mov ah, #2
        ; 完成读磁盘的功能
        int 0x13
        
        pop dx
        pop c
        pop bx
        pop ax
        ret
```





为下一次读入做准备

* 执行到现在，当前磁道上的全部（对应情况（1））或者部分（对应情况（2））内容已经读入到内存中了

```assembly
        SETUPSEG = 0x9020
        
        ; AX 寄存器中的内容是此次读入的扇区个数，将其和 sread（开始读的扇区号）加和以后，和 sectors 对比
        ;   如果不等，说明当前磁道还有扇区没有读完，就执行语句“jne goon_read”继续读当前磁道。
        ;   否则说明当前磁道已经读完了，此时就要移动到下一个磁道了
        mov cx, ax
        add ax, sread
        cmp ax, sectors
        jne goon_read
        
        ; 1 减 head，如果不为 0，说明当前的 head 值为 0，即当前读取的是上面的那个磁道，现在要去读取下面的那个磁道
        ;    如果不为 0，说明当前的 head 值为 0，即当前读取的是上面的那个磁道，现在要去读取下面的那个磁道，应该 track 不变，head 变为 1
        ;    如果为 0，那么就要读取下一个磁道
        mov ax, #1
        sub ax, head
        jne nexthead_read
        inc track
        
nexthead_read: mov head, ax
		xor ax, ax
		
; 更新 sread、es、bx，然后接着读取
goon_read: mov sread, ax
        shl cx, #9
        ; 增加 bx
        add bx,cx
        ; 没有溢出
        jnc rp_read
        
        ; 增加 es，清空 bx
        mov ax,es
        add ax,#0x1000
        mov es, ax
        xor bx,bx
        jmp rp_read
        
end_read:
		; 执行 setup
        jmpi 0, SETUPSEG
        .org 510
        .word 0xAA55
```





bootsect 模块到此结束，可以总结一下

* （1）将磁盘上从第 2 到 5 的四个扇区构成的 setup 模块读入到了内存的0x90200 处；
* （2）然后打出一个 Logo，并在这个 Logo 的“掩护”下；
* （3）从磁盘的第 6 个扇区开始读入了长度为 SYSSIZE 的操作系统 system 模块，并将其存放到内存的 0x10000 处。



软驱上操作系统各模块

* 磁盘上的第 1 个扇区必须放置 bootsect.s 编译后的结果
* 第 2 到 5 的四个扇区必须放置 setup.s 编译后的结果
* 第 6 个扇区开始放置编译后的 system 模块。

![](https://img-blog.csdnimg.cn/20210121000248960.png)






Makefile

* Makefile 就是控制编译来形成一个满足特定格式的 image 文件

* image 是由 bootsect，setup 和 system 三个顺序合并形成的；bootsect 是由 bootsect.s 汇编产生的；setup是由 setup.s 汇编产生的；system 是由进程模块、内存模块、设备驱动、初始化模块等部分组成的；而进程模块又是由 ······ 显然这是一个树状依赖结构，而Makefile 就是用来定义这个树状结构的。
* Makefile 文件的基本格式（树状结构）：
  * 目标：该目标依赖的其他目标
  * 产生该目标要执行的命令





## 2、setup.s

* 为系统初始化做准备，即执行setup，因此现在要转向去执行 setup.s
* 总结起来，setup 主要做两件事：准备初始化参数、进入保护模式



保护模式说明

* 从 0x100000（1M）地址处开始的内存被称为扩展内存，因为 16 位机器用“16 位段寄存 ≪4+16  位偏移”最多只能形成一个 20 位地址放到地址总线上，所以最多只能寻址 1M 以内的内存。
* 我们的机器内存显然要比 1M 大得多，所以启动保护模式以后，应该允许访问 1M 以后的扩展内存。另外，在操作系统启动以后，上层应用使用的就是扩展内存，因为1M 以下的内存都给操作系统了
  * 地址从 0x0 到 0x90000 处的内存用来存放 system 模块，
  * 而地址从 0x90000到 0x100000 处的内存用来存放那些重要的参数，比如此处获得的“扩展内存大小”这一参数，因此地址 1M 以后是扩展内存是将来的应用程序可以使用的



IBM PC 32 位模式寻址（ CS 不变，IP 变为 32 位的 EIP）

* 用段寄存器作为索引在一个地址表里找到 32 位的基地址，再和偏移寄存器中存放的 32 位数字加在一起，形成最终的地址放到地址总线上去选定内存，

* 显然，这个寻址过程中有两样东西我们现在还没有：一个就是那个地址表，通常这个表被称为 GDT（Global Description Table，全局描述符表）表。另一个是让计算机硬件找到这个表，因为整个寻址过程是硬件自动完成的（计算机体系机构中叫作虚拟存储器）。

* 对于 GDT 表，这个表的起始地址会被存放到一个被称为 GDTR 的寄存器中。一旦有了 GDT 和 GDTR，设计一个硬件电路根据 CS 来获取基地址然后再和 EIP 内容相加就很容易了。

  * GDT 表是 GDT 数组，其每项结构如下

    ![](https://img-blog.csdnimg.cn/20210121000058285.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)


  * CS（选择子） 是 GDT 或 LDT 表的索引，其高 13 位为索引，第 14 位为判断是 LDT 还是 GDT ，最后两位为权限

    ![](https://img-blog.csdnimg.cn/20210121000134620.png)


  * GDTR 是 GDT 表入口地址存放的地方，实际上是个地址寄存器，通过 GDTR 就可以找到 GDT 表，然后用 CS 的高 13 位就可以在 GDT 表找到对用的描述项

* setup.s 需要在实模式下先完成这个 GDT 表的形成以及初始化（按照 GDT 表的格式手动初始化一段内存内容即可）

  ```assembly
  ; SETUPSEG = 0x9020
  mov ax,#SETUPSEG
  mov ds, ax
  
  ; 指令 “lgdt gdtaddr”用从 DS:gdtaddr 内存地址取出来的内容赋给寄存器 GDTR
  ; lgdt 指令需要取出 6 个字节的数据，其中前 2 个字节是 GDT 表的表长上界，后 4 个字节是 GDT 表的基地址。
  lgdt gdtaddr
  
  ; 初始化了一个具有三个表项的 GDT 表
  ; 表项 0 没有用
  ; 表项 1 用来表示操作系统内核代码段
  ; 表项 2 表示操作系统内核数据段
  gdt:
  .word 0, 0, 0, 0
  .word 0x07FF, 0x0000, 0x9A00, 0x00C0
  .word 0x07FF, 0x0000, 0x9200, 0x00C0
  
  ; 用下面取出的 GDT 表基地址就是 0x90000+512+gdt，其中 512 表示跳过一个扇区，那是什么？bootsect。
  ; 0x90000+512 正好就是 setup 模块的开始地址，标号 gdt 是相对于 setup 开始地址的偏移，
  ; 因此 0x90000+512+gdt 恰好就是数据段起始地址 9020，然后就把定义的 gdt 放到了 gdtr 寄存器
  gdtaddr:
  .word 0x800
  .word 512+gdt, 0x9
  ```

* setup.s 还会在实模式下将整个 system 模块拖动到 0x0 地址处

  * 更具体的说，就是将内存地址 0x10000 开始到 0x90000 的全部内存内容移动到地址 0x00000 到 0x80000处（这个指的实际的物理地址，会覆盖掉 BIOS 的中断向量的地址，之后用 IDT 代替）







首先，setup 要获取一些硬件参数（通过 BIOS 的中断）

* 获得内存大小（int 0x15）

* 获得磁盘信息

  * 用 0x41 号 BIOS 中断可以取出硬盘信息

  * 特殊中断入口说明：BIOS 的 0x41 中断和别的 BIOS 中断存在一定的区别。

    * 别的中断，比如 0x10 号中断，在执行“int 0x10”指令时，会根据中断号到中断向量表中的特定位置取出中断处理地址的入口地址，对于 0x10 号中断，这入口地址就存放在 0+4×0x10内存地址处，因为 BIOS 中断向量表存放在内存 0 地址处，而每个中断入口的处理程序地址对应 4 个字节。
    * 但中断向量表 0x41 表项处存放却不是中断处理程序的入口地址，而就是第一个硬盘的基本参数（共 16 个字节），其中最主要的三个参数及其偏移位置如表所示。

    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210121000155747.png)

  ```assembly
  INITSEG = 0x9000
  mov ax, #INITSEG
  mov ds,ax
  
  ; 以 0x88 作为功能号调用 0x15 号 BIOS 中断就能获得扩展内存的大小，单位是 KB，返回值要放到寄存器 AX 中。
  mov ah,#0x88
  int 0x15
  ; “mov [0], ax” 将内存尺寸存放在地址 0x90000 处，将来系统初始化时可以读取这个值来初始化内存管理。
  mov [0],ax
  
  mov ax,#0x0000
  mov ds,ax
  
  ; 获取硬盘信息的核心是将这些重要信息（共 16 个字节）从地址 4×0x41处拷贝到内存地址 0x90080 处（这个地址可以自己指定）。
  ; 代码“rep”，“movsb”可以从 DS:SI 内存地址连续拷贝 CX 个字节到内存地址 ES:DI 处。
  lds si, [4*0x41]
  mov ax, #INITSEG
  mov es,ax
  mov di,#0x0080
  mov cx,#0x10
  rep
  movsb
  ```

* 还要获得很多其它硬件信息，比如显示器信息等，这些信息的获取方式是完全类似的。



硬件参数获得以后，setup 的下一项核心工作是要启动保护模式。

* 核心工作应该是让程序代码可以寻址到 32 位地址空间。

* 启动了 32 位寻址方式，现在机器要启动另外一套电路来解释执行指令。具体来说，CS:IP 不应该再按照以前的 CS≪4+IP 方式来工作了。

  ```assembly
  ; 为了启动这个新电路，setup.s 需要将 A20 号地址线选通，
  mov al,#0xD1
  out #0x64,al
  mov al, #0xDF
  out #0x60,al //选通 A20 地址线
  
  ; 并且将寄存器 CR0 （这是一个非常重要的控制寄存器，硬件的很多重大控制都是通过设置 CR0 来完成的）的最后一位设置为 1。
  ; 一旦完成这两项设置以后，内存寻址方式会采用另外一套电路 保护模式的电路。
  mov ax,#0x0001
  mov cr0,ax
  
  ; 跳到 CS = 8 , EIP = 0
  ; CS = 8 = 1 0 00，即 索引 = 1（即第二项）、tl = 0（代表选的 GDT表）、权限 = 00
  ; 由下图分析可知，最后得到的 32 位的段基址是 0000 0000，再加上 EIP 的 0，最后的物理地址就是 0000 0000
  ; 刚好对应上面说的，setup 把系统代码从 10000 移到了 0000 0000 的位置
  jmpi 0,8
  ```

  ![](https://img-blog.csdnimg.cn/202101210002205.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)



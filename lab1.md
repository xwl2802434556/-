## lab1 ##
### Part 1: PC Bootstrap ###
- 熟悉x86汇编语言和PC引导过程，以及QEMU/GDB
- Exercise1:[https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf](https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf)
- Simulating the x86:
	- make:生成obj目录，里面包含boot和kern
	- make qemu[-nox]:启动模拟器
	- 退出：按住ctrl和a，然后同时松开，再按x
- The PC's Physical Address Space
|Physical Address Space||
|----------------------|--|
|32-bit memory mapped|0xFFFFFFFF(4GB)|
|Unused||
|Extended Memory|depends on amount of RAM|
|BIOS ROM**|0x00100000(1MB)|
|16-bit devices use|0x000F0000(960KB)|
|VGA Display||
|Low Memory|0x00000000,早期用作RAM使用|
	- BIOS：系统初始化并检查内存，加载操作系统，将控制权交给操作系统。
- The ROM BIOS
	- 打开两个terminal，都进入lab，一个输入make qemu-nox-gdb,另一个输入make gdb.
	- [f000:fff0]    0xffff0: ljmp   $0xf000,$0xe05b
		- PC从0xffff0开始运行，即BIOS所在地址，初始时CS=0xf000,IP=0xfff0
		- 第一条指令是跳转指令，设置CS=0xf000,IP=0xe05b
	- Exercise2:使用si观察开始的几条指令，猜测bios在干嘛
		- 启动一个中断表，初始化各种设备，比如VGA，PCI总线
		- 然后找到一个可以启动的硬件，加载并将控制权转移
### Part2: The Boot Loader ###
- BIOS找到可启动的硬件(boot sector 引导区)
	- 初始化硬件
	- 加载引导区所在的512字节到物理内存0x7c00-0x7dff
	- 设置跳转指令，将CS:IP设置为0000:7c00，传递控制权
	- 现在可以加载2048bytes，功能更加强大
- boot loader包括两个文件：boot/boot.S,boot/main.c
	- 主要干两件事情：
	- boot.S将处理器从实模式转换为保护模式，这样才能访问超过1MB的物理空间，然后转入main.c
	- main.c从硬盘上根据特殊的IO指令，将kernal读入内存，并将控制权转到kernal
- 阅读obj/boot/boot.asm文件，知道每条指令的地址
- 常用debug命令：
	- b *0x7c00:设置断点，然后c，运行到断点处
	- si [N]:依次执行N条指令
	- x/Ni [Addr]:查看Addr开始的Ni条指令
- Exercise3：[https://pdos.csail.mit.edu/6.828/2018/labguide.html](https://pdos.csail.mit.edu/6.828/2018/labguide.html)
	1. 再0x7c00(引导区被加载地址)处设置断点，运行到0x7c00处，使用x/i观看汇编代码，与obj/boot/boot.asm对比
	2. 在boot.asm中找到bootmain地址，下个断点，再在readsect处下一个断点
	3. 在for循环后设一个断点
	4. 观看每个断点后的指令
	5. 问答
	    - At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?
		answer: ljmp    $PROT_MODE_CSEG, $protcseg
	    - What is the last instruction of the boot loader executed, and what is the first instruction of the kernel it just loaded?
	    answer：call *0x10018;   movw $0x1234,0x472
	    Where is the first instruction of the kernel?
		answer:movw $0x1234,0x472
	    How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?
		answer: ,ELFHDR->e_phnum
- Loading the Kernal
	- 理解指针：[https://pdos.csail.mit.edu/6.828/2018/labs/lab1/pointers.c](https://pdos.csail.mit.edu/6.828/2018/labs/lab1/pointers.c)
	- 理解ELF文件：[https://pdos.csail.mit.edu/6.828/2018/readings/elf.pdf](https://pdos.csail.mit.edu/6.828/2018/readings/elf.pdf)
	- ELF包含几个重要部分
		- .text：代码段
		- .rodata：只读段，例如C中字符串常量
		- .data：数据段，保存初始化的全局变量
		- .bss：数据段，保存未初始化的全局变量（只记录变量的地址和大小）
	- objdump -h obj/kern/kernel
		- 查看elf中各种节的大小，...
		- LMA:节所在的物理地址
		- VMA:节所在的虚拟地址
	- objdump -x obj/kern/kernel
		- LOAD表示需要加载到内存中的节
	- Exercise5:
		- 阅读boot.asm文件，找到第一条可能使程序崩溃的指令(如果修改了boot/Makefrag文件的话) answer:lgdt gdtdesc
		- lgdt gdtdesc会将虚拟地址映射到物理地址
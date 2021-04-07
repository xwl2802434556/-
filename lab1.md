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
	- objdump -f obj/kern/kernel
		- 查看程序入口点e_entry
	- main.c的作用
		- 将内核的每个节从磁盘读入内存，然后将CS:IP指向程序入口点
	- Exercise6:
		- 重新开始运行make qemu-gdb,在boot loader前使用x/8x 0x100000查看地址0x100000内容，然后在boot loader后查看地址0x100000内容，观察有啥不同
		- 在0x7c00处设置断点，使用x/8x 0x100000查看，发现这八个字节都是0
		- 在0x7d6b处设置断点，使用x/8x 0x100000查看，发现内容已经被修改
### Part3: The kernel ###		
- Using virtual memory to work around position dependence
	- kern/kernel.ld: 查看物理地址和虚拟地址
	- kern/entrypgdir.c: 初始化页目录和页表
	- kern/entry.S: 设置CR0_PG标志，一旦设置，那么之后的所有地址都是虚拟地址，虚拟地址需要转换为物理地址。
	- 物理地址的0x0-0x400000对应着虚拟地址0xf0000000-0xf0400000和0x0-0x400000
- Exercise7:
	- 使用gdb在`movl %eax, %cr0`处设置断点，观察地址0x00100000和0xf0100000,然后单步执行，观察两地址变化
	- 在obj/kern/kernel.asm中找到`movl %eax, %cr0`的地址为0xf0100025，但此时还没有建立虚拟地址与物理地址映射，所以`movl %eax, %cr0`的物理地址为0x00100025，在此处下断点
	- 可以发现执行`movl %eax, %cr0`前0xf0100000内容都为0，执行后0xf0100000和0x00100000内容一样
	- 映射建立后的第一条指令是什么？干嘛用的？
		- answer：`mov $0xf010002f, %eax   jmp *%eax`，用来将CS:IP设置到虚拟地址
- Formatted Printing to the console
	- 阅读文件kern/printf.c,kern/printfmt.c,kern/console.c，确定他们的关系
	- Exercise8:我们省略了一小段代码-使用“％o”形式的模式打印八进制数字所必需的代码。 查找并填写此代码片段。answer:在printfmt.c的207行，照着上面修改就行。
	- q1:解释printf.c,console.c之间的接口，console.c提供了什么功能，如何被printf.c使用？
		- a1:printf.c使用了console.c提供的void cputchar(int);函数用来向控制台输出一个字符
	- q2:解释console.c195行代码
		- a2:当显示字符超过CRT一屏显示的字符时，将屏幕上第二行到最后一行上移一行，并将最后一行设置为黑色空格，然后将光标移动到最后一行开始位置。
	- q3:看完lecture2后，单步跟踪下面代码，回答问题
		- a3:
	- q4:
		- a4:
- The Stack
	- Exercise9:确定内核初始化栈的代码位置，以及栈在内存的位置，内核如何为栈分配空间，在何处初始化栈指针指向？
		- 查看kernel.asm代码，在48-58行初始化栈，将栈放在0xf0110000中
	- 刚开始时，栈指针%esp指向栈的最底位置，栈中其他位置都是空闲的。入栈包含两个操作，先将%esp-4,然后将值放入%esp;出栈包含两个操作，先读出%esp中的值，然后将%esp+4。%esp在32位机中总是4字节对齐
	- 基指针%ebp，当进入一个C函数时，有两个操作：先push %ebp,然后mov %esp,%ebp。
	- Exercise10:为了熟悉C函数调用过程，在kernel.asm中找到test_backtrace处设置断点，观察每一步指向发生了什么。
	- Exercise11:实现backtrace功能
		```
		cprintf("Stack backtrace:\n");
		int* ebp = (int*)read_ebp();
		while(ebp){
			cprintf("  ebp %08x  ", ebp);
			cprintf("eip %08x  ", *(ebp + 1));
			cprintf("args %08x ", *(ebp + 2));
			cprintf("%08x ", *(ebp + 3));
			cprintf("%08x ", *(ebp + 4));
			cprintf("%08x ", *(ebp + 5));
			cprintf("%08x\n", *(ebp + 6));
			ebp = (int*)(*ebp);
		}
		```
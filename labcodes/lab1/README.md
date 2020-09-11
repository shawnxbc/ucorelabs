# Lab 0 & Lab 1

*由于这是第一个实验报告，所以我把Lab 0搭建实验环境的部分也放进来了。*

## Lab 0

### 搭建uCore的实验环境

为了方便， 我在Windows上安装了虚拟机，使用的发行版是Deepin20-Beta。主要原因是Deepin对4K屏幕的支持很好，桌面GUI这些都非常漂亮，而且就我的日常使用来说，并没有出现网上说的十分卡顿影响体验的情况。首先是安装相应的软件，这里先选择必要的那些安装包就行：

```
sudo apt update
sudo apt upgrade
sudo apt install build-essential git qemu-system-x86 gdb make diffutils gcc-multilib
```

要在Linux平台上进行编程，首先需要你熟悉命令行的基本操作和一些常用的工具，例如：Vim，Gcc，GDB，Git以及make/Makefile等等。如果你对这些东西不够了解，建议首先在网上查找一些快速入门的资料。注意，对于上面提到的这些工具以及其他后面可能用到的软件，没有必要上来就做很深入的学习，有基本了解后根据需要学习就行了。

在执行完上面的命令后，我们就已经安装好了qemu模拟器，我们可以查看一下安装结果：

```shell
$ ls /usr/bin/qemu*
/usr/bin/qemu-img  /usr/bin/qemu-nbd        /usr/bin/qemu-system-i386
/usr/bin/qemu-io   /usr/bin/qemu-pr-helper  /usr/bin/qemu-system-x86_64
```

其中`qemu-system-i386`是32位模拟器，`qemu-system-x86_64`是64位模拟器。在这门课的实验过程中，我们使用的是32位的`qemu-system-i386`模拟器，为了方便，我们创建`qemu-system-i386`的软链接：

```
$ cd /usr/bin
$ sudo ln -s qemu-system-i386 qemu
```

从GitHub获取ucore的实验基准源代码，如果因为网络问题导致速度很慢或者下载失败，可以挂下代理：

```
$ git clone https://github.com/LearningOS/ucore_os_lab.git
```

注意：为了后面提交代码的方便，你最好fork上述仓库，然后将链接替换成你自己的版本。

在下载后的"ucore_os_lab"文件夹下，有两个子文件夹："labcodes"和"labcodes_answer"。"labcodes"文件夹里是空的魔板，你需要做的就是填充代码，使之完成相应的功能；"labcodes_answer"文件夹则如其名所显示的那样，里面有参考代码。为了感受qemu模拟硬件的效果，可以尝试先进入lab1文件夹编译运行：

```
$ cd labcodes_answer/lab1_result/
$ make
$ make qemu
```

上述命令执行完毕后，就会出现一个模拟的机器窗口，不断地打印时钟频率。你也可以进入"labcodes"目录下的"lab1"子目录下，执行同样的步骤，但你会发现窗口一闪而过，因为里面还没有实际的代码。

注意，如果你在QEMU的界面上点击鼠标，QEMU会捕捉你的键鼠操作，按下 `Ctrl + Alt + g` 取消捕捉。

## Lab 1

### 练习 1：理解通过 make 生成执行文件的过程。

> 在此练习中，大家需要通过静态分析代码来了解：
>
> 1. 操作系统镜像文件 ucore.img 是如何一步一步生成的？(需要比较详细地解释 Makefile 中每一条相关命令和命令参数的含义，以及说明命令导致的结果)
> 2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

先给出如下的ucore.img整体的编译流程，然后我们开始具体分析。

![ucore编译流程](https://raw.githubusercontent.com/shawnxbc/PicGo/master/technotes/ucore%E7%BC%96%E8%AF%91%E6%B5%81%E7%A8%8B.png)

在完成这个练习的时候，我首先是非常naive地打开了lab1目录下的Makefile，然后一头扎进去开始阅读，结果就是这个两百多行的配置文件直接把我整蒙了，所以肯定得换种方式。我们先通过 `make -n > make_commands` 查看在编译过程中Make依次执行了哪些命令，从后往前反向跟踪会比较方便。打开后发现最后3条命令如下，清楚说明了ucore.img的生成过程：

```
// 这里使用dd命令来创建磁盘镜像ucore.img。dd默认的block size是512字节，下面的命令先写了10000个block的0到ucore.img，也就是5000KB。然后bootblock的大小也是512字节，它是BIOS默认会加载到0x7c00的MBR（主引导扇区）。notrunc选项表示不要截断输出文件，也就是只把bootblock写入到ucore.img的前512字节，而ucore.img剩下的部分保持不变。最后，跳过第一个block（seek=1），把kernel写到紧接在第一个block之后的位置，形成完全的启动磁盘。
dd if=/dev/zero of=bin/ucore.img count=10000
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```

由于ucore.img是通过bootblock和kernel这两个文件构造的，因此我们只需要弄清楚这两个文件是怎么生成的就行了。

下面是第一条的编译指令，其中各个选项的含义如下：

- `-Ikern/init/` 这条指令以及后面几条类似的 `-I*` 指令都表示编译的时候将指定加入 `#include` 文件的搜索路径。
- `-march=i686 ` 指定目标CPU架构，这里 `i686` 实际是"Pentium Pro"，是Intel推出的一款32位CPU。
- `-fno-builtin` 指明不把非 `__builtin_` 开头的函数识别为内置函数。
- `-fno-PIC` 不使用位置无关代码。这个选项是必要的，在piazza下的这个[讨论](https://piazza.com/class/i5j09fnsl7k5x0?cid=964)中，有同学编译bootblock时代码体积超过了限制，下面有人提出关闭PIC就能顺利编译，我猜测可能是为支持PIC而设置的诸如全局偏移量表(GOT)和过程链接表(PLT)这样的结构增加了代码体积。[fuckoff]
- `-Wall` 打开所有的警告信息。
- `-ggdb` 编译时保留相关信息用于调试。
- `-m32` 编译成32位的目标程序。
- `-gstabs` 指定调试信息的格式为stabs(symbol table strings)。
- `-nostdinc` 不在系统标准路径下搜索头文件，只使用 `-I` 指定的目录。
- `fno-stack-protector` 禁用栈保护，所谓栈保护是指通过在栈上插入变量和返回时检查变量来检测是否发生堆溢出。
- `-c kern/init/init.c -o obj/kern/init/init.o` 表示要编译的源文件是 `init.c`， 输出文件是 `init.o` 。

```
gcc -Ikern/init/ -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
```

由于其他各行使用了很多跟这里相同的编译选项，我们可以先将这部分选项删除，使make_commands文件看起来更加简洁。

#### bootblock的生成

最后生成bootblock的命令如下，可以看出它是由bootasm.o和bootmain.o这两个文件链接得到的。

```
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```

其中各个选项的含义如下：

- `-m elf_i386` 生成32位的ELF可执行文件。
- `-nostdlib` 链接时不使用标准库。
- `-N` 将代码段和数据段都设置为可读写的，这样在加载的时候会变得容易些，ELF文件里不会有额外的空白的零，减少文件的体积。[fuckoff]
- `-e start` 显式地指定start符号作为程序的入口。
- `-Ttext 0x7C00` 设置代码段的起始地址为0x7c00，这是BIOS会加载MBR到的内存地址。

Gcc编译链接得到的bootblock.o是ELF格式的目标文件，这种目标文件包含ELF文件头代码段数据段等部分，实际需要loader这样的额外支持才能加载到内存。由于我们是在开发操作系统，并没有这样的支持，因此需要的是真正的与内存镜像对应的二进制文件，我们使用如下的ojbcopy命令完成这个任务。其中，`-S` 表示"strip all"，含义是不复制任何重定位和符号信息；`-O binary` 表示输出为raw binary格式。如果你查看相关文件，会发现 `bootblock.o` 有5.53K，而原生二进制格式的 `bootblock.out` 只有492B。

```
objcopy -S -O binary obj/bootblock.o obj/bootblock.out
```

最后一步是把 `bootblock.out` 转换成完整的512B的MBR，这个转换是通过sign这个工具进行的：

```
objcopy -S -O binary obj/bootblock.o obj/bootblock.out
```

sign程序做的事情非常简单，除了一些常规的检查，比如看 `bootblock.out` 大小是否超过了510B以外，核心操作就是往MBR的最后两个字节写入了"0x55AA"。这个数字是一个disk signature，表明这是一个有效的MBR。

```c
    buf[510] = 0x55;
    buf[511] = 0xAA;
```

#### kernel的生成

生成kernel的命令涉及的目标文件比较多，但是其中的 `-m    elf_i386 -nostdlib` 这些选择都是之前提到过的。唯一需要注意的是，链接kernel的时候通过 `-T tools/kernel.ld` 选项指定了自定义链接脚本。当我们正常使用Gcc 的时候，最后一步使用的链接器其实也是 `ld` ，只不过执行的是默认的链接脚本，生成的可执行文件是为了在当前的操作系统环境中执行。比如，你可以使用  `ld --verbose`  查看默认链接脚本。然而，kernel是跑在bare metal上的，所以需要使用定制化的链接脚本。

```
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/stdio.o obj/kern/libs/readline.o obj/kern/debug/panic.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/picirq.o obj/kern/driver/intr.o obj/kern/trap/trap.o obj/kern/trap/vectors.o obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  obj/libs/string.o obj/libs/printfmt.o
```

链接脚本是由多个如下形式的命令组成的，最简单的链接脚本只需要一个命令 `SECTION` ，指明在输入的目标文件中抽取哪些节(section)，在输出的二进制文件中合成哪些节(section)。如果你打开上面链接的自定义脚本文件 `tools/kernel.ld` ，就会发现它就是这样格式的脚本，所做的事情也是抽取合并。

```
COMMAND {
	sub-command 1
	sub-command 2
	.... more sub-command....
}
```

下面是一段链接脚本的例子：

```
SECTIONS {
	. = 0x10000; /* sub-command 1：将后续命令的基地址设为0x10000 */
	.text : { *(.text) } /* sub-command 2：将所有目标文件中的.text节合并到最后二进制文件中的.text节 */
	. = 0x8000000; /* sub-command 3 ：将后续命令的基地址设为0x8000000*/
	.data : { *(.data) } /* sub-command 4：将所有目标文件中的.data节合并到最后二进制文件中的.data节 */
	.bss : { *(.bss) } /* sub-command 5：将所有目标文件中的.bss节合并到最后二进制文件中的.bss节 */
}
```

### 练习 2：使用 qemu 执行并调试 lab1 中的软件。

> 为了熟悉使用 qemu 和 gdb 进行的调试工作，我们进行如下的小练习：
>
> 1. 从 CPU 加电后执行的第一条指令开始，单步跟踪 BIOS 的执行。
> 2. 在初始化位置 0x7c00 设置实地址断点,测试断点正常。
> 3. 从 0x7c00 开始跟踪代码运行,将单步跟踪反汇编得到的代码与 bootasm.S 和 bootblock.asm 进行比较。
> 4. 自己找一个 bootloader 或内核中的代码位置，设置断点并进行测试。

注意：由于Makefile里指定的终端是gnome-terminal，如果你的操作系统默认终端不是这个，那你要么更改Makefile里的终端选择，或者安装gnome-terminal。安装gnome-terminal的命令如下：`sudo apt install gnome-terminal dbus-x11` 。

默认的gdbinit文件中将断点设在了 `kern_init` 的入口处，要从CPU加电后执行的第一条指令开始，我们需要先修改gdbinit文件：

```
file bin/kernel
set arch i8086
target remote: 1234
```

然后执行 `make debug` 就能进入调试界面。进入调试界面后，使用 `info registers reg` 查看相应寄存器的值：

```
(gdb) info registers cs
cs             0xf000              61440
(gdb) info registers eip
eip            0xfff0              0xfff0
```

使用以下命令查看要执行的第一条指令的内容，发现其确实是一条长跳转指令：

```
(gdb) x/i $cs*16+$eip
   0xffff0:     ljmp   $0x3630,$0xf000e05b
```

但是麻烦的地方在于，尽管我设置了 `set architecture i8086`，如果直接使用 `layout asm` 会发现显示的汇编指令不对。查了一下，主要原因是GDB对8086的支持有问题，它没有按照段加偏移的方式去寻找指令，而是直接到 `0xfff0` 的位置去找的指令。因此只有当段寄存器的内容为0时，GDB才能找到正确的指令地址。要解决这个问题，要么参照[这个回答](https://stackoverflow.com/questions/62513643/qemu-gdb-does-not-show-instructions-of-firmware)做一些额外的设置，要么换BOCHs这种对此有支持的模拟器。

使用 `b *0x7c00` 在设置断点，然后继续执行。当程序在断点暂停的时候，使用 `layout asm` 发现相应的汇编指令跟 `bootasm.S` 和 `bootblock.asm` 一致：
![Screen Capture_gnome-terminal-server_20200903095706](https://raw.githubusercontent.com/shawnxbc/PicGo/master/technotes/Screen%20Capture_gnome-terminal-server_20200903095706.png)

### 练习 3：分析 bootloader 进入保护模式的过程。

从实模式到保护模式的转换要做三件事情：开启A20地址线，突破物理内存寻址空间不能超过1MB的限制；在内存中准备好全局描述符表(GDT)并加载；将 `cr0` 寄存器的 `PE` 位置1实现真正从实模式到保护模式的切换。下面我们将更细致地讲解这个过程。

在 bootloader 接手 BIOS 的工作后，当前的 PC 系统处于实模式（16 位模式）运行状态，在这种状态下软件可访问的物理内存空间不能超过 1MB，且无法发挥 Intel 80386 以上级别的 32 位 CPU 的 4GB 内存管理能力，我们通过修改 A20 地址线可以完成从实模式到保护模式的转换。由于当时Intel的8042键盘控制器本身有一个未使用的引脚，所以他们决定用这个引脚来控制A20地址线，所以我们通过对8042键盘控制器的读写来开启A20地址线，整个过程如下：

```
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port
    
seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```

尽管Intel处理器的分段机制是硬件架构强制要求的，我们无法关闭，但我们可以通过对全局描述符表(GDT)中段描述符的设置实现一个平坦的内存模型(basic flat model)，相当于忽略掉分段机制对我们影响。在平坦内存模型下，我们实际将代码段和数据段定义为重叠的内存区间，每个段都覆盖整个4GB的可寻址内存，当然这也意味着这两个段之间不是相互保护各自隔离的。除此以外，CPU还要求GDT的首元素是空描述符(null descriptor)，这是为了能够方便检测到诸如未设置段寄存器情况下就进行内存访问这样的错误。

然后，下面的代码定义了全局描述符表和GDTR寄存器的内容，并通过 `lgdt gdtdesc` 将相应内容从内存加载到GDTR寄存器：

```assembly
# Bootstrap GDT
.p2align 2                                          # force 4 byte alignment
gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel

gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt
```

定义在内存中的全局描述符表其实就是之前谈到的3个表项，包括null descriptor、代码段描述符和数据段描述符。ucore源码里这部分的定义代码比较简洁，但是不太好理解，这里给个繁琐一点但比较清楚的重写版：

```assembly
; GDT
gdt_start:

gdt_null:	; the mandatory null descriptor
	dd 0x0	; ’dd ’ means define double word ( i.e. 4 bytes )
	dd 0x0
	
gdt_code: 	; the code segment descriptor
	; base =0x0 , limit =0 xfffff ,
	; 1st flags : ( present )1 ( privilege )00 ( descriptor type )1 -> 1001 b
	; type flags : ( code )1 ( conforming )0 ( readable )1 ( accessed )0 -> 1010 b
	; 2nd flags : ( granularity )1 (32 - bit default )1 (64 - bit seg )0 ( AVL )0 -> 1100 b
	dw 0xffff	 ; Limit ( bits 0 -15)
	dw 0x0 		 ; Base ( bits 0 -15)
	db 0x0 		 ; Base ( bits 16 -23)
	db 10011010b ; 1st flags , type flags
	db 11001111b ; 2nd flags , Limit ( bits 16 -19)
	db 0x0 		 ; Base ( bits 24 -31)
	
gdt_data: ; the data segment descriptor
	; Same as code segment except for the type flags :
	; type flags : ( code )0 ( expand down )0 ( writable )1 ( accessed )0 -> 0010 b
	dw 0xffff 	  ; Limit ( bits 0 -15)
	dw 0x0 	 	  ; Base ( bits 0 -15)
	db 0x0 	 	  ; Base ( bits 16 -23)
	db 10010010 b ; 1st flags , type flags
	db 11001111 b ; 2nd flags , Limit ( bits 16 -19)
	db 0x0 		  ; Base ( bits 24 -31)
	
gdt_end: ; The reason for putting a label at the end of the
		 ; GDT is so we can have the assembler calculate
		 ; the size of the GDT for the GDT decriptor ( below )
		 
; GDT descriptior
gdt_descriptor:
	dw gdt_end - gdt_start - 1 ; Size of our GDT , always less one of the true size
	dd gdt_start ; Start address of our GDT
	
; Define some handy constants for the GDT segment descriptor offsets , which
; are what segment registers must contain when in protected mode. For example ,
; when we set DS = 0 x10 in PM , the CPU knows that we mean it to use the
; segment described at offset 0 x10 ( i.e. 16 bytes ) in our GDT , which in our
; case is the DATA segment (0 x0 -> NULL ; 0x08 -> CODE ; 0 x10 -> DATA )
CODE_SEG equ gdt_code - gdt_start
DATA_SEG equ gdt_data - gdt_start
```

第三步是使能控制寄存器 `cr0` 的PE位。为了避免更改 `cr0` 中其他标志位的状态，我们使用或操作完成这个过程：

```assembly
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
```

接着还有一个长跳转，这个长跳转看起来有点奇怪，因为跳转的目标地址其实就在该指令的后面。这条指令其实非常重要，因为现代CPU为了提高指令的执行速度，采用了流水线之类的优化策略，这种策略会使得多条指令同时执行。而考虑到我们正在做的从实地址模式到保护模式的切换，就可能出现同一条指令执行过程的不同阶段处于不同的模式中。类似长跳转或者函数调用之类的指令，会迫使CPU结束当前流水线中的任务，避免上述问题。另外，这个长跳转还会使用指令中指定的段地址更新 `cs` 寄存器的值，这里就是 `PROT_MODE_CSEG`。在 `bootasm.S` 的开头，我们可以看到 `PROT_MODE_CSEG` 的定义，这个偏移量的计算很简单：每个描述符的长度是8个字节，代码段描述符是第二个，那偏移量就是 `0x8` ，数据段描述符是第三个，那偏移量就是 `0x10` 。

```assembly
    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp $PROT_MODE_CSEG, $protcseg
```

```assembly
.set PROT_MODE_CSEG,        0x8                     # kernel code segment selector
.set PROT_MODE_DSEG,        0x10                    # kernel data segment selector
```

最后是对数据段寄存器和其他段寄存器的初始化，然后就跳转到 `bootmain` 开始加载kernel：

```assembly
.code32                                             # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
    call bootmain

    # If bootmain returns (it shouldn't), loop.
spin:
    jmp spin
```

### 练习 4：分析 bootloader 加载 ELF 格式的 OS 的过程。

> bootloader 如何读取硬盘扇区的？

读取硬盘扇区的代码放在 `readsect` 函数中，整个过程包括：等待磁盘准备完成，依次向0x1f0-0x1f7地址写入参数和读取命令，等待磁盘完成数据准备，使用 `insl` 命令读取数据到目标地址。在 `bootmain.c` 中还有一个对 `readsect` 进行封装的函数 `readseg(va, count, offset)` ，该函数相对于kernel偏移offset的地方开始，读取count个字节，到va这个地址。`readseg` 的功能明显更加易用，在 `bootmain` 中实际使用的也是该函数。

> bootloader 是如何加载 ELF 格式的 OS？

首先检查这是否是个合格的ELF文件，方式是看kernel最开始几个字节是否是 `"0x7FELF"` ，这跟之前使用 `0x55aa` 检查磁盘的第一个扇区是否是合格的主引导扇区是一样的。然后通过ELF头找到对应的program header，program header里定义了可执行文件中连续的片跟内存段之间的映射关系。然后就是根据这种映射关系，使用for循环，将各个段从可执行文件中的偏移位置读取到内存的特定位置。到这kernel就被加载到内存中了，最后跳转到ELF头中指定的入口即可。kernel的ELF头中的入口地址是通过kernel.ld这个自定义链接脚本指定的，就是 `kern_init ` 函数的入口地址。

```c
    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
```

### 练习 5：实现函数调用堆栈跟踪函数 

由于模块中已经提供了 `read_ebp` 、`read_eip` 和 `print_debuginfo` 这样的辅助函数，所以只要理解函数过程调用过程中栈的结构就能很容易地完成补全，而栈的结构已经在“函数堆栈”材料中给出。整体逻辑是：先获取 `ebp` 和 `eip` 的值，打印当前函数的相关信息；然后将 `eip` 的值修改为调用函数的返回值，将 `ebp` 修改为已入栈的上个 `ebp` 的值，开始下一轮循环。退出循环的条件是 `ebp == 0x0` ，因为这是最开始 `bootasm.S` 中设置的值，也就意味着函数栈到顶了。

```c
void print_stackframe(void) {
      uint32_t ebp = read_ebp();
      uint32_t eip = read_eip();
      while (ebp != 0x0) {
          uint32_t *args = (uint32_t *)ebp + 2;
          cprintf("ebp:0x%08x eip:0x%08x args:0x%08x 0x%08x 0x%08x 0x%08x\n", 
                    ebp, eip, args[0], args[1], args[2], args[3]);
          print_debuginfo(eip - 1);
          eip = ((uint32_t *)(ebp))[1];
          ebp = ((uint32_t *)(ebp))[0];
      }
}
```

测试结果如下：

```c
ebp:0x00007b28 eip:0x00100a50 args:0x000100b4 0x000100b4 0x00007b58 0x0010008e
    kern/debug/kdebug.c:306: print_stackframe+22
ebp:0x00007b38 eip:0x00100d15 args:0x00000000 0x00000000 0x00000000 0x00007ba8
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b58 eip:0x0010008e args:0x00000000 0x00007b80 0xffff0000 0x00007b84
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b78 eip:0x001000b8 args:0x00000000 0xffff0000 0x00007ba4 0x00000029
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b98 eip:0x001000d7 args:0x00000000 0x00100000 0xffff0000 0x0000001d
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007bb8 eip:0x001000fd args:0x0010329c 0x00103280 0x0000130a 0x00000000
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007be8 eip:0x00100051 args:0x00000000 0x00000000 0x00000000 0x00007c4f
    kern/init/init.c:28: kern_init+80
ebp:0x00007bf8 eip:0x00007d72 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007d71 --
++ setup timer interrupts
```

最后一行的内容摘录如下。这行内容是对 `bootmain.c` 里的 `bootmain` 函数的描述，`ebp:0x00007bf8` 表示该函数的栈基址是 `0x00007bf8` ，跟 `0x7c00` 相差8个字节，刚好等于调用 `bootmain` 时入栈的返回地址和原 `ebp` ；`eip:0x00007d72` 表示 `bootmain.S` 中 `call bootmain` 指令的下一条指令的地址，也就是 `spin` 标签的地址。

```c
ebp:0x00007bf8 eip:0x00007d72 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007d71 --
```

### 练习 6：完善中断初始化和处理 

> 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

IDT中的每个表项跟GDT一样，都是占8个字节。其中，最高位的两个字节和最低位的两个字节共同定义了偏移量，第3-4字节定义了段选择子。在确定中断处理代码的入口时，先通过段选择子到GDT中查找到相应的段基址，加上中断描述符中的偏移量获得最后的执行入口。

> 请编程完善 kern/trap/trap.c 中对中断向量表进行初始化的函数 idt_init。在 idt_init 函数中，依次对所有中断入口进行初始化。使用 mmu.h 中的 SETGATE 宏，填充 idt 数组内容。每个中断的入口由 tools/vectors.c 生成，使用 trap.c 中声明的 vectors 数组即可。

由于各个中断已经在vector.S中被定义好了，并且这些中断的入口都已经放在了 `__vectors` 这个数组中，我们只需要用这个数组来初始化中断描述符表就行了。我们通过 `for` 循环依次对中断描述符表 `idt[256]` 的每个条目调用 `SETGATE` 宏进行初始化，该宏的参数列表为 `SETGATE(gate, istrap, sel, off, dpl)` ，相应参数已经在 `memlayout.h` 中定义，直接使用即可：

```c
void idt_init(void) {
      extern uintptr_t __vectors[];
      for (int i = 0; i < 256; i++) {
          SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
      }
      // 这里对系统调用做特殊处理：系统调用的编号0x80已经在trap.h中定义为T_SYSCALL，我们把系统调用的istrap设为1，
      // 把描述符权限级别设置为3，因为系统调用是向普通用户进程暴露的服务接口，权限应该设置为用户级别。
      SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
      lidt(&idt_pd);
}
```

> 请编程完善 trap.c 中的中断处理函数 trap，在对时钟中断进行处理的部分填写 trap 函数中处理时钟中断的部分，使操作系统每遇到 100 次时钟中断后，调用 print_ticks 子程序，向屏幕上打印一行文字”100 ticks”。

由于kern/driver/clock.c中已经提供了用于记录时钟中断次数的 `ticks` 全局变量，使用该变量就能很容易地实现目标功能：

```c
    case IRQ_OFFSET + IRQ_TIMER:
        ticks++;
        if ((ticks % TICK_NUM) == 0)
            print_ticks();
        break;
```

### 扩展练习 Challenge 1

注：有关中断描述符的结构、中断处理时硬件的入栈操作和中断类型的整体描述可查看[这里](https://pdos.csail.mit.edu/6.828/2018/lec/x86_idt.pdf)。

这里首先有必要对中断发生时特权切换的机制进行解释。CPU在拿到中断号时，会先去中断描述符表里查找对应的表项，检查CPL是否小于等于该中断描述符的DPL。如果满足条件，CPU会通过该中断描述符里的段选择子去GDT里查找对应的表项，如果CPL小于段描述符的DPL，则确认发生了特权切换，当前特权级被提升，以使得CPL等于段描述符的DPL。这时 CPU 会从当前程序的 TSS 信息（该信息在内存中的起始地址存在 TR 寄存器中）里取得该程序的内核栈地址，即包括内核态的 ss 和 esp 的值，并立即将系统当前使用的栈切换成新的内核栈，这个栈就是即将运行的中断服务程序要使用的栈。然后CPU需要先把当前程序使用的用户态的 ss 和 esp 压到新的内核栈中，然后依次压入当前被打断程序使用的 eflags，cs，eip，errorCode（如果是有错误码的异常）信息。注意，上述过程由硬件完成。

后面就是跳转到中断处理程序的入口：1. 对不带有错误码的中断，也要压入错误代码的占位符，以统一处理的模式；2. 压入当前中断的中断号；3. 所有中断在完成上述两步后都跳转到 `__alltraps` 开始执行，该例程先将 ds, es, fs, gs 以及其他通用寄存器压栈，然后设置好内核用的数据段，(内核用的代码段在查找中断入口和跳转到该入口的时候就完成了），最后把当前的 esp 也压入栈将其作为实参传递给 `trap`，最后调用 `trap`。`trap` 直接调用 `trap_dispatch` ，通过 `switch` 语句根据中断号进行中断处理。处理完毕后，返回 `__alltraps` 例程，该例程弹出之前作为实参入栈的 esp，然后进入 `__trapret` 例程，开始中断的扫尾工作，整体就是步骤1 和 `__alltraps` 开始部分的反过程：弹出所有通用寄存器和 gs, fs, es, ds，弹出中断号和错误代码，最后执行中断返回的 `iret` 指令。`iret` 指令做的工作，是中断开始到步骤1之前由硬件完成的工作的反过程：首先从内核栈里弹出先前保存的被打断的程序的现场信息，即 eflags，cs，eip 重新开始执行，这里当恢复 cs 寄存器时实际就会恢复到原来的特权级；如果存在特权级转换（从内核态转换到用户态），则还需要从内核栈中弹出用户态栈的 ss 和 esp，这样也意味着栈也被切换回原先使用的用户态的栈了。

由于之前在初始化IDT的时候，除了 `idt[T_SYSCALL]` 对应的 `DPL` 设置为了 `DPL_USER` 以外，其他中断描述符默认的DPL都是 `DPL_KERNEL` ，所以首先需要把 `idt[T_SWITCH_TOK]` 的 `DPL` 也设置为 `DPL_USER` 才能在用户态的时候调用 `idt[T_SWITCH_TOK]` 对应的中断处理程序：

```c
SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
```

我一开始没整明白题目要求完成的用户态和内核态之间的切换说的什么，其实背后的逻辑是这样的。正常情况下，当我们在用户态调用某个中断后，只要该中断对应的中断描述符的DPL不小于CS中的CPL，计算机就会开始中断程序的执行。由于我们初始化IDT的时候，给所有中断描述符表项传入的段选择子都是 `KERNEL_CS` ，这意味着所有中断处理程序都会在内核态执行。当中断处理结束返回时，我们恢复原程序的现场并且切换回用户态。所以，`lab1_switch_to_user` 要实现向用户态的转换的话，我们可以先构造出特定的 `trapframe` ，该 `trapframe` 使得我们就像是从用户态发起的中断，然后中断返回时就能回到用户态。最开始本身处于内核态，进入中断时不会发生PL切换，CPU执行 `int` 指令时不会入栈 ss 和 esp，但离开中断时由于我们伪装成了是从用户态进入的中断，就会出现PL的切换，也就会导致 `iret` 会出栈 ss 和 esp，所以在 `lab1_switch_to_user` 里我们需要在 `int` 指令之前自己在栈中留下这两个元素的位置，退出中断后通过 `movl %ebp, %esp` 实现栈的平衡。相反地，从用户态切换成内核态时，由于发生了PL切换，所以CPU在执行 `int` 指令时会自己入栈 ss 和 esp，但离开中断时由于我们伪装成了是从内核态进入的中断，就不会出现PL的切换，也就会导致 `iret` 不会出栈 ss 和 esp，我们同样需要 `movl %ebp, %esp` 实现栈的平衡。下面是具体的代码实现：

```c
static void lab1_switch_to_user(void) {
    asm volatile (
        "sub $0x8, %%esp \n"
        "int %0 \n"
        "movl %%ebp, %%esp"
        :
        : "i"(T_SWITCH_TOU)
    );
}

static void lab1_switch_to_kernel(void) {
    asm volatile (
        "int %0 \n"
        "movl %%ebp, %%esp \n"
        :
        : "i"(T_SWITCH_TOK)
    );
}
```

```c
case T_SWITCH_TOU:
    if (tf->tf_cs != USER_CS) {
        tf->tf_cs = USER_CS;
        tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
        tf->tf_eflags |= FL_IOPL_MASK;
    }
    break;

case T_SWITCH_TOK:
    if (tf->tf_cs != KERNEL_CS) {
        tf->tf_cs = KERNEL_CS;
        tf->tf_ds = tf->tf_es = tf->tf_ss = KERNEL_DS;
        tf->tf_eflags &= ~FL_IOPL_MASK;
    }
    break;
```

当然，如果你整体审视上面的过程，实际可以发现我们只是非常纯粹地在形式上实现了特权级的切换。但上述实验确实比较详细了考察了我们对中断机制各种细节的理解，这应该也是这个Challenge想要实现的目标。

### 扩展练习 Challenge 2

这里我们按照题目的提示，把之前软中断处理的代码拿过来就行。为了方便，把相应的语句封装成了函数：

```c
    case IRQ_OFFSET + IRQ_KBD:
        c = cons_getc();
        switch (c) {
            case '0':
                cprintf("+++ switch to  kernel  mode +++\n");
                switch_to_kernel(tf);
                print_trapframe(tf);
                break;
            case '3':
                cprintf("+++ switch to user mode +++\n");
                switch_to_user(tf);
                print_trapframe(tf);
                break;
        }
        cprintf("kbd [%03d] %c\n", c, c);
        break;
```

对于上面的这种处理方式，我本来是有疑问的，因为这显然会导致之前在Challenge 1里提到过的栈平衡问题。但实际写出来测试的结果是成功的，这说明硬中断跟软中断实现机制上应该是有所差别的，不会导致我们担心的栈平衡的问题。另外，在上面完成Challenge 1后我们使用了 `make grade` 跑了一下测试脚本，需要重新编译一下项目，否则会由于调用 `panic` 进入 kernel debug monitor。
# 实现一个简单的操作系统

## 1. 搭建实验环境

### 工具安装

```
// Mac下安装
$ brew install x86_64-elf-gcc
$ brew install x86_64-elf-gdb
$ brew install cmake
brew install qemu


// Liunx下安装
$ sudo apt-get install gcc-i686-linux-gnu
$ sudo apt-get install gdb
$ sudo apt-get install cmake
$ sudo apt-get install qemu-system-x86
```

### 安装vscode插件

```
C/C++ Extension Pack  // C/C++扩展包
C/C++ // 必备
x86 and x86_64 Assembly // x86汇编语言支持
LinkerScript //提供链接脚本的语法高亮
Hex Editor // 十六进制编辑器
Makefile Tools // make文件工具
```

### 设置vscode任何文件都可以打断点

![](./images/breakpo.png)

## 2. 项目运行

```
原始文件

--- scripts // 代码启动调试脚本文件
	--- qemu-debug-liunx.sh liunx下的qemu启动脚本文件
	--- qemu-debug-osx.sh Mac下的qemu启动脚本文件
	--- qemu-debug-win.bat window下的qemu启动脚本文件
--- source  // 我们的代码
	--- start.S 启动第一扇区的代码
	--- os.c 剩余其他扇区的代码
	--- os.h 头文件
--- Makefile  // 编译文件
--- image  // 镜像文件
	--- disk.img 
```

### makeFile文件分析

```c
all: source/os.c source/os.h source/start.S
	$(TOOL_PREFIX)gcc $(CFLAGS) source/start.S // 编译start.S汇编文件，生成start.o文件
	$(TOOL_PREFIX)gcc $(CFLAGS) source/os.c	 // 编译os.cC语言文件,生成os.o文件
	$(TOOL_PREFIX)ld -m elf_i386 -Ttext=0x7c00 start.o os.o -o os.elf // 将生成的.o文件进行链接,生成os.elf文件,并指定生成的代码段的内存地址是0x7c00，运行的时候会放到0x7c00的内存地址上去
	${TOOL_PREFIX}objcopy -O binary os.elf os.bin // 复制成为os.bin文件
	${TOOL_PREFIX}objdump -x -d -S  os.elf > os_dis.txt	 // 头部文件内容输出
	${TOOL_PREFIX}readelf -a  os.elf > os_elf.txt	 // 头部文件内容输出
	dd if=os.bin of=../image/disk.img conv=notrunc // 将os.bin文件写入到disk.img中去，可理解为start.S中的代码会放入到镜像文件的第一个扇区中

```
 ***目的: 将代码段放置到0x7c00处，qemu启动的时候,BIOS会从这个内存的这个地方读取***

### 调试启动步骤

调试模式:
1. 首先 终端 -->  运行任务 : 启动我们的qemu任务
![](./images/%E8%BF%90%E8%A1%8C%E4%BB%BB%E5%8A%A1.png)
2. 运行 ---> 启动调试 : 启动我们的qemu调试工具
![](./images/%E5%90%AF%E5%8A%A8%E8%B0%83%E8%AF%95.png)

> Mac下稍微麻烦一点,要先启动qemu后再开启调试

全速飞奔模式:

去除scripts中的参数

```
# 适用于mac
# 将 -s  -S 参数去除即可
qemu-system-i386 -m 128M -s -S -drive file=disk.img,index=0,media=disk,format=raw 
```

## 3. 添加引导标志

前置: 
- 镜像文件中存放启动代码,第一个扇区 
- 代码启动时候从0x7c00处启动

在start.S中，我们使用了 .org 0x1fe的伪指令，通知gcc工具链，将0X55、0XAA这两个值放在相对于生成的二进制文件os.bin开头偏移510个字节的地方。然后os.bin会由dd命令写到磁盘映像文件的最开始处。通过这种方式就实现了在磁盘映像文件的第1个扇区最后两个字节添加引导标志的目的。

## 4. 设置寄存器的初始化值

```c
// 此时代码应该是从0x7c00处开始执行的,$_start代表着内存地址0x7c00
_start: 
	mov $0, %ax				// 设置代码段
	mov %ax, %ds			// 设置数据段
	mov %ax, %es			// 设置数据段
	mov %ax, %ss			// 设置栈段
	mov $_start, %esp		// 设置栈的起始地址

// 这里会将代码段寄存器、数据段寄存器、栈段寄存器设置为0,即便不设置，初始化的值也是0，栈顶是0x7c00往上，栈是从高地址向低地址增长
```
此时，计算机应该还是实模式，实模式下访问的是真实的内存地址,大概的访问范围是0 - 1M , 2的20次方 === 1M ,寻址范围是0x00000 - 0xFFFFF,为啥不是2的16次方呢？主要是最早的8086的寄存器都是16位的，后面内存扩充成1M，16位的寄存器无法访问到全部，就采用了段寄存器 << 4  + 偏移(16)位的组合来形成20位的地址,实现对1MB内存的访问

## 5. 加载自己的剩余部分

```c
read_self_all:
	mov $_start_32, %bx // 读取到的内存地址 0x7E00
	mov $0x2, %cx // ch:磁道号, cl:起始扇区
	mov $0x0240, %ax // ah:0x42读磁盘命令,al=0x40 64个扇区,多读一些,32kb
	mov %0x80, %dx // dh:磁头号 ,dl驱动器号0x80 磁盘1
	int $0x0013 // 调用中断,读取磁盘信息
	jc read_self_all // 读取失败则重复

磁盘第一个扇区     磁盘第2到64个扇区
 _start          _start_32
    ↓                 ↓
---------- | -----------------------|
  0x7c00   |       0x7E00           |
---------- | -----------------------|	

```
	 
## 6. 开启保护模式,从16位到32位


开启保护模式的主要作用还是初始化各个段寄存器的值(选择子)

在进行跳转_start_32之前,要开启保护模式，也就是在执行_start_32代码之前


1. 首先关闭中断

```c
cli // 关中断
```
2. 加载gdt表
```c
lgdt gdt_desc // 加载gdt表
// gdt描述符,由lgdt加载
gdt_desc: .word (256*8) - 1 // 边界
	.long gdt_table // 表的地址
// 表的内容
typedef unsigned char uint8_t;
typedef unsigned short uint16_t;
typedef unsigned int uint32_t;

struct {uint16_t limit_l, base_l, basehl_attr, base_limit;}gdt_table[256] __attribute__((aligned(8))) = {
    // 0x00cf9a000000ffff - 从0地址开始，P存在，DPL=0，Type=非系统段，32位代码段（非一致代码段），界限4G，
    [KERNEL_CODE_SEG / 8] = {0xffff, 0x0000, 0x9a00, 0x00cf},
    // 0x00cf93000000ffff - 从0地址开始，P存在，DPL=0，Type=非系统段，数据段，界限4G，可读写
    [KERNEL_DATA_SEG/ 8] = {0xffff, 0x0000, 0x9200, 0x00cf},
};

0 置空的表项
1 内核代码段  0 - 4G
2 内核数据段  0 - 4G
```

3. 设置PE位，进入保护模式

```c
mov $1,%eax
lmsw %ax // 设置PE位,进入保护模式
```

4. 进入32位指令下执行

```c

// 此时，我们已经设置好了代码段寄存器的值,就可以直接用jmp跳转过去执行了
jmp $KERNEL_CODE_SEG, $_start_32 // 段选择子，偏移量 0x7E00
// 32位保护模式,位于512字节后
	.code32
	.text
// 此时已经设置好了内核代码区和数据区的内存寻址范围,将覆盖原来的数据段、代码段寄存器的值	
_start_32:
	mov $KERNEL_DATA_SEG, %ax // 将数据段寄存器设置为16,偏移量0x0000
	mov %ax, %ds
	mov %ax, %es
	mov %ax, %ss
	mov $_start, %esp
	jmp .
```

## 7. 进入分页模式


回顾: 
1. 代码段寄存器1 ---> gdt表 1 : 0x7E00 (_start_32的代码位置), 内存寻址空间从0x00000000 - 0xFFFFFFFF
2. 数据段寄存器2 ---> gdt表 2 : 0x0000 (空),内存寻址空间是从0x00000000 - 0xFFFFFFFF

做完分段操作（实模式）之外，程序员实际操作的还是物理地址,开启分页后,经过分页转化后得到物理地址

分页的映射原理：

计算机中有一个硬件，叫MMU,内存管理单元，这个部件来负责将虚拟地址转化为物理地址


建立二级页表，然后将虚拟地址(线性地址，程序使用的地址)化分为3部分，高10位用于索引页目录表，中间10位用于索引第二级的页表，从中找到对应的物理页，然后再使用低12位访问页中的偏移

如下图所示:

![](./images/page.gif)

下面，我们要建立2个映射。第一个为页大小为4MB的映射，从虚拟地址0-4MB映射到物理地址0-4M，以便我们的程序仍然存在于它应该存在的地址，不至于程序跑飞。第二个为虚拟地址0x80000000到map_pyh_buffer的映射

> liunx0.1的代码中对于分页, 一个页目录下4个页表，一个页表1024个页,一个页4kb大小, 4 * 1024 * 4kb = 16Mb

```c
;内存开始 0x000 处设置页目录表
_pg_dir:

.org 0x1000 pg0: ;第1个页表在内存0x1000位置
.org 0x2000 pg1: ;第2个页表在内存0x2000位置
.org 0x3000 pg2: ;第3个页表在内存0x3000位置
.org 0x4000 pg3: ;第4个页表在内存0x4000位置

...

;将4个页目录项填写好
  mov dword [_pg_dir], pg0+7
  mov dword [_pg_dir+4], pg1+7
  mov dword [_pg_dir+8], pg2+7
  mov dword [_pg_dir+12], pg3+7
	
;设置4个页表中所有项的内容
  mov ecx, 1000
  mov edi, pg3+4092
  mov eax, 0xfff007
  mov dword [edi],eax
  cp:
    sub eax,0x1000
    sub edi,4
    mov dword [edi],eax
    cmp eax,0x007
    jne cp

;开启分页
;即设置 cr3 寄存器（页目录表基址寄存器）
   xor eax,eax
   mov cr3,eax
   mov eax,cr0
   or eax,0x80000000
   mov cr0,eax
```

1. 设置页目录PDE

```c
// 跳转到C语言中运行
call os_init

#define PDE_P			(1 << 0) // 存在位
#define PDE_W			(1 << 1) // 读写位
#define PDE_U			(1 << 2) // 权限位
#define PDE_PS			(1 << 7) // 声明4M对齐
// 设置目录表 4kb对齐
uint32_t pg_dir[1024] __attribute__((aligned(4096))) = {
    [0] = (0) | PDE_P | PDE_PS | PDE_W | PDE_U,	  // 开始位置0 - 4MB 映射到物理内存 0 - 4MB
};
// 这样，页目录表中第0项就存放着0-4MB的物理映射
```

2. 设置cr3寄存器

```c
mov $pg_dir, %eax // 将页目录的地址给到cr3寄存器
mov %eax, %cr3
```

3. 设置cr0寄存器

```c
mov %cr0, %eax
orl $(1 << 31), %eax // 将cr0的最高位设置为1,打开分页机制
mov %eax, %cr0
```

4. 设置cr4寄存器

```c
mov %cr4,%eax
orl $(1 << 4), %eax  // 允许使用4M到4M之间的映射,否则就要参考liunx0.1的代码进行4k的映射了
mov %eax, %cr4
```

5. 将0x80000000的虚拟地址映射到map_pyh_buffer

```c
// 定义一个数组
uint8_t map_phy_buffer[4096] __attribute__((aligned(4096)));

// 定义一下二级页表
static uint32_t pg_table[1024] __attribute__((aligned(4096))) = {PDE_U};    // 要给个值，否则其实始化值不确定

//  完成映射关系
void os_init (void) {
	// 0x80000000地址的高10位作为索引（左移22位）,它的值是二级页表的值
    pg_dir[MAP_ADDR >> 22] = (uint32_t)pg_table | PDE_P | PDE_W | PDE_U;
	// 0x8000000地址的中间10位作为索引(左移12位 & 0x3FF),它的值就是我要映射的数组的值
    pg_table[(MAP_ADDR >> 12) & 0x3FF] = (uint32_t)map_phy_buffer| PDE_P | PDE_W | PDE_U;
};
// 完成映射后，0x80000000的虚拟地址就对应着map_phy_buffer的物理地址了
```

## 8. 开启定时中断

在操作系统看来，中断分为软中断(系统调用/异常)和硬中断(IO设备)

中断的处理程序如下:

![](./images/%E4%B8%AD%E6%96%AD.jpeg)


首先配置8253芯片 --> 配置8259芯片 ---> 准备idt表 ----> 相应的表项设置段选择子(内核代码段) + 偏移量(中断处理函数的位置) ----> 中断发生 ---> 传递给CPU --> CPU在idt表中查询并跳转到中断处理函数 ---> 最终返回中断发生的地方

1. 8253芯片设置

```c
// offset_l 低16位偏移 selector 段选择子 attr 属性 offset_h 高16位偏移
struct {uint16_t offset_l, selector, attr, offset_h;} idt_table[256] __attribute__((aligned(8))) = {1};
// 在c语言中调用汇编指令
void outb(uint8_t data, uint16_t port) {
	// data要写的数据
	// port 端口
	// "d" "a" 代表寄存器al dx
	__asm__ __volatile__("outb %[v], %[p]" : : [p]"d" (port), [v]"a" (data));
}
// 在os_init中触发中断
// 初始化8259中断控制器，打开定时器中断
    outb(0x11, 0x20);       // 开始初始化主芯片
    outb(0x11, 0xA0);       // 初始化从芯片
    outb(0x20, 0x21);       // 写ICW2，告诉主芯片中断向量从0x20开始
    outb(0x28, 0xa1);       // 写ICW2，告诉从芯片中断向量从0x28开始
    outb((1 << 2), 0x21);   // 写ICW3，告诉主芯片IRQ2上连接有从芯片
    outb(2, 0xa1);          // 写ICW3，告诉从芯片连接g到主芯片的IRQ2上
    outb(0x1, 0x21);        // 写ICW4，告诉主芯片8086、普通EOI、非缓冲模式
    outb(0x1, 0xa1);        // 写ICW4，告诉主芯片8086、普通EOI、非缓冲模式
    outb(0xfe, 0x21);       // 开定时中断，其它屏蔽
    outb(0xff, 0xa1);       // 屏蔽所有中断

    // 设置8253定时器，每100ms中断一次
    int tmo = (1193180);      // 时钟频率为1193180
    outb(0x36, 0x43);               // 二进制计数、模式3、通道0
    outb((uint8_t)tmo, 0x40); // 定时器的内部端口0x40
    outb(tmo >> 8, 0x40); // 定时器的内部端口0x40

	// 添加中断 0x20是中断表中的定时中断
    idt_table[0x20].offset_h = (uint32_t)timer_init >> 16;
    idt_table[0x20].offset_l = (uint32_t)timer_init & 0xffff;
    idt_table[0x20].selector = KERNEL_CODE_SEG; // 选择子指向代码段
    idt_table[0x20].attr = 0x8E00;      // 存在，DPL=0, 中断门

```

2. idt_table加载到寄存器中去

```c
lidt idt_desc // 加载中断表
// idt描述符,由idt加载
idt_desc: .word (256*8) - 1
	.long idt_table
```

3. 添加中断处理函数

```c
// 中断函数
timer_init:
	push %ds // 数据段压栈  
	pusha // 保护现场,将寄存器全部压栈
	mov $0x20, %al
	outb %al, $0x20 // 发送EOI 
	popa // 恢复现场，将所有的寄存器全部出栈
	pop %ds 
	iret // 中断返回
```

4. 开启中断

```c
_start_32:
	... 
	sti  // 开中断
	jump . 
```


## 9. 切换低特权级

x86提供了特权级机制，用于将不同的代码划分成不同的特权级。其中操作系统运行于最高特权级0，可以执行很多系统指令，如开关中断等。而应用程序工作在最低特权级模式3，只能执行不需要特权的代码。

1.  首先在gdt表中添加两个进程相对应的代码段

```c
#define APP_CODE_SEG            ((3 * 8) | 3)       // 特权级3
#define APP_DATA_SEG            ((4 * 8) | 3)       // 特权级3

// 0x00cffa000000ffff - 从0地址开始，P存在，DPL=3，Type=非系统段，32位代码段，界限4G
[APP_CODE_SEG/ 8] = {0xffff, 0x0000, 0xfa00, 0x00cf},
// 0x00cff3000000ffff - 从0地址开始，P存在，DPL=3，Type=非系统段，数据段，界限4G，可读写
[APP_DATA_SEG/ 8] = {0xffff, 0x0000, 0xf300, 0x00cf},
```

2. 代码从特权级0（内核）切换到特权级3（应用）

```c
uint32_t task0_dpl3_stack[1024];
_start_32:
	... // 之前就是特权级0
	// 内核栈的栈顶是0x7c00处开始压栈ss:eip
	// 下面模拟中断返回,从而实现从特权级0到特权级3的变化
	// 中断发生时,会自动压入原SS, ESP, EFLAGS, CS, EIP到栈中
	push $APP_DATA_SEG
	push $task0_dpl3_stack + 1024	// 特权级3时的栈
	push $0			// 中断暂时关掉 0x202						// EFLAGS
	push $APP_CODE_SEG				// CPL=3时的选择子
	push $task_0_entry					// eip
	iret							// 从中断返回,将切换至任务0

task_0_entry:
	// 进入任务0时,需要重设其数据段寄存器为特权级3的
	mov %ss, %ax
	mov %ax, %ds
	mov %ax, %es
	jmp .	

```

> 现在的情况是暂时关闭了定时中断,在内核0特权级的时候调用了iret指令,进入了特权级3的应用代码task_0_entry

## 10. 两个任务的切换

一个任务的切换，要保存内核寄存器的状态，还需要一栈来保存局部变量和函数调用参数等，等任务结束后，进程的栈会被清除，从而也会将内核寄存器的值恢复

> 提一嘴，函数调用就是栈中的栈帧,每个进程的栈空间都是私有的

因为进程的切换必须在特权级0的环境下进行切换，所以，我们把任务的切换代码放在了定时器中断处理程序中

定时中断在发生的时候，就将内核的寄存器压入了栈中，等定时中断处理程序结束的时候，会出栈恢复内核寄存器的初始状态

1. 开启中断并在特权级3的情况下跳转task_0

```c
task_0_entry:
	jmp task_0	  // 跳转到任务0运行
_start_32:			
	push $0x202	 // 上个课程中没有开中断
```


2. 添加两个任务代码

```c
/**
 *  任务0
 */
void task_0 (void) {
    // 加上下面这句会跑飞
    // *(unsigned char *)MAP_ADDR = 0x1;

    uint8_t color = 0;
    for (;;) {
        color++;

        // CPL=3时，非特权级模式下，无法使用cli指令
        // __asm__ __volatile__("cli");
    }
}
/**
 *  任务1
 */
void task_1 (void) {
    uint8_t color = 0xff;
    for (;;) {
        color--;
    }
}

```

3. 配置两个进程的TSS结构

```c
// gdt表中添加进程的tss
#define TASK0_TSS_SEL           (5 * 8)
#define TASK1_TSS_SEL           (6 * 8)
// 暂时未给出起始地址
[TASK0_TSS_SEL/ 8] = {0x0068, 0, 0xe900, 0x0},
[TASK1_TSS_SEL/ 8] = {0x0068, 0, 0xe900, 0x0},
// 添加基地址
gdt_table[TASK0_TSS_SEL / 8].base_l = (uint16_t)(uint32_t)task0_tss;
gdt_table[TASK1_TSS_SEL / 8].base_l = (uint16_t)(uint32_t)task1_tss;

/**
 *  任务0和1的栈空间
 */
uint32_t task0_dpl0_stack[1024], task0_dpl3_stack[1024], task1_dpl0_stack[1024], task1_dpl3_stack[1024];

// 定义我们的tss结构
uint32_t task0_tss[] = {
    // prelink, esp0, ss0, esp1, ss1, esp2, ss2
    0,  (uint32_t)task0_dpl0_stack + 4*1024, KERNEL_DATA_SEG , /* 后边不用使用 */ 0x0, 0x0, 0x0, 0x0,
    // cr3, eip, eflags, eax, ecx, edx, ebx, esp, ebp, esi, edi,
    (uint32_t)pg_dir,  (uint32_t)task_0/*入口地址*/, 0x202, 0xa, 0xc, 0xd, 0xb, (uint32_t)task0_dpl3_stack + 4*1024/* 栈 */, 0x1, 0x2, 0x3,
    // es, cs, ss, ds, fs, gs, ldt, iomap
    APP_DATA_SEG, APP_CODE_SEG, APP_DATA_SEG, APP_DATA_SEG, APP_DATA_SEG, APP_DATA_SEG, 0x0, 0x0,
};

uint32_t task1_tss[] = {
    // prelink, esp0, ss0, esp1, ss1, esp2, ss2
    0,  (uint32_t)task1_dpl0_stack + 4*1024, KERNEL_DATA_SEG , /* 后边不用使用 */ 0x0, 0x0, 0x0, 0x0,
    // cr3, eip, eflags, eax, ecx, edx, ebx, esp, ebp, esi, edi,
    (uint32_t)pg_dir,  (uint32_t)task_1/*入口地址*/, 0x202, 0xa, 0xc, 0xd, 0xb, (uint32_t)task1_dpl3_stack + 4*1024/* 栈 */, 0x1, 0x2, 0x3,
    // es, cs, ss, ds, fs, gs, ldt, iomap
    APP_DATA_SEG, APP_CODE_SEG, APP_DATA_SEG, APP_DATA_SEG, APP_DATA_SEG, APP_DATA_SEG, 0x0, 0x0,
};

```


4. 将当前进程寄存器tr指向task_0

```c
// 设置当前的任务0
// 模拟中断返回，返回至任务0，特权级3模式
mov $TASK0_TSS_SEL, %ax		// 加载任务0对应的状态段
ltr %ax

```

5. 添加调度程序进行跳转

```c
timer_init:
	... 
	// 内核寄存器入栈
	outb %al, $0x20 // 发送EOI
	// 使用内核的数据段寄存器 ??? 
	mov $KERNEL_DATA_SEG, %ax
	mov %ax, %ds
	call task_sched // 调用调试函数
	// 内核寄存器出栈
	...

```

```c

void task_sched (void) {
    static int task_tss = TASK0_TSS_SEL;

    // 更换当前任务的tss，然后切换过去
    task_tss = (task_tss == TASK0_TSS_SEL) ? TASK1_TSS_SEL : TASK0_TSS_SEL;
    uint32_t addr[] = {0, task_tss };
    __asm__ __volatile__("ljmpl *(%[a])"::[a]"r"(addr));
}

```


## 11. 系统调用

....待补充















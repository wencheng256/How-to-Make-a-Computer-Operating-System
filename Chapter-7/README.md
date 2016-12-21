##第七章: IDT 和中断

中断是一种被硬件或者软件发送给处理器用来引起处理器注意的的信号。

一共有三种类型的中断：

- **硬件中断：** 扩展设备（键盘、鼠标、硬盘等等）传输给处理器的中断 。硬件中断是一种减少cpu资源浪费在等待扩展设备响应做法。
- **软件中断** 由软件触发，用来管理系统调用。
- **异常：**  当程序运行当中发生了程序自身无法处理的异常时会抛出一个异常中断。

#### 例如键盘：

当用户按下键盘上一个按键，键盘控制器将会发送一个中断到中断控制器。如果中断没有被标记，则控制器将会向处理器发送这个中断，处理器将会执行一段程序处理中断（键盘按下或松开），比如这段程序可能从键盘控制器获取被按下的按键的信息然后在屏幕上打印出被按下的字母。 当字符处理程序被完成以后，刚刚被中断的工作就可以继续执行了。

#### 什么是PIC？

[PIC](http://en.wikipedia.org/wiki/Programmable_Interrupt_Controller) (Programmable interrupt controller 程序中断控制器)是一个能将多种来源的终端信息合成一条或者更多cpu指令的设备， 同时支持中断输出优先级的设置。当中断控制器同时收到了多个中断去处理，他会依据中断的相对优先级来排序。

最为出名的PIC是8259A，每个8259A可以处理8个设备，但大多数的电脑拥有两个控制器： 一主一从，这使得电脑可以同时管理14个设备的中断。

本章中，我们将对控制器进行编程以对中断进行初始化和掩码。

#### 什么是 IDT？

> 中断描述表(IDT) 是x86架构机为实现中断向量表使用的数据结构。处理器使用根据IDT对中断和异常进行正确的返回。

我们的内核将使用IDT定义若干方法以处理中断。

就像GDT一样，IDT使用LIDT汇编指令进行装载，IDT内存地址结构如下：

```cpp
struct idtr {
	u16 limite;
	u32 base;
} __attribute__ ((packed));
```

IDT表由以下结构的IDT区块组成：

```cpp
struct idtdesc {
	u16 offset0_15;
	u16 select;
	u16 type;
	u16 offset16_31;
} __attribute__ ((packed));
```

**注意:** ` __attribute__ ((packed))`指令用来引导gcc，这个结构体应该是用尽可能少的内存。如果没有这行代码，gcc会引入一些字节去优化执行时内存的分配和获取。

现在我们需要定义我们的IDT表，然后使用LIDT进行装载。同GDR，IDT可以被放置到内存的任意位置，只要我们使用IDTR注册了IDT的地址，处理器就可以获取到。

常见中断表如下（可屏蔽硬件中断被称作IRQ）:


| IRQ   |         Description        |
|:-----:| -------------------------- |
| 0 | 可编程中断，定时器中断 |
| 1 | 键盘中断 |
| 2 | 瀑布流中断 (被两个PIC使用的内部中断 不会遇到) |
| 3 | COM2 (如果存在的话) |
| 4 | COM1 (如果存在的话) |
| 5 | LPT2 (如果存在的话) |
| 6 | 软盘 |
| 7 | LPT1 |
| 8 | CMOS 实时时钟 (如果存在的话) |
| 9 | 供其他外设自由使用 / legacy SCSI / NIC |
| 10 | 供其他外设自由使用  / SCSI / NIC |
| 11 | 供其他外设自由使用  / SCSI / NIC |
| 12 | PS2 鼠标 |
| 13 | FPU / 协处理器 / Inter-processor |
| 14 | 主 ATA 硬盘 |
| 15 | 二级 ATA 硬盘 |

#### 如何初始化中断？

这里有个定义IDT区块的简单方法：

```cpp
void init_idt_desc(u16 select, u32 offset, u16 type, struct idtdesc *desc)
{
	desc->offset0_15 = (offset & 0xffff);
	desc->select = select;
	desc->type = type;
	desc->offset16_31 = (offset & 0xffff0000) >> 16;
	return;
}
```

我们现在可以初始化中断了：

```cpp
#define IDTBASE	0x00000000
#define IDTSIZE 0xFF
idtr kidtr;
```


```cpp
void init_idt(void)
{
	/* Init irq */
	int i;
	for (i = 0; i < IDTSIZE; i++)
		init_idt_desc(0x08, (u32)_asm_schedule, INTGATE, &kidt[i]); //

	/* Vectors  0 -> 31 are for exceptions */
	init_idt_desc(0x08, (u32) _asm_exc_GP, INTGATE, &kidt[13]);		/* #GP */
	init_idt_desc(0x08, (u32) _asm_exc_PF, INTGATE, &kidt[14]);     /* #PF */

	init_idt_desc(0x08, (u32) _asm_schedule, INTGATE, &kidt[32]);
	init_idt_desc(0x08, (u32) _asm_int_1, INTGATE, &kidt[33]);

	init_idt_desc(0x08, (u32) _asm_syscalls, TRAPGATE, &kidt[48]);
	init_idt_desc(0x08, (u32) _asm_syscalls, TRAPGATE, &kidt[128]); //48

	kidtr.limite = IDTSIZE * 8;
	kidtr.base = IDTBASE;


	/* Copy the IDT to the memory */
	memcpy((char *) kidtr.base, (char *) kidt, kidtr.limite);

	/* Load the IDTR registry */
	asm("lidtl (kidtr)");
}
```

初始化了IDT之后，我们需要配置pic以让中断生效。以下的方法会使用程序的输出端口在两个PIC的注册表写入以配置它们。

```io.outb```. 我们使用这些端口配置PIC:


* 主 PIC: 0x20 and 0x21
* 从 PIC: 0xA0 and 0xA1

每个PIC，有两种方式可以注册它们：

* ICW (Initialization Command Word): reinit the controller
* OCW (Operation Control Word): configure the controller once initialized (used to mask/unmask the interrupts)

```cpp
void init_pic(void)
{
	/* Initialization of ICW1 */
	io.outb(0x20, 0x11);
	io.outb(0xA0, 0x11);

	/* Initialization of ICW2 */
	io.outb(0x21, 0x20);	/* start vector = 32 */
	io.outb(0xA1, 0x70);	/* start vector = 96 */

	/* Initialization of ICW3 */
	io.outb(0x21, 0x04);
	io.outb(0xA1, 0x02);

	/* Initialization of ICW4 */
	io.outb(0x21, 0x01);
	io.outb(0xA1, 0x01);

	/* mask interrupts */
	io.outb(0x21, 0x0);
	io.outb(0xA1, 0x0);
}
```

#### PIC ICW configurations details

The registries have to be configured in order.

**ICW1 (port 0x20 / port 0xA0)**
```
|0|0|0|1|x|0|x|x|
         |   | +--- with ICW4 (1) or without (0)
         |   +----- one controller (1), or cascade (0)
         +--------- triggering by level (level) (1) or by edge (edge) (0)
```

**ICW2 (port 0x21 / port 0xA1)**
```
|x|x|x|x|x|0|0|0|
 | | | | |
 +----------------- base address for interrupts vectors
```

**ICW2 (port 0x21 / port 0xA1)**

For the master:
```
|x|x|x|x|x|x|x|x|
 | | | | | | | |
 +------------------ slave controller connected to the port yes (1), or no (0)
```

For the slave:
```
|0|0|0|0|0|x|x|x|  pour l'esclave
           | | |
           +-------- Slave ID which is equal to the master port
```

**ICW4 (port 0x21 / port 0xA1)**

It is used to define in which mode the controller should work.

```
|0|0|0|x|x|x|x|1|
       | | | +------ mode "automatic end of interrupt" AEOI (1)
       | | +-------- mode buffered slave (0) or master (1)
       | +---------- mode buffered (1)
       +------------ mode "fully nested" (1)
```

#### Why do idt segments offset our ASM functions?

You should have noticed that when I'm initializing our IDT segments, I'm using offsets to segment the code in Assembly. The different functions are defined in [x86int.asm](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/x86int.asm) and are of the following scheme:

```asm
%macro	SAVE_REGS 0
	pushad
	push ds
	push es
	push fs
	push gs
	push ebx
	mov bx,0x10
	mov ds,bx
	pop ebx
%endmacro

%macro	RESTORE_REGS 0
	pop gs
	pop fs
	pop es
	pop ds
	popad
%endmacro

%macro	INTERRUPT 1
global _asm_int_%1
_asm_int_%1:
	SAVE_REGS
	push %1
	call isr_default_int
	pop eax	;;a enlever sinon
	mov al,0x20
	out 0x20,al
	RESTORE_REGS
	iret
%endmacro
```

These macros will be used to define the interrupt segment that will prevent corruption of the different registries, it will be very useful for multitasking.

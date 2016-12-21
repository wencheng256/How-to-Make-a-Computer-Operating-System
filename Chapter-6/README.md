##第六章：GDT

多亏了GRUB，我们的内核可以不用在16位下运行，并且早已进入了[保护模式](http://en.wikipedia.org/wiki/Protected_mode)，保护模式让可以让我们发挥微处理器的最大能力，比如虚拟内存管理，分页和安全的多任务。

#### 什么是 GDT？

[GDT](http://en.wikipedia.org/wiki/Global_Descriptor_Table) ("Global Descriptor Table")是一种用来定义不同内存区域的数据结构： 基础地址、大小、读写执行权限等等。这些内存分区被称作区块。

我们将使用GDT 定义不同的内存区块：

* *"code"*: 内核代码区，用来存储可执行的二进制代码。
* *"data"*: 内核数据区
* *"stack"*: 内核栈区，用来存储内核执行时的调用栈
* *"ucode"*: 用户代码区，用来存储用户程序的可执行二进制代码。
* *"udata"*: 用户数据区
* *"ustack"*: 用户栈区， 用来存储在用户区执行的调用栈

#### 如何加载我们的 GDT？

GRUB 初始化了一个GDT，但这个GDT并不和我们的内核对应。
使用汇编命令LGDT可以装在GDT。所描述的地址结构如下。

![GDTR](./gdtr.png)

C定义的结构体如下：

```cpp
struct gdtr {
	u16 limite;
	u32 base;
} __attribute__ ((packed));
```

**注意：** ```__attribute__ ((packed))```指令用来引导gcc，这个结构体应该是用尽可能少的内存。如果没有这行代码，gcc会引入一些字节去优化执行时内存的分配和获取。

我们现在去使用LGDT去定义和装载GDT表。GDT表可以被放置在内存中的任意地方， 它的内存地址应该使用GDTR注册到程序中。

GDT 表由下列结构描述的区块组成：

![GDTR](./gdtentry.png)

使用C结构体描述如下：

```cpp
struct gdtdesc {
	u16 lim0_15;
	u16 base0_15;
	u8 base16_23;
	u8 acces;
	u8 lim16_19:4;
	u8 other:4;
	u8 base24_31;
} __attribute__ ((packed));
```

#### 如何定义我们的GDT表？

我们需要知道如何在内存中定义GDT，以及如何从GDTR注册中读取GDT。

我们将在以下地址放置我们的GDT表：

```cpp
#define GDTBASE	0x00000800
```

 **init_gdt_desc** in [x86.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/x86.cc) 方法初始化了GDT"区块"的描述。


```cpp
void init_gdt_desc(u32 base, u32 limite, u8 acces, u8 other, struct gdtdesc *desc)
{
	desc->lim0_15 = (limite & 0xffff);
	desc->base0_15 = (base & 0xffff);
	desc->base16_23 = (base & 0xff0000) >> 16;
	desc->acces = acces;
	desc->lim16_19 = (limite & 0xf0000) >> 16;
	desc->other = (other & 0xf);
	desc->base24_31 = (base & 0xff000000) >> 24;
	return;
}
```

 **init_gdt**方法初始化GDT，下面方法的一部分会在多任务的时候再次用到，到那时在进行介绍。

```cpp
void init_gdt(void)
{
	default_tss.debug_flag = 0x00;
	default_tss.io_map = 0x00;
	default_tss.esp0 = 0x1FFF0;
	default_tss.ss0 = 0x18;

	/* initialize gdt segments */
	init_gdt_desc(0x0, 0x0, 0x0, 0x0, &kgdt[0]);
	init_gdt_desc(0x0, 0xFFFFF, 0x9B, 0x0D, &kgdt[1]);	/* code */
	init_gdt_desc(0x0, 0xFFFFF, 0x93, 0x0D, &kgdt[2]);	/* data */
	init_gdt_desc(0x0, 0x0, 0x97, 0x0D, &kgdt[3]);		/* stack */

	init_gdt_desc(0x0, 0xFFFFF, 0xFF, 0x0D, &kgdt[4]);	/* ucode */
	init_gdt_desc(0x0, 0xFFFFF, 0xF3, 0x0D, &kgdt[5]);	/* udata */
	init_gdt_desc(0x0, 0x0, 0xF7, 0x0D, &kgdt[6]);		/* ustack */

	init_gdt_desc((u32) & default_tss, 0x67, 0xE9, 0x00, &kgdt[7]);	/* descripteur de tss */

	/* initialize the gdtr structure */
	kgdtr.limite = GDTSIZE * 8;
	kgdtr.base = GDTBASE;

	/* copy the gdtr to its memory area */
	memcpy((char *) kgdtr.base, (char *) kgdt, kgdtr.limite);

	/* load the gdtr registry */
	asm("lgdtl (kgdtr)");

	/* initiliaz the segments */
	asm("   movw $0x10, %ax	\n \
            movw %ax, %ds	\n \
            movw %ax, %es	\n \
            movw %ax, %fs	\n \
            movw %ax, %gs	\n \
            ljmp $0x08, $next	\n \
            next:		\n");
}
```

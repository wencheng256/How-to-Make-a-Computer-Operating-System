
## 第三章: 使用 GRUB 初始化引导

#### 引导是一个怎么样的过程？

当x86架构的电脑启动，它将进入一个非常复杂的过程，直到将控制权转移到我们的内核主程序手中 (`kmain()`). 本课程中，我们只考虑BIOS引导方法的情况，而不考虑他的后继者（UEFI）。


BIOS 引导的顺序是：RAM 探查->硬件探查/初始化->“引导序列”


对我们来说，最重要的部分就是“引导序列”。
引导序列指的是：当BIOS完成了初始化过程，准备将控制权转移到一个引导装载程序的过程。


根据“引导序列”，BIOS将会确定一个“引导设备”(例如：软盘、硬盘、CD、USB闪存、或者网络设备)。我们的OS最初会从硬盘中引导 （但是以后也可能从CD或者一个USB闪存中引导）如果一个设备的第511和512字节分别包含指定的签名`0x55`和`0xAA`，那么这个设备将会被认为是可引导设备(被称作主引导记录的魔术代码，或者被称为 MBR）。签名的二进制编码为 `0b1010101001010101`。这些交替的01位被视作是应对某些特定错误的保护，如果这串序列被篡改或者抹除，那么这个设备不会被认作引导设备。

BIOS会依次读取每一个设备的前512字节到物理内存从`0x7C00`的部分 （32Kb前的1Kb）。 如果设备拥有校验签名的话，BIOS将会将指令移动至`0x7C00`处 （通过跳转指令）以执行引导扇区的代码。

到此为止CPU一致运行在16位实时模式下，这是x86CPU的默认模式，之所以这样是为了保证向后兼容性。现在，为了执行我们以32位指令编写的内核，引导程序必须将CPU转换成保护模式。

#### 什么是 GRUB？

> GNU GRUB （GNU GRand Unified Bootloader） 是GNU项目的一个引导装载包。 GRUB 是参考自由软件基金会的多引导规范的一个实现，他为用户提供了一个选择界面以选取某个特定的已经安装的操作系统，或者安装在某个盘的相同操作系统的不同内核配置。


简而言之， GRUB 是被计算机（引导装载器）装载的第一个 程序 而且将会明确哪一个系统内核最终会被我们从硬盘引导出来。

#### 我们为什么使用GRUB？

* GRUB 用起来十分简单
* 可以完全不使用16位指令代码就可以装载32位系统内核。
* 多引导选择启动Linux，Windows或者其他操作系统
* 更容易将拓展模块装载到内存中

#### 如何使用 GRUB？

GRUB 遵循多引导协议, 可执行二进制代码必须为32位而且必须在最初的8192字节包含铁定的多引导头。我们的内核必须为ELF可执行文件（"Executable and Linkable Format"，一个在大多数UNIX系统中通用的可执行文件的标准格式）。

The first boot sequence of our kernel is written in Assembly: [start.asm](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/start.asm) and we use a linker file to define our executable structure: [linker.ld](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/linker.ld).

我们操作系统内核的第一个引导序列使用Assemble书写: [start.asm](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/start.asm) 我们使用链接文件定义我们的可执行结构 [linker.ld](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/linker.ld).

引导程序同样会初始化一些我们所使用的C++运行环境，这些将在下一章进行介绍

多引导头结构体：

```cpp
struct multiboot_info {
	u32 flags;
	u32 low_mem;
	u32 high_mem;
	u32 boot_device;
	u32 cmdline;
	u32 mods_count;
	u32 mods_addr;
	struct {
		u32 num;
		u32 size;
		u32 addr;
		u32 shndx;
	} elf_sec;
	unsigned long mmap_length;
	unsigned long mmap_addr;
	unsigned long drives_length;
	unsigned long drives_addr;
	unsigned long config_table;
	unsigned long boot_loader_name;
	unsigned long apm_table;
	unsigned long vbe_control_info;
	unsigned long vbe_mode_info;
	unsigned long vbe_mode;
	unsigned long vbe_interface_seg;
	unsigned long vbe_interface_off;
	unsigned long vbe_interface_len;
};
```

你可以使用命令 ```mbchk kernel.elf``` 以多引导标准校验你的kernal.elf
同样可以使用命令 ```nm -n kernel.elf``` 校验ELF二进制代码中不同对象之间的距离。

####为我们的内核和GRUB创建一个磁盘镜像

脚本文件 [diskimage.sh](https://github.com/wencheng256/How-to-Make-a-Computer-Operating-System/blob/master/src/sdk/diskimage.sh) 会自动生成一个可以被QEMU使用的磁盘镜像

创建磁盘镜像(c.img) 的第一步是使用qumu-img命令:

```
qemu-img create c.img 2M
```

我们现在要使用fdisk命令为这个disk进行分区：

```bash
fdisk ./c.img

# Switch to Expert commands
> x

# Change number of cylinders (1-1048576)
> c
> 4

# Change number of heads (1-256, default 16):
> h
> 16

# Change number of sectors/track (1-63, default 63)
> s
> 63

# Return to main menu
> r

# Add a new partition
> n

# Choose primary partition
> p

# Choose partition number
> 1

# Choose first sector (1-4, default 1)
> 1

# Choose last sector, +cylinders or +size{K,M,G} (1-4, default 4)
> 4

# Toggle bootable flag
> a

# Choose first partition for bootable flag
> 1

# Write table to disk and exit
> w
```

We need now to attach the created partition to the loop-device using losetup. This allows a file to be access like a block device. The offset of the partition is passed as an argument and calculated using: **offset= start_sector * bytes_by_sector**.

我们现在要使用loseup命令将创建好的分区装载到loop-device上。这样使一个文件可以向一个块状设备一样使用。 分区之间的间距被当做一个参数传入，计算方式如下 **间距= 起始扇区* 每个扇区的字节数量**.

通过```fdisk -l -u c.img```, 得: 63 * 512 = 32256.

```bash
losetup -o 32256 /dev/loop1 ./c.img
```
使用以下命令在新设备上创建一个EXT2文件系统：

```bash
mke2fs /dev/loop1
```

将盘挂载，然后拷贝你的文件：

```bash
mount  /dev/loop1 /mnt/
cp -R bootdisk/* /mnt/
umount /mnt/
```

在盘上安装GRUB：

```bash
grub --device-map=/dev/null << EOF
device (hd0) ./c.img
geometry (hd0) 4 16 63
root (hd0,0)
setup (hd0)
quit
EOF
```

最后我们释放这个虚拟设备：

```bash
losetup -d /dev/loop1
```

#### 参考资料

* [GNU GRUB on Wikipedia](http://en.wikipedia.org/wiki/GNU_GRUB)
* [Multiboot specification](https://www.gnu.org/software/grub/manual/multiboot/multiboot.html)

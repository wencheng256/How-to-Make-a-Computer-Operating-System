## 第四章：OS的骨架和C++运行环境

#### C++ 内核运行环境

内核可以使用C或者C++编写，但是为了避免一些可能的陷阱，我们决定使用C++（运行环境支持、构造函数等等）


编译器会假定所有的C++运行环境都是默认支持的，但是我们并不准备把libsupc++连接进我们的内核，所以我们要在 [cxx.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/cxx.cc) 文件中添加一些基础方法，以便我们使用。

**注意:** 不能在虚拟内存和分页系统初始化之前使用`new`和`delete`命令。

#### 基础 C/C++ 方法

内核代码不能使用标准库的代码，所以我们需要为管理内存、字符串等功能添加一些基本方法。

```cpp
void 	itoa(char *buf, unsigned long int n, int base);

void *	memset(char *dst,char src, int n);
void *	memcpy(char *dst, char *src, int n);

int 	strlen(char *s);
int 	strcmp(const char *dst, char *src);
int 	strcpy(char *dst,const char *src);
void 	strcat(void *dest,const void *src);
char *	strncpy(char *destString, const char *sourceString,int maxLength);
int 	strncmp( const char* s1, const char* s2, int c );
```

这些方法在 [string.cc](https://github.com/wencheng256/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/string.cc), [memory.cc](https://github.com/wencheng256/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/memory.cc), [itoa.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/itoa.cc)等文件中定义

#### C 类型

下一步，我们将定义一些我们常用的类型。大多数我们所需要使用的类型都是unsigned。即所有位都是数据位的数字类型。 Signed 类型使用第一位来记录符号。
```cpp
typedef unsigned char 	u8;
typedef unsigned short 	u16;
typedef unsigned int 	u32;
typedef unsigned long long 	u64;

typedef signed char 	s8;
typedef signed short 	s16;
typedef signed int 		s32;
typedef signed long long	s64;
```

#### 编译内核

编译内核和编译一个linux可执行文件不太相同，因为我们不能使用标准库，也不能依赖于操作系统。

我们的[Makefile](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/Makefile) 定义了编译并链接我们内核的处理过程。

x86 架构中， 这些选项将在 gcc/g++/ld中使用:

```
# Linker
LD=ld
LDFLAG= -melf_i386 -static  -L ./  -T ./arch/$(ARCH)/linker.ld

# C++ compiler
SC=g++
FLAG= $(INCDIR) -g -O2 -w -trigraphs -fno-builtin  -fno-exceptions -fno-stack-protector -O0 -m32  -fno-rtti -nostdlib -nodefaultlibs 

# Assembly compiler
ASM=nasm
ASMFLAG=-f elf -o
```

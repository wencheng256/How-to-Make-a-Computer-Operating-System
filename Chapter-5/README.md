## 第五章：管理x86架构的基础类

现在，我们知道如何编译C++内核，知道如何使用GRUB引导二进制代码了，我们可以开始使用C/C++做一些比较酷的事情了。

#### 向屏幕打印字符

我们使用VGA的默认模式（03h）向用户展示一些字符。使用从0xB8000开始的内存位，我们可以很容易的获取到屏幕。 屏幕的分辨率是80*25 并且每个屏幕上的字符使用2个字节： 一个字节用来定义字符，另一个字节用来定义字符样式。 这意味着总显存为4000B （80*25*2B）。

在我们的IO类 ([io.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/io.cc))中:
* **x,y**: 用来定义光标在屏幕上的位置
* **real_screen**: 用来定位显存指针
* **putc(char c)**: 用来向屏幕打印字符，并移动光标的位置
* **printf(char* s, ...)**: 打印字符串

我们在 [IO Class](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/io.cc)中增加一个putc方法，向屏幕中打印一个字符，并且更新光标位置。

```cpp
/* 向屏幕中打印字符 */
void Io::putc(char c){
	kattr = 0x07;
	unsigned char *video;
	video = (unsigned char *) (real_screen+ 2 * x + 160 * y);
	// newline
	if (c == '\n') {
		x = 0;
		y++;
	// back space
	} else if (c == '\b') {
		if (x) {
			*(video + 1) = 0x0;
			x--;
		}
	// horizontal tab
	} else if (c == '\t') {
		x = x + 8 - (x % 8);
	// carriage return
	} else if (c == '\r') {
		x = 0;
	} else {
		*video = c;
		*(video + 1) = kattr;

		x++;
		if (x > 79) {
			x = 0;
			y++;
		}
	}
	if (y > 24)
		scrollup(y - 24);
}
```

我们同样添加了非常著名切有效的方法： [printf](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/io.cc#L155)

```cpp
/* 打印字符串 */
void Io::print(const char *s, ...){
	va_list ap;

	char buf[16];
	int i, j, size, buflen, neg;

	unsigned char c;
	int ival;
	unsigned int uival;

	va_start(ap, s);

	while ((c = *s++)) {
		size = 0;
		neg = 0;

		if (c == 0)
			break;
		else if (c == '%') {
			c = *s++;
			if (c >= '0' && c <= '9') {
				size = c - '0';
				c = *s++;
			}

			if (c == 'd') {
				ival = va_arg(ap, int);
				if (ival < 0) {
					uival = 0 - ival;
					neg++;
				} else
					uival = ival;
				itoa(buf, uival, 10);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				if (neg)
					print("-%s", buf);
				else
					print(buf);
			}
			 else if (c == 'u') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 10);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print(buf);
			} else if (c == 'x' || c == 'X') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 16);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print("0x%s", buf);
			} else if (c == 'p') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 16);
				size = 8;

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print("0x%s", buf);
			} else if (c == 's') {
				print((char *) va_arg(ap, int));
			}
		} else
			putc(c);
	}

	return;
}
```

#### 汇编语言对外接口

在汇编语言中，有很多命令非常有用，但是却并不能在C中使用 （例如cli, sti, in and out）， 所以我们需要为这些指令添加一个对外接口。

在C语言中，我们可以使用`asm()`方法直接引入Assembly，gcc编译器会使用gas处理这些汇编语言。

**Caution:** gas使用AT&T汇编语法

```cpp
/* output byte */
void Io::outb(u32 ad, u8 v){
	asmv("outb %%al, %%dx" :: "d" (ad), "a" (v));;
}
/* output word */
void Io::outw(u32 ad, u16 v){
	asmv("outw %%ax, %%dx" :: "d" (ad), "a" (v));
}
/* output word */
void Io::outl(u32 ad, u32 v){
	asmv("outl %%eax, %%dx" : : "d" (ad), "a" (v));
}
/* input byte */
u8 Io::inb(u32 ad){
	u8 _v;       \
	asmv("inb %%dx, %%al" : "=a" (_v) : "d" (ad)); \
	return _v;
}
/* input word */
u16	Io::inw(u32 ad){
	u16 _v;			\
	asmv("inw %%dx, %%ax" : "=a" (_v) : "d" (ad));	\
	return _v;
}
/* input word */
u32	Io::inl(u32 ad){
	u32 _v;			\
	asmv("inl %%dx, %%eax" : "=a" (_v) : "d" (ad));	\
	return _v;
}
```

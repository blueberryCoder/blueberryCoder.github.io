---
title: Android动态链接技术
date: 2025-09-06 10:43:41
tags:
---


# 前言
Android开发中，涉及Native（C/C++等）程序、接入外部动态库、处理线上崩溃、Native Hook、加固、逆向等。都需要了解到动态链接技术。

# 动态库与位置⽆关代码

## 共享库
使用一些高级语言（C/C++）最终编译后产物是二进制文件。我们通常引用第三方库，会使用它的动态共享库。同时我们自己开发的二方库本身也可能以来其他的共享库，也可能给别人提供共享库。
同时我们为了代码复用，多个Module复用同样的代码时，我们可以将这些代码剥离处理做成一个共享库供其使用。
<img src="/images/android/native_shared_library.png" alt="shared_library" width="720">
共享库又分为静态库库和动态共享库。

## 静态共享库
静态库是编译时依赖，即编译程序时，会将共享库的内容打包到可执行程序中（假设我们是一个可执行程序，依赖一个共享库）。而静态库不会作为单独的文件和可执行程序发布。
<img src="/images/android/native_dynamic_shared_library.png" alt="dynamic_shared" width="720">

## 动态共享库
动态库是运行时以来，即编译程序时，不会将共享库的内容打包到可执行程序。发布程序时需要将动态库一起发布。
<img src="/images/android/native_static_shared_library.png" alt="static_shared" width="720">

## 位置无关代码
因为动态库是要运行时动态加载，其加载位置即运行时地址是不固定的。因此需要保证代码时位置无关的(PIC, Position Idependent Code)。
为了接下来的演示，我们在Android Arm64平台写下如下代码:
```c++
#include <jni.h>
#include <string>

extern "C" {
    void SayHello() {
        printf("Hello World!");
    }

    void SayHelloWrapper() {
        static int i = 0 ;
        while(1)  {
            if (i++ % 2  == 0) {
                break;
            }
        }
        SayHello();
    }
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_blueberry_anative_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = "Hello from C++";

    SayHelloWrapper();
    return env->NewStringUTF(hello.c_str());
}

```
这段代码编译完后，我们可以在arm64-v8a目录中得到动态(libanative.so)。

我们使用`objdump -d libanative.so` 对产物进行反汇编。可以得到：
```asm
...
   1dc68:	d65f03c0 	ret

000000000001dc6c <SayHelloWrapper>:
// 指令地址  机器码       汇编指令                 符号名（可选）
   1dc6c:	a9bf7bfd 	stp	x29, x30, [sp, #-16]!
   1dc70:	910003fd 	mov	x29, sp
   1dc74:	14000001 	b	1dc78 <SayHelloWrapper+0xc>
   1dc78:	f000014a 	adrp	x10, 48000 <_DYNAMIC+0x12b8>
   1dc7c:	b9446148 	ldr	w8, [x10, #1120]
   1dc80:	11000509 	add	w9, w8, #0x1
   1dc84:	b9046149 	str	w9, [x10, #1120]
   1dc88:	5280004a 	mov	w10, #0x2                   	// #2
   1dc8c:	1aca0d09 	sdiv	w9, w8, w10
   1dc90:	1b0a7d29 	mul	w9, w9, w10
   1dc94:	6b090108 	subs	w8, w8, w9
   1dc98:	35000068 	cbnz	w8, 1dca4 <SayHelloWrapper+0x38>
   1dc9c:	14000001 	b	1dca0 <SayHelloWrapper+0x34>
   1dca0:	14000002 	b	1dca8 <SayHelloWrapper+0x3c>
   1dca4:	17fffff5 	b	1dc78 <SayHelloWrapper+0xc>
   1dca8:	9400920e 	bl	424e0 <SayHello@plt>
   1dcac:	a8c17bfd 	ldp	x29, x30, [sp], #16
   1dcb0:	d65f03c0 	ret

000000000001dcb4 <Java_com_blueberry_anative_MainActivity_stringFromJNI>:
   1dcb4:	d10183ff 	sub	sp, sp, #0x60
...
```
上面的是SayHelloWrapper这个函数对应的汇编指令，我们来看这一行：
```asm
1dca0:	14000002 	b	1dca8 <SayHelloWrapper+0x3c>
```
这条指令对应的汇编语言是跳转到1dca8地址，即当前指令地址+2的位置。虽然反汇编的结果是直接指向这个地址值，但它的指令其实用的是相对偏移。这一点从它的机器码的地位其实可以看出端倪（它的末尾是2）。另外这些指令的地址都是想对地址并不是动态库被加载后的运行时的地址。运行时的真实地址是BASE+这里的相对地址。BASE为加载动态库时系统分配基地址。
这里的指令集时AArch64指令集，其中b指令的格式为：
```
31  30........5    0
|  opcode=000101   | imm26 |
- 高 6 位固定 = 000101 (0x05)
- 剩下 26 位是带符号偏移量 imm26（相对于当前指令地址，单位是 4 字节）
```
0x14000002的二进制是0001 0100 0000 0000 0000 0000 0000 0010
opcode = 000101，imm26 = 0x0000002 的偏移量 = 2 * 4 = 8 字节
也就是跳转的目标地址是距离当前指令地址+8字节的地址，因为当前指令的地址是：1dca0，所有目标地址是：1dca8。
那么无论这个动态库被加载在什么位置都不影响这个跳转指令的正确性。

## 绝对地址跳转

当跨模块调用时，比如我们上面的代码在SayHello中使用了libc的printf函数。编译器在编译期间是无法知道这个函数的地址的，它是怎么保证跳转的正确呢？
那么我们先来来使用objdump看下SayHello函数的汇编指令：
```asm

000000000001dc50 <SayHello>:
   1dc50:	a9bf7bfd 	stp	x29, x30, [sp, #-16]!
   1dc54:	910003fd 	mov	x29, sp
   1dc58:	d0ffffa0 	adrp	x0, 13000 <GCC_except_table133+0x14>
   1dc5c:	913be000 	add	x0, x0, #0xef8
   1dc60:	94009224 	bl	424f0 <printf@plt>
   1dc64:	a8c17bfd 	ldp	x29, x30, [sp], #16
   1dc68:	d65f03c0 	ret
```
其中
```
   1dc60:	94009224 	bl	424f0 <printf@plt>  
```
是想跳转到printf函数，它跳转的地址是424f0，对应的符号是printf@plt。
我们使用`objdump -d -j .plt libanative.so`查看对应plt表，得到：
```asm
...
00000000000424f0 <printf@plt>:
   424f0:	b0000030 	adrp	x16, 47000 <_ZTVSt9bad_alloc@@Base+0x740>
   424f4:	f9402e11 	ldr	x17, [x16, #88]
   424f8:	91016210 	add	x16, x16, #0x58
   424fc:	d61f0220 	br	x17
...
```
可以看到地址00000000000424f0对应的代码段，它只有4条指令。这段代码通常被称为“蹦床（trampoline"或跳板函数(stbus)。下面我们逐行解释写这段代码。

```asm
...
00000000000424f0 <printf@plt>:
   // 将47000（.got.plt表的基址）放到x16寄存器。
   424f0:	b0000030 	adrp	x16, 47000 <_ZTVSt9bad_alloc@@Base+0x740>
   // 将x16寄存器的值+88得到的地址存放到x17寄存器。（即got表中这个函数对应的slot地址）
   424f4:	f9402e11 	ldr	x17, [x16, #88]
   // 将x16寄存器的值自加0x58也就是十进制88,（也即got表中这个函数对应的slot地址）用户之后的resolve回填函数地址。
   424f8:	91016210 	add	x16, x16, #0x58
   // 跳转到x17中存放的地址（即got表中对应的地址）。
   424fc:	d61f0220 	br	x17
...
```
上面的代码首先会找到GOT表的基地址，然后加上当前函数对应在got表slot槽的偏移。然后望这个slot对应的地址跳转，并且用x16保存了slot的地址。x16存放的是slot的地址，x17存放的是slot中值。slot中存放的值是目标函数的地址（假设已经完成延时绑定的话），即printf函数的地址。
下面我们来看got表的信息，着重也看got表0x58偏移对应的内容（got表的基地址为0x47000）。
因为.got.plt是数据段，我们用`objdump -s -j .got -j .got.plt libanative.so`来查看：
.got用来保存全局变量引用的地址，.got.plt用来保存函数引用的地址。

```asm
➜  arm64-v8a objdump -s -j .got  -j .got.plt libanative.so

libanative.so:     file format elf64-littleaarch64

Contents of section .got:
 46f18 00000000 00000000 00000000 00000000  ................
 46f28 00000000 00000000 00000000 00000000  ................
 46f38 00000000 00000000 00000000 00000000  ................
 46f48 00000000 00000000 00000000 00000000  ................
 46f58 00000000 00000000 00000000 00000000  ................
 46f68 00000000 00000000 00000000 00000000  ................
 46f78 00000000 00000000 00000000 00000000  ................
 46f88 00000000 00000000 00000000 00000000  ................
 46f98 00000000 00000000 00000000 00000000  ................
 46fa8 00000000 00000000 00000000 00000000  ................
 46fb8 00000000 00000000 00000000 00000000  ................
 46fc8 00000000 00000000 00000000 00000000  ................
 46fd8 00000000 00000000 00000000 00000000  ................
 46fe8 00000000 00000000 00000000 00000000  ................
 46ff8 00000000 00000000 00000000 00000000  ................
 47008 00000000 00000000 00000000 00000000  ................
 47018 00000000 00000000                    ........
Contents of section .got.plt:
 47020 00000000 00000000 00000000 00000000  ................
 47030 00000000 00000000 90240400 00000000  .........$......
 47040 90240400 00000000 90240400 00000000  .$.......$......
 47050 90240400 00000000 90240400 00000000  .$.......$......
 47060 90240400 00000000 90240400 00000000  .$.......$......
 47070 90240400 00000000 90240400 00000000  .$.......$......
 47080 90240400 00000000 90240400 00000000  .$.......$......
 47090 90240400 00000000 90240400 00000000  .$.......$......
 470a0 90240400 00000000 90240400 00000000  .$.......$......
```
我们可以看到这个数据是以小端方式排列的，并且一行显示16字节，那么0x47000 + 0x58对应的槽的就在数据的第4行第3列。
它的值为90240400 00000000，另外我们可以看到got表中前0x47038前的都是0，后面每个槽的值都是90240400 00000000 。（这里每个槽都是8个字节）。

这里是因为，GOT表中的前3个槽都时预留的不给我们的代码使用，后续的槽默认都PLT0(PLT表的基地址)。
具体来讲，动态库被加载后动态加载器会给他们赋值：

GOT[0]:保存的是 .dynamic 段的地址（即动态段的起始地址）。动态链接器 ld.so 启动时会用这个地址来获取需要解析的动态信息，比如 .dynsym、.dynstr、.rel.plt 等表。

GOT[1]:保存的是 linker/loader 的解析函数地址，即 _dl_runtime_resolve（或平台相关的 resolver stub）。当你第一次调用某个外部函数时，PLT entry 会跳到 GOT[1] 指向的解析器。解析器会根据 GOT[2] + 槽地址找到具体要解析的符号，然后去加载对应的动态库函数地址。

GOT[2]:保存的是 link_map 结构体指针，这是动态链接器内部维护的 ELF 链接状态表（每个已加载的共享对象有一个 link_map 节点）。_dl_runtime_resolve 通过 GOT[2] 能够知道“当前进程有哪些已加载的库”，从而去符号查找、重定位。

GOT[6]:47030地址，会存放解析器（_dl_runtime_resolve）的地址。
另外我们也可以通过readelf查看槽位的属性：
```
$ readelf -r libanative.so

Relocation section '.rela.plt' at offset 0x11d88 contains 122 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000047038  000100000402 R_AARCH64_JUMP_SL 0000000000000000 __cxa_finalize@LIBC + 0
000000047040  000200000402 R_AARCH64_JUMP_SL 0000000000000000 __cxa_atexit@LIBC + 0
000000047048  000300000402 R_AARCH64_JUMP_SL 0000000000000000 __register_atfork@LIBC + 0
000000047050  021400000402 R_AARCH64_JUMP_SL 000000000001dc50 SayHello + 0
000000047058  000400000402 R_AARCH64_JUMP_SL 0000000000000000 printf@LIBC + 0
000000047060  018900000402 R_AARCH64_JUMP_SL 000000000001dc6c SayHelloWrapper + 0
000000047068  01aa00000402 R_AARCH64_JUMP_SL 000000000001dd64 _ZNSt6__ndk112bas[...
```
我们可以看到printf函数对应的槽信息，它的类型为：R_AARCH64_JUMP_SLOT 表示“这个 GOT 条目需要在运行时被填充成目标函数的真实地址”。

90240400 00000000 因为是小段存储的，它的实际值是：0x402490，它就是PLT0的地址。
我们使用`objdump -d -j .plt libanative.so`重新来看.plt表。
```asm
// PLT 表头（PLT0）
0000000000042490 <__cxa_finalize@plt-0x20>:
//  入栈保存 x16/IP0 与 LR（x30）
   42490:	a9bf7bf0 	stp	x16, x30, [sp, #-16]!
// x16 = .got.plt 所在页基址   
   42494:	b0000030 	adrp	x16, 47000 <_ZTVSt9bad_alloc@@Base+0x740>
// x17 = [.got.plt + 0x30（48）]（解析器入口指针）   
   42498:	f9401a11 	ldr	x17, [x16, #48]
// x16 =  .got.plt + 0x30（把“当前槽地址”传给解析器）   
   4249c:	9100c210 	add	x16, x16, #0x30
//  跳到解析器（_dl_runtime_resolve 的桩）   
   424a0:	d61f0220 	br	x17
   424a4:	d503201f 	nop
   424a8:	d503201f 	nop
   424ac:	d503201f 	nop

...

```
从上面的分析，我们得到我们在第一次调用print这个外部函数时，由于GOT表中的条目是只想PLT0，所以我们最终调用了_dl_runtime_resolve函数。 

_dl_runtime_resolve因为知道GOT表的槽（保存在x16寄存器的槽位置）从而通过.rela.plt也能知道我们调用的函数的符号信息，找到对应的外部函数的地址。然后执行，并且讲查到的地址写入到我们槽。那么下次再调用这个函数就不用再执行解析操作了。

这个机制也叫延时绑定机制(lazy binding)

_dl_runtime_resolve是一个glic中的一个汇编段，它在dl-trampoline.S文件中，它的大致实现为：
```asm

/* RELA relocatons are 3 pointers */
#define RELA_SIZE (PTR_SIZE * 3)

	.text
	.globl _dl_runtime_resolve
	.type _dl_runtime_resolve, #function
	cfi_startproc
	.align 2
_dl_runtime_resolve:
	/* AArch64 we get called with:
	   ip0		&PLTGOT[2]
	   ip1		temp(dl resolver entry point)
	   [sp, #8]	lr
	   [sp, #0]	&PLTGOT[n]
	 */

	cfi_rel_offset (lr, 8)

	/* Save arguments.  */
	stp	x8, x9, [sp, #-(80+8*16)]!
	cfi_adjust_cfa_offset (80+8*16)
	cfi_rel_offset (x8, 0)
	cfi_rel_offset (x9, 8)
```
<https://elixir.bootlin.com/glibc/glibc-2.29/source/sysdeps/aarch64/dl-trampoline.S>

但是Android使用的bionic的libc，它没有_dl_runtime_resolve函数的定义，这段的代码我并没有从代码的实现上来验证。
而且Android默认可能并没有开启延时绑定，可能实在so加载时直接进行了绑定。但是我分析的so是Android平台的，它的ELF实现有GOT表、PLT表，所以它整体的流程和linux是差不多的。

<https://android.googlesource.com/platform/bionic/>

# ELF文件格式

ELF（Executable and Linkable Format）文件是linux平台使用的一个二进制文件格式。我们平时的.o文件(可重定位文件)、so文件、可执行文件、coredump文件都是ELF文件格式。

<img src="/images/android/native_elf_brief.png" alt="elf_brief" width="720">

## 编译链接流程
如图描述了c/c++程序编译&链接的流程。
<img src="/images/android/native_c_compilation.png" alt="elf_brief" width="720">

## Ehdr 
Ehdr，定义了文件基础信息，它在整个文件的开头，它拥有文件的标识（Magic 为'E','L','F'）、ELF版本号、CPU架构类型、程序入的虚拟地址、Promgram header表的偏移、数量、Section Header表的偏移、数量、Section String表的偏移、大小等等。


```
#define EI_NIDENT (16)
// 32 bit
typedef struct
{
  unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
  Elf32_Half	e_type;			/* Object file type */
  Elf32_Half	e_machine;		/* Architecture */
  Elf32_Word	e_version;		/* Object file version */
  Elf32_Addr	e_entry;		/* Entry point virtual address */
  Elf32_Off	e_phoff;		/* Program header table file offset */
  Elf32_Off	e_shoff;		/* Section header table file offset */
  Elf32_Word	e_flags;		/* Processor-specific flags */
  Elf32_Half	e_ehsize;		/* ELF header size in bytes */
  Elf32_Half	e_phentsize;		/* Program header table entry size */
  Elf32_Half	e_phnum;		/* Program header table entry count */
  Elf32_Half	e_shentsize;		/* Section header table entry size */
  Elf32_Half	e_shnum;		/* Section header table entry count */
  Elf32_Half	e_shstrndx;		/* Section header string table index */
} Elf32_Ehdr;

// 64bit
typedef struct
{
  unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
  Elf64_Half	e_type;			/* Object file type */
  Elf64_Half	e_machine;		/* Architecture */
  Elf64_Word	e_version;		/* Object file version */
  Elf64_Addr	e_entry;		/* Entry point virtual address */
  Elf64_Off	e_phoff;		/* Program header table file offset */
  Elf64_Off	e_shoff;		/* Section header table file offset */
  Elf64_Word	e_flags;		/* Processor-specific flags */
  Elf64_Half	e_ehsize;		/* ELF header size in bytes */
  Elf64_Half	e_phentsize;		/* Program header table entry size */
  Elf64_Half	e_phnum;		/* Program header table entry count */
  Elf64_Half	e_shentsize;		/* Section header table entry size */
  Elf64_Half	e_shnum;		/* Section header table entry count */
  Elf64_Half	e_shstrndx;		/* Section header string table index */
} Elf64_Ehdr;

```
通过`readelf -h libanative.so`我们也能查看文件的头信息。

```
➜  arm64-v8a readelf -h libanative.so
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           AArch64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          1897688 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         34
  Section header string table index: 32
```
文件头信息中，多数信息是固定不变的（Margic、Version、Machine等）。所以我们重点关注下e_phoff 、 e_shoff 、 e_phnum 、 e_shnum 、 e_shstrndx这些值。这几个字段描述了3类信息：Section头、Program头、字符串。
Section和Program对这个文件的两种不同视角的描述，Section为不加载的情况下的静态分析视角、Pragma描述的是文件加载后的运行时视角。

## Shdr
Section Header, ELF文件中的代码(code)、数据(data)不化分位了一些连续的块。Section Header就是描述这些块的名称、类型、在文件中的偏移、被加载后的虚拟地址（应该被加载在什么地方）、大小等。为了节省偏移，我后续只贴出64位的数据结构。

```
typedef struct
{
  Elf64_Word	sh_name;		/* Section name (string tbl index) */
  Elf64_Word	sh_type;		/* Section type */
  Elf64_Xword	sh_flags;		/* Section flags */
  Elf64_Addr	sh_addr;		/* Section virtual addr at execution */
  Elf64_Off	sh_offset;		/* Section file offset */
  Elf64_Xword	sh_size;		/* Section size in bytes */
  Elf64_Word	sh_link;		/* Link to another section */
  Elf64_Word	sh_info;		/* Additional section information */
  Elf64_Xword	sh_addralign;		/* Section alignment */
  Elf64_Xword	sh_entsize;		/* Entry size if section holds table */
} Elf64_Shdr;
```
这里的sh_name描述了Section的名称，但它并不是一个字符串而是指向字符串表的索引。字符串表也是一个Section，它是一个字符串池，也有自己的压缩方式。
我们可以使用`readelf -S libanative.so`来查看ELF文件中的所有Section Header:
```
➜  arm64-v8a readelf -S libanative.so
There are 34 section headers, starting at offset 0x1cf4d8:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .note.androi[...] NOTE             0000000000000238  00000238
       0000000000000098  0000000000000000   A       0     0     4
  [ 2] .note.gnu.bu[...] NOTE             00000000000002d0  000002d0
       0000000000000024  0000000000000000   A       0     0     4
  [ 3] .dynsym           DYNSYM           00000000000002f8  000002f8
       0000000000003708  0000000000000018   A       7     1     8
  [ 4] .gnu.version      VERSYM           0000000000003a00  00003a00
       0000000000000496  0000000000000002   A       3     0     2
  [ 5] .gnu.version_r    VERNEED          0000000000003e98  00003e98
       0000000000000040  0000000000000000   A       7     2     4
  [ 6] .gnu.hash         GNU_HASH         0000000000003ed8  00003ed8
       0000000000000e2c  0000000000000000   A       3     0     8
  [ 7] .dynstr           STRTAB           0000000000004d04  00004d04
       000000000000484a  0000000000000000   A       0     0     1
  [ 8] .rela.dyn         RELA             0000000000009550  00009550
       0000000000008838  0000000000000018   A       3     0     8
  [ 9] .rela.plt         RELA             0000000000011d88  00011d88
       0000000000000b70  0000000000000018  AI       3    21     8
  [10] .gcc_except_table PROGBITS         00000000000128f8  000128f8
       0000000000000bc0  0000000000000000   A       0     0     4
  [11] .rodata           PROGBITS         00000000000134c0  000134c0
       000000000000364c  0000000000000000 AMS       0     0     16
  [12] .eh_frame_hdr     PROGBITS         0000000000016b0c  00016b0c
       0000000000001534  0000000000000000   A       0     0     4
  [13] .eh_frame         PROGBITS         0000000000018040  00018040
       0000000000005ba4  0000000000000000   A       0     0     8
  [14] .text             PROGBITS         000000000001dbf0  0001dbf0
       0000000000024894  0000000000000000  AX       0     0     16
  [15] .plt              PROGBITS         0000000000042490  00042490
       00000000000007c0  0000000000000000  AX       0     0     16
  [16] .data.rel.ro      PROGBITS         0000000000043c50  00042c50
       00000000000030e0  0000000000000000  WA       0     0     8
  [17] .fini_array       FINI_ARRAY       0000000000046d30  00045d30
       0000000000000010  0000000000000000  WA       0     0     8
  [18] .init_array       INIT_ARRAY       0000000000046d40  00045d40
       0000000000000008  0000000000000000  WA       0     0     8
  [19] .dynamic          DYNAMIC          0000000000046d48  00045d48
       00000000000001d0  0000000000000010  WA       7     0     8
  [20] .got              PROGBITS         0000000000046f18  00045f18
       0000000000000108  0000000000000000  WA       0     0     8
  [21] .got.plt          PROGBITS         0000000000047020  00046020
       00000000000003e8  0000000000000000  WA       0     0     8
  [22] .data             PROGBITS         0000000000048408  00046408
       0000000000000058  0000000000000000  WA       0     0     8
  [23] .bss              NOBITS           0000000000048460  00046460
       0000000000000ac0  0000000000000000  WA       0     0     16
  [24] .comment          PROGBITS         0000000000000000  00046460
       00000000000000b1  0000000000000001  MS       0     0     1
  [25] .debug_abbrev     PROGBITS         0000000000000000  00046511
       0000000000004286  0000000000000000           0     0     1
  [26] .debug_info       PROGBITS         0000000000000000  0004a797
       000000000006df30  0000000000000000           0     0     1
  [27] .debug_ranges     PROGBITS         0000000000000000  000b86c7
       0000000000022bd0  0000000000000000           0     0     1
  [28] .debug_str        PROGBITS         0000000000000000  000db297
       000000000003acd0  0000000000000001  MS       0     0     1
  [29] .debug_line       PROGBITS         0000000000000000  00115f67
       0000000000025b35  0000000000000000           0     0     1
  [30] .debug_loc        PROGBITS         0000000000000000  0013ba9c
       000000000006f3b4  0000000000000000           0     0     1
  [31] .symtab           SYMTAB           0000000000000000  001aae50
       0000000000010ed8  0000000000000018          33   2303     8
  [32] .shstrtab         STRTAB           0000000000000000  001bbd28
       0000000000000163  0000000000000000           0     0     1
  [33] .strtab           STRTAB           0000000000000000  001bbe8b
       0000000000013649  0000000000000000           0     0     1
```

重点看下字段的类型吧，

**sh_type**

    - SHT_NULL ：Section入口（未使用）
    - SHT_PROGBITS ：存放指令、常量。
    - SHT_SYMTAB ： 静态符号表
    - SHT_STRTAB： 字符串表
    - SHT_RELA： 重定位信息(有addends)
    - SHT_HASH： 哈希表
    - SHT_DYNAMIC： 动态链接信息
    - SHT_NOTE： Note
    - SHT_NOBITS： 未初始化数据
    - SHT_REL: 重定位信息(无addends)
    - SHT_DYNSYM：动态链接符号表

**sh_flags**

    - SHF_WRITE：运行时可写入。
    - SHF_ALLOC：运行时将被加载到虚拟内存。
    - SHF_EXECINSTR： 包含可执行指令。

**sh_addr**

该Section在运行时的虚拟地址。

**sh_offset**

该Section在ELF文件中的偏移。        

为了加载可执行文件，我们需要以不同的方式组织code和data，所有ELF有另一种逻辑视图（也就是我上面说的运行视图），它叫做segments，它用在运行时。相对应的上面讲的Section它用在链接时。

所有其实Section信息时，在运行时是可选的。程序通过Pragma header就可以知道如何去加载这个ELF文件到内存中。

## 常见的Section

- .init
存放程序最开始执行时的任务。（可执行文件中，这里面的程序会运行在main函数之前）。
- .fini
存放程序结束时需要运行的代码。
- .text
存放用户写的代码。
- .bss
存放没有初始化的数据，它不占用磁盘（避免空间开销）所以它的数据会在运行时被初始化为0。编译器/链接器只需要知道 .bss 这块区域需要多大。
- .data
存放程序的初始化数据。
- .rodata
存放程序的只读数据，如程序中用到的字符串。
- .plt
即 Procedure Linkage Table上文已经提到，它和GOT表一起来实现调用外部函数。
- .got.plt
用于延时绑定，上文已经提到。
- .rel.*
包含重定向的信息，_dl_runtime_resolv 借助它来寻找外部函数以及修改GOT表。
- .rela.*
同上，只是他有addend信息。
- .dynamic
动态链接的数据结构和对象，用来描述依赖库、符号表、重定位信息、初始化/析构函数等；它是动态链接工作的“说明书”。
- .init_array
存放一组函数指针，这组函数会在可执行文件初始化后立即执行。C语言中`__attribute__((constructor)`标记的函数就会粗放在这里。
- .fini_array
与.init_array相反，程序销毁的时候执行。
- .shstrtab
字符串表，存放一些节区名（.text、.data 等）
- .symtab
符号表，如存放函数、变量的符号。
- .strtab
字符串表，符号名（函数、变量、文件名等）
- .dynsym
动态链接符号表。
- .dynstr
字符串表，用于动态链接时。
- .interp
RTLD_* 是一组 动态加载标志（flags），用于 dlopen() 这个函数。如：RTLD_LAZY，懒解析：只在第一次调用符号时才解析（默认）。RTLD_NOW 立即解析：dlopen() 时就把所有未定义符号解析完（性能差但安全）。
- .rel.dyn
全局变量可重定位表。
- .rel.plt
函数可重定位表。

## Phdr
与Shdr相对，Phdr是提供segments视图(执行视图)，它给操作系统或动态链接器（dynamic-linker）提供了如何去加载这个程序的信息。与之相的Shdr可以看作是提供给程序静态链接用的。
```
typedef struct
{
  Elf64_Word	p_type;			/* Segment type */
  Elf64_Word	p_flags;		/* Segment flags */
  Elf64_Off	p_offset;		/* Segment file offset */
  Elf64_Addr	p_vaddr;		/* Segment virtual address */
  Elf64_Addr	p_paddr;		/* Segment physical address */
  Elf64_Xword	p_filesz;		/* Segment size in file */
  Elf64_Xword	p_memsz;		/* Segment size in memory */
  Elf64_Xword	p_align;		/* Segment alignment */
} Elf64_Phdr;
```
这个头信息描述了Segemnt的类型、标志、在ELF文件中的偏移、被加载的虚拟的地址、物理地址、该Segement在文件中的大小、在内存中的占用大小等。

**p_offset**

Segment在文件中的偏移。

**p_vaddr**

Segment在内存中的虚拟地址，（对于可加载段，p_vaddr % PAGE_SIZE 必须和 p_offset % PAGE_SIZE相等）。

**p_paddr**

物理地址，适用于没有MMU的操作系统，对于现代的操作系统无用。

**p_memsz**

Segment在内存中的大小。（一些Section只制定了需要在内存中分配多大空间，并不要求讲Section中的数据映射到内存，比如.bss）。

同样我们可以用readelf查看Segments Header的信息。
```
➜  arm64-v8a readelf -l libanative.so

Elf file type is DYN (Shared object file)
Entry point 0x0
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000001f8 0x00000000000001f8  R      0x8
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000042c50 0x0000000000042c50  R E    0x1000
  LOAD           0x0000000000042c50 0x0000000000043c50 0x0000000000043c50
                 0x00000000000037b8 0x00000000000037b8  RW     0x1000
  LOAD           0x0000000000046408 0x0000000000048408 0x0000000000048408
                 0x0000000000000058 0x0000000000000b18  RW     0x1000
  DYNAMIC        0x0000000000045d48 0x0000000000046d48 0x0000000000046d48
                 0x00000000000001d0 0x00000000000001d0  RW     0x8
  GNU_RELRO      0x0000000000042c50 0x0000000000043c50 0x0000000000043c50
                 0x00000000000037b8 0x00000000000043b0  R      0x1
  GNU_EH_FRAME   0x0000000000016b0c 0x0000000000016b0c 0x0000000000016b0c
                 0x0000000000001534 0x0000000000001534  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x0
  NOTE           0x0000000000000238 0x0000000000000238 0x0000000000000238
                 0x00000000000000bc 0x00000000000000bc  R      0x4

 Section to Segment mapping:
  Segment Sections...
   00
   01     .note.android.ident .note.gnu.build-id .dynsym .gnu.version .gnu.version_r .gnu.hash .dynstr .rela.dyn .rela.plt .gcc_except_table .rodata .eh_frame_hdr .eh_frame .text .plt
   02     .data.rel.ro .fini_array .init_array .dynamic .got .got.plt
   03     .data .bss
   04     .dynamic
   05     .data.rel.ro .fini_array .init_array .dynamic .got .got.plt
   06     .eh_frame_hdr
   07
   08     .note.android.ident .note.gnu.build-id
```
从上面的信息中，我们可以看到Segemnts的被映射的地址范围、标记、权限信息。以及每个Segment与Sections的对应关系。
这个so文件中

PHDR：是Program Header自身。

GNU_STACK： 表示的是程序的栈是否允许有可执⾏权限，其p_flags⼀般是RW，表示栈不允许有可执⾏权限。

LOAD: 有3个LOAD Segments他们的执行权限分别是：R、RE、RW。

DYNAMIC： 内存映射结束后，dynamic linker还会去查找外部符号（函数、变量）的地址，并填写GOT表的Slot，这个过程叫重定向，其相关的信息都在.dynamic中。

NOTE： 是一些非必要的注释信息，其中.note.gnu.build-id是动态库的build-id信息。

GNU_EH_FRAME：用于栈回溯。

# 符号与符号表

符号是一些data或code的一种符号化引用，例如一些函数、变量的引用。

它的结构如下：
```
typedef struct
{
  Elf64_Word	st_name;		/* Symbol name (string tbl index) */
  unsigned char	st_info;		/* Symbol type and binding */
  unsigned char st_other;		/* Symbol visibility */
  Elf64_Section	st_shndx;		/* Section index */
  Elf64_Addr	st_value;		/* Symbol value */
  Elf64_Xword	st_size;		/* Symbol size */
} Elf64_Sym;
```

**st_name**

符号名称。

**st_info**

符号的binding和类型，binging有：
- STB_LOCAL(局部符号（仅当前文件可见）)
- STB_GLOBAL：全局符号（可导出/可被其他文件引用）
- STB_WEAK：弱符号（可被同名全局符号覆盖）

常见的类型有：
- STT_NOTYPE： 未定义类型
- STT_FUNC： 函数类型、或其它可执行代码类型。
- STT_OBJECT：数据对象类型
- STT_SECTION： Section类型

**st_other**

符号的可见性
- STV_DEFAULT： 由符号的binding来决定。
- STV_PROTECTED：符号对其他模块可见，但不能被重定义
- STV_HIDDEN：符号对其他模块不可见，但在本模块内可用
- STV_INTERNAL：内部可见性（保留，通常不用）

## 符号表

符号存放在符号表中，其中.dynsym存放GLOBAL和WEAK类的符号、.symtab存放GLOBAL、WEAK、LOCAL符号。因此.dynsym是.symtab的一个子集。每个符号表都有与之对应的字符串池，如.shstrtab、.strtab。
使用命令`readelf --dyn-sym -W  libanative.so` 可以查看符号表中的符号信息：
```
➜  arm64-v8a readelf --dyn-sym -W  libanative.so

Symbol table '.dynsym' contains 587 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_finalize@LIBC (2)
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_atexit@LIBC (2)
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __register_atfork@LIBC (2)
     4: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@LIBC (2)
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __stack_chk_fail@LIBC (2)
     6: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND strlen@LIBC (2)
     7: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND memmove@LIBC (2)
     8: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND memcpy@LIBC (2)
     9: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND memchr@LIBC (2)
    10: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND memset@LIBC (2)
    ...
   532: 000000000001dc50    28 FUNC    GLOBAL DEFAULT   14 SayHello
   ...    
```
也可以使用`readelf -s` 将.dynsym和.symtab中的符号都打印出来：
```
➜  arm64-v8a readelf -s -W  libanative.so

Symbol table '.dynsym' contains 587 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_finalize@LIBC (2)
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_atexit@LIBC (2)
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __register_atfork@LIBC (2)
     4: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@LIBC (2)
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __stack_chk_fail@LIBC (2)
     6: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND strlen@LIBC (2)
...
   292: 00000000000248a0     0 NOTYPE  LOCAL  DEFAULT   14 $x.170
   293: 0000000000024a44     0 NOTYPE  LOCAL  DEFAULT   14 $x.171
   294: 0000000000024b84     0 NOTYPE  LOCAL  DEFAULT   14 $x.172
   295: 0000000000024cc4     0 NOTYPE  LOCAL  DEFAULT   14 $x.173
   296: 0000000000024e04     0 NOTYPE  LOCAL  DEFAULT   14 $x.174
   297: 0000000000012f74     0 NOTYPE  LOCAL  DEFAULT   10 GCC_except_table129
   298: 0000000000012f74     0 NOTYPE  LOCAL  DEFAULT   10 $d.175
   299: 0000000000024fb4     0 NOTYPE  LOCAL  DEFAULT   14 $x.176
   300: 0000000000012f8c     0 NOTYPE  LOCAL  DEFAULT   10 GCC_except_table130
   301: 0000000000012f8c     0 NOTYPE  LOCAL  DEFAULT   10 $d.177
   302: 0000000000025164     0 NOTYPE  LOCAL  DEFAULT   14 $x.178
   303: 0000000000012fa4     0 NOTYPE  LOCAL  DEFAULT   10 GCC_except_table131
   304: 0000000000012fa4     0 NOTYPE  LOCAL  DEFAULT   10 $d.179
   305: 00000000000148a0     0 NOTYPE  LOCAL  DEFAULT   11 $d.180
   306: 000000000002530c     0 NOTYPE  LOCAL  DEFAULT   14 $x.181
   307: 0000000000012fbc     0 NOTYPE  LOCAL  DEFAULT   10 GCC_except_table132
   308: 0000000000012fbc     0 NOTYPE  LOCAL  DEFAULT   10 $d.182
   309: 0000000000025510     0 NOTYPE  LOCAL  DEFAULT   14 $x.183
   310: 0000000000012fec     0 NOTYPE  LOCAL  DEFAULT   10 GCC_except_table133
   311: 0000000000012fec     0 NOTYPE  LOCAL  DEFAULT   10 $d.184
   312: 0000000000025710     0 NOTYPE  LOCAL  DEFAULT   14 $x.185
   313: 000000000001301c     0 NOTYPE  LOCAL  DEFAULT   10 GCC_except_table134
   314: 000000000001301c     0 NOTYPE  LOCAL  DEFAULT   10 $d.186
   315: 0000000000025908     0 NOTYPE  LOCAL  DEFAULT   14 $x.187
   316: 0000000000025940     0 NOTYPE  LOCAL  DEFAULT   14 $x.188     
```
可以看到.symtab中的符号的数量远比.dynsym 中的符号数量多的多，而多出来的符号只是用于调试，不会影响程序的正常运行。因此一般的release的动态库中，都会把 .symtab 给裁剪掉。所以局部符号是外部不可见的。

## Hash表
我们知道dlsym等接口可以通过符号名查找符号的地址，这是如何实现的呢？

最简单的，我们可以通过Section Headers等信息，获取 .dynsym 或者 .symtab 的地址，然后遍历一下符号表中的所有符号，并匹配符号名即可。显然这种方式效率太低。

为了提升程序的加载速度和运行效率，ELF采用了hash查找方式，传入符号名，就可以通过hash算法算得符号在符号表中得index。

有两种hash表，分别为 .hash 和 .gnu_hash ，可以只存在一种，也可以同时存在两种。

```
➜  arm64-v8a readelf -S  libanative.so
There are 34 section headers, starting at offset 0x1cf4d8:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .note.androi[...] NOTE             0000000000000238  00000238
       0000000000000098  0000000000000000   A       0     0     4
  [ 2] .note.gnu.bu[...] NOTE             00000000000002d0  000002d0
       0000000000000024  0000000000000000   A       0     0     4
  [ 3] .dynsym           DYNSYM           00000000000002f8  000002f8
       0000000000003708  0000000000000018   A       7     1     8
  [ 4] .gnu.version      VERSYM           0000000000003a00  00003a00
       0000000000000496  0000000000000002   A       3     0     2
  [ 5] .gnu.version_r    VERNEED          0000000000003e98  00003e98
       0000000000000040  0000000000000000   A       7     2     4
// Hash表       
  [ 6] .gnu.hash         GNU_HASH         0000000000003ed8  00003ed8
       0000000000000e2c  0000000000000000   A       3     0     8
  [ 7] .dynstr           STRTAB           0000000000004d04  00004d04
       000000000000484a  0000000000000000   A       0     0     1
  [ 8] .rela.dyn         RELA             0000000000009550  00009550
       0000000000008838  0000000000000018   A       3     0     8
  [ 9] .rela.plt         RELA             0000000000011d88  00011d88
```
已GNU hash为例，大致算法实现如下：

- 来自bionic libc:

```c
// linker_soinfo.cpp

const ElfW(Sym)* soinfo::find_symbol_by_name(SymbolName& symbol_name,
                                             const version_info* vi) const {
  return is_gnu_hash() ? gnu_lookup(symbol_name, vi) : elf_lookup(symbol_name, vi);
}

const ElfW(Sym)* soinfo::gnu_lookup(SymbolName& symbol_name, const version_info* vi) const {
     // 先计算符号的HASH
  const uint32_t hash = symbol_name.gnu_hash();

  constexpr uint32_t kBloomMaskBits = sizeof(ElfW(Addr)) * 8;
  const uint32_t word_num = (hash / kBloomMaskBits) & gnu_maskwords_;
  const ElfW(Addr) bloom_word = gnu_bloom_filter_[word_num];
  const uint32_t h1 = hash % kBloomMaskBits;
  const uint32_t h2 = (hash >> gnu_shift2_) % kBloomMaskBits;

  LD_DEBUG(lookup, "SEARCH %s in %s@%p (gnu)",
           symbol_name.get_name(), get_realpath(), reinterpret_cast<void*>(base));

  // test against bloom filter
  if ((1 & (bloom_word >> h1) & (bloom_word >> h2)) == 0) {
    return nullptr;
  }

  // 先bloom筛选，加速
  // bloom test says "probably yes"...
  uint32_t n = gnu_bucket_[hash % gnu_nbucket_];

  if (n == 0) {
    return nullptr;
  }

  const ElfW(Versym) verneed = find_verdef_version_index(this, vi);
  const ElfW(Versym)* versym = get_versym_table();

  do {
    ElfW(Sym)* s = symtab_ + n;
    // 计算出桶
    if (((gnu_chain_[n] ^ hash) >> 1) == 0 &&
        check_symbol_version(versym, n, verneed) &&
        strcmp(get_string(s->st_name), symbol_name.get_name()) == 0 &&
        is_symbol_global_and_defined(this, s)) {
      return symtab_ + n;
    }
    // 遍历链表
  } while ((gnu_chain_[n++] & 1) == 0);

  return nullptr;
}

// 计算HASH, HASH算法如下：
uint32_t SymbolName::gnu_hash() {
  if (!has_gnu_hash_) {
    gnu_hash_ = calculate_gnu_hash(name_).first;
    has_gnu_hash_ = true;
  }

  return gnu_hash_;
}

static inline std::pair<uint32_t, uint32_t> calculate_gnu_hash(const char* name) {
#if USE_GNU_HASH_NEON
  return calculate_gnu_hash_neon(name);
#else
  return calculate_gnu_hash_simple(name);
#endif
}
__attribute__((unused))
static std::pair<uint32_t, uint32_t> calculate_gnu_hash_simple(const char* name) {
  uint32_t h = 5381;
  const uint8_t* name_bytes = reinterpret_cast<const uint8_t*>(name);
  #pragma unroll 8
  while (*name_bytes != 0) {
    h += (h << 5) + *name_bytes++; // h*33 + c = h + h * 32 + c = h + h << 5 + c
  }
  return { h, reinterpret_cast<const char*>(name_bytes) - name };
}

```

需要注意的是，只有 .dynsym 有hash表， .symtab 是没有hash表只能遍历查找。


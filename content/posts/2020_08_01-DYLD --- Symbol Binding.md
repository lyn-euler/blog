---
title: "DYLD-符号绑定"
description: "符号绑定"
icon: "flag"
---

系统环境
```
> ProductName:	macOS
> ProductVersion:	11.0
> BuildVersion:	20A5343i
> Darwin Kernel Version 20.0.0: Thu Jul 30 22:49:28 PDT 2020; root:xnu-7195.0.0.141.5~1/RELEASE_X86_64 x86_64
```

## 引子
在软件工程的鸿蒙时代, 一个程序的所有源码都是在一个文件上的, 随着工程的扩大和代码量增加, 多人协同、代码复用、维护、编译时间等问题就日益突出了. 为了解决这些问题也就有了静态链接.
但是静态链接也还是存在着很多问题.比如典型的:空间浪费(包括内存和硬盘)、程序更新&部署不方便等.

动态链接的出现使得程序可以在运行时才进行链接, 很好的解决了上述问题. 动态链接的好处有很多，其中包括:
1. **代码重用**
  常用的代码可以被提取到一个库中，然后共享使用。
2. **易于更新**
  只要符号大体相同(API接口不变)，驻留在库中的代码可以很容易地更新，库也可以被替换。
3. **减少磁盘使用量**
  因为动态库中的代码并不会在每一个使用它的二进制中都要包含, 只有在程序运行时才会被链接到可执行文件中, 而不同的程序可以共享同一个动态库代码。
4. **减少RAM的使用**
  这是它最重要的优势。一个库的副本可能会被mmap-ed到所有的进程中，而在RAM中实际只占用一次。库代码通常被标记为r-x(只读可执行)，因此同一个物理副本被许多进程隐含地共享。

注: 同时动态链接的运行时特性赋予了我们很重要的能力--函数拦截、审计和HOOK。dyld允许通过环境变量--DYLD_INSERT_LIBRARIES(类似于ld的LD_PRELOAD)和DYLD_LIBRARY_PATH(类似于ld的LD_LIBRARY_PATH)--以及它的函数插值机制来实现HOOK和拦截; 此外你还可以直接通过修改符号表等来实现类似功能.后续可以单独开篇来讲解这方面的应用.

## 准备
因为整个符号绑定的流程比涉及面比较广, 在开始具体流程分析之前, 可能需要掌握一些准备知识和了解专业术语.
### 基本概念 
- [**Mach-O**](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/MachOTopics/0-Introduction/introduction.html): Mach-O为Mach Object文件格式的缩写，它是一种用于可执行文件，目标代码， 动态库，内核转储的文件格式, 包括文中提及的镜像也是一种Mach-O文件.

- **dyld**: Apple 生态操作系统（macOS、iOS）的动态链接器.
- **dylib**: 动态库.
  类似于Unix中的Shared Object。一个 MH_DYLIB (0x6)类型的Mach-O对象，通过`LC_LOAD_DYLIB` Mach-O命令或 `dlopen` API加载到其他可执行文件中。
- **符号(symbol)**: Mach-O文件中的一个变量或函数，在该文件之外可能可见，也可能不可见。
- **Binding(绑定)**: 将一个符号引用和它在内存中的地址连接起来。绑定可以是加载时的，懒惰的（延迟）或（缺失/可覆盖）。这些都可以在编译时控制：ld的 `-bind_at_load` 指定了加载时绑定，`__attribute((weak_import))`指定了弱符号。还有一个 ld 的 `-prebind switch`可以将库预绑定到固定地址.

## 启动流程
iOS中冷启动的大致流程如图(dyld2):
![启动](https://raw.githubusercontent.com/lyn-euler/assets/master/img/dyld-lanch.png)
- 内核fork并创建程序进程
- 加载程序和依赖的动态库
- Rebase
- Binding
- ObjC runtime 初始化
- 其他初始化代码

dyld3中略有不同增加了缓存以及将解析Mach-O和查找依赖库符号的部分放到了out-of-process进行, 但总体上的流程没有太大变化.
![](https://raw.githubusercontent.com/lyn-euler/assets/master/img/dyld3.png)

## 符号Binding
首先, 我们来创建一个简单的工程看下, 添加以下的代码.
```Objective-C
- (void)viewDidLoad {
    NSString *name = UIApplicationDidFinishLaunchingNotification;
    printf("%s", name.UTF8String);
}
```
编译后, 可以通过otool工具查看其汇编代码:
```shell
$ otool -tv dyld-test

dyld-test:
...
(__TEXT,__text) section
-[ViewController viewDidLoad]:
0000000100001c60	pushq	%rbp
0000000100001c61	movq	%rsp, %rbp
0000000100001c64	subq	$0x20, %rsp
0000000100001c68	movq	0x2391(%rip), %rax #_UIApplicationDidFinishLaunchingNotification
0000000100001c6f	movq	%rdi, -0x8(%rbp)
0000000100001c73	movq	%rsi, -0x10(%rbp)
0000000100001c77	movq	(%rax), %rax
0000000100001c7a	movq	%rax, %rdi
0000000100001c7d	callq	*0x2395(%rip)        
0000000100001c83	movq	%rax, -0x18(%rbp)
0000000100001c87	movq	-0x18(%rbp), %rax
0000000100001c8b	movq	%rax, %rdi
0000000100001c8e	callq	0x100002382
0000000100001c93	movq	0x7716(%rip), %rsi
0000000100001c9a	movq	%rax, %rdi
0000000100001c9d	callq	*0x2365(%rip)
0000000100001ca3	leaq	0x7ac(%rip), %rdi
0000000100001caa	movq	%rax, %rsi
0000000100001cad	movb	$0x0, %al
0000000100001caf	callq	0x100002394 # call _print_stub
0000000100001cb4	xorl	%ecx, %ecx
0000000100001cb6	movl	%ecx, %esi
0000000100001cb8	leaq	-0x18(%rbp), %rdx
0000000100001cbc	movq	%rdx, %rdi
0000000100001cbf	movl	%eax, -0x1c(%rbp)
0000000100001cc2	callq	0x10000238e
0000000100001cc7	addq	$0x20, %rsp
0000000100001ccb	popq	%rbp
0000000100001ccc	retq
0000000100001ccd	nop
....
```
关注上面有备注的两行, 这两行指令分别引用了`_UIApplicationDidFinishLaunchingNotification`和`_print`符号. 那么按RIP-relative 寻址方式我们可以计算得到
```shell
> 0x100001c6f + 0x2391 = 0x100004000 = _UIApplicationDidFinishLaunchingNotification 目标虚拟地址
> 0x100002394 = 0x100001cb4 + 0x6d = _print函数目标虚拟地址
```
那么`0x100001c6f`和`0x100002394`在Mach-O中哪里呢? 通过`otool -s`或者MachOView我们可以找到这个两个地址分别在`__DATA_CONST,__got`和`__TEXT,__stubs` 的section中:
```shell
$ otool dyld-test -s __DATA_CONST __got
dyld-test:
Contents of (__DATA_CONST,__got) section
0000000100004000	00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0000000100004010	00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0000000100004020	00 00 00 00 00 00 00 00

$ otool -v dyld-test -s __TEXT __stubs
dyld-test:
Contents of (__TEXT,__stubs) section
0000000100002334	jmpq	*0x5cc6(%rip)
000000010000233a	jmpq	*0x5cc8(%rip)
0000000100002340	jmpq	*0x5cca(%rip)
0000000100002346	jmpq	*0x5ccc(%rip)
000000010000234c	jmpq	*0x5cce(%rip)
0000000100002352	jmpq	*0x5cd0(%rip)
0000000100002358	jmpq	*0x5cd2(%rip)
000000010000235e	jmpq	*0x5cd4(%rip)
0000000100002364	jmpq	*0x5cd6(%rip)
000000010000236a	jmpq	*0x5cd8(%rip)
0000000100002370	jmpq	*0x5cda(%rip)
0000000100002376	jmpq	*0x5cdc(%rip)
000000010000237c	jmpq	*0x5cde(%rip)
0000000100002382	jmpq	*0x5ce0(%rip)
0000000100002388	jmpq	*0x5ce2(%rip)
000000010000238e	jmpq	*0x5ce4(%rip)
0000000100002394	jmpq	*0x5ce6(%rip)
```
实际上, Mach-O中的 `__TEXT` section对外部符号的引用地址也就是指向这两个section.
- `__DATA_CONST,__got`:  no-lazy symbol pointer, 比如全局变量/常量、dyld中的函数符号.
- `__TEXT,__stubs`: lazy symbol stub, 比如其他动态库中的函数符号.

### section(__DATA_CONST __got)
可以把`section(__DATA_CONST __got)`看做一个表, 每个条目是一个地址值.但是上面的结果我们可以看到`0x100004000`对应的内容都是0.所以dyld需要在运行时用实际的符号地址替换.这也是该section被定义在`__DATA` segment中的原因了.
那么dyld又是如何获取每个条目的符号信息的呢? 按我的理解至少要该条目的符号名称以及所在的库.
我们知道Mach-O中每个segment都是由`LC_SEGMENT`定义的, 该命令结束后的阐述描述了section的信息对应的数据结构是:
```c
struct section_64 { /* for 64-bit architectures */
	char		sectname[16];	/* name of this section */
	char		segname[16];	/* segment this section goes in */
	uint64_t	addr;		/* memory address of this section */
	uint64_t	size;		/* size in bytes of this section */
	uint32_t	offset;		/* file offset of this section */
	uint32_t	align;		/* section alignment (power of 2) */
	uint32_t	reloff;		/* file offset of relocation entries */
	uint32_t	nreloc;		/* number of relocation entries */
	uint32_t	flags;		/* flags (section type and attributes)*/
	uint32_t	reserved1;	/* reserved (for offset or index) */
	uint32_t	reserved2;	/* reserved (for count or sizeof) */
	uint32_t	reserved3;	/* reserved */
};
```
对于`__got`、`__stubs`、`__la_symbol_ptr`这几个 section，该结构体reserved1描述了对应 list 中条目在 [indirect symbol table]
() (下文会讲, 这里就把它看成一个表就好了)中的index.

还是用上面的程序做例子
![](https://raw.githubusercontent.com/lyn-euler/assets/master/img/__got.png)
那么`__got`符号对应indirect symbol table中第17个条目.

先告一段落, 我们重新回头来看lazy symbol的情况.
### section(__TEXT,__stubs)
不同于`__DATA_CONST, __got`, `__stubs`位于`__TEXT`段, 所以该内容是不能运行时修改的. 该section的每个表项都是一段汇编代码, 称为`符号桩`. 比如上面`_printf`对应的项为:

```armasm
0000000100002394	jmpq	*0x5ce6(%rip) 
# 间接寻址, 0x100002394 + 0x5ce6 = 0x100008080
```


### section(__DATA,__la_symbol_ptr)
那么上面 0x100008080 又在哪部分呢? 
```shell 
$ otool -v dyld-test -s __DATA __la_symbol_ptr
dyld-test:
Contents of (__DATA,__la_symbol_ptr) section
Unknown section type (0x00000007)
0000000100008000	ac 23 00 00 01 00 00 00 4c 24 00 00 01 00 00 00
0000000100008010	06 24 00 00 01 00 00 00 10 24 00 00 01 00 00 00
0000000100008020	1a 24 00 00 01 00 00 00 24 24 00 00 01 00 00 00
0000000100008030	2e 24 00 00 01 00 00 00 38 24 00 00 01 00 00 00
0000000100008040	b6 23 00 00 01 00 00 00 c0 23 00 00 01 00 00 00
0000000100008050	ca 23 00 00 01 00 00 00 d4 23 00 00 01 00 00 00
0000000100008060	de 23 00 00 01 00 00 00 e8 23 00 00 01 00 00 00
0000000100008070	f2 23 00 00 01 00 00 00 fc 23 00 00 01 00 00 00
0000000100008080	42 24 00 00 01 00 00 00
```
可以看到该地址位于`(__DATA,__la_symbol_ptr)`中,也就是说符号桩jump的目标地址是0x100002442(小端).

### section(__TEXT, __stub_helper)
继续查找该地址, 发现位于`__TEXT, __stub_helper`中
```shell
$ otool -v dyld-test -s __TEXT __stub_helper
dyld-test:
Contents of (__TEXT,__stub_helper) section
000000010000239c	leaq	0x712d(%rip), %r11
00000001000023a3	pushq	%r11
00000001000023a5	jmpq	*0x1c75(%rip)
00000001000023ab	nop
....
0000000100002438	pushq	$0xd2
000000010000243d	jmp	0x10000239c
0000000100002442	pushq	$0x1ca
0000000100002447	jmp	0x10000239c
000000010000244c	pushq	$0x19
0000000100002451	jmp	0x10000239c
```
可以看到这段代码的执行流程 0000000100002442 -> ..-> 000000010000239c -> ..-> 00000001000023a5. 最终执行:

```armasm
00000001000023a5 jmpq	*0x1c75(%rip)
00000001000023ab	nop
# 0x1c75 + 0x1000023ab = 0x100004020
```
继续查找`0x100004020`发现位于`__DATA_CONST,__got`中, 跟上面分析no-lazy symbol一样, 最终indirect symbol table中. 其实所有的 lazy 函数符号在dyld初始化后, 都是指向dyld_stub_binder函数.

### __LINKEDIT
从OS X 10.5或10.6开始，苹果决定在Mach-O文件中实现一个特殊的段，供DYLD使用。这个段，传统上称为`__LINKEDIT`，由DYLD在链接和绑定符号的过程中使用的信息组成。上面提到的`indirect symbol table`就位于这一部分.
LC_SEGMENT定义了该segment, DYLD依靠一个特殊的加载命令LC_DYLD_INFO_ONLY来作为段的 "目录"进一步对动态loader info做了细分.
下面是用`jtool`输出的格式(类似pagestuff).
```shell
$ ./jtool --pages /bin/ls -arch x86_64
0x0-0x8000	__TEXT
	0x3d84-0x73ce	__TEXT.__text
	0x73ce-0x759c	__TEXT.__stubs
	0x759c-0x78ae	__TEXT.__stub_helper
	0x78b0-0x7a93	__TEXT.__const
	0x7a93-0x7f59	__TEXT.__cstring
	0x7f5c-0x7ff8	__TEXT.__unwind_info
0x8000-0xc000	__DATA
	0x8000-0x8008	__DATA.__nl_symbol_ptr
	0x8008-0x8038	__DATA.__got
	0x8038-0x82a0	__DATA.__la_symbol_ptr
	0x82a0-0x84c8	__DATA.__const
	0x84d0-0x84f8	__DATA.__data
0xc000-0xe890	__LINKEDIT
	0xc000-0xc018	Rebase Info     (opcodes)
	0xc018-0xc080	Binding Info    (opcodes)
	0xc080-0xc5f0	Lazy Bind Info  (opcodes)
	0xc5f0-0xc610	Exports
	0xc610-0xc658	Function Starts
	0xc658-0xc680	Data In Code
	0xc680-0xcbd0	Symbol Table
	0xcbd0-0xce54	Indirect Symbol Table
	0xce58-0xd230	String Table
	0xd230-0xe890	Code Signature
```
__LINKEDIT的一般布局如下, DYLD大量使用了ULEB128编码(按Jonathan Levin的说法这是一种粗陋的编码方法。), 底层实现者将广泛熟悉该编码，该编码在DWARF和其他二进制相关格式中也有使用.
![__LINKEDIT](https://raw.githubusercontent.com/lyn-euler/assets/master/img/__LINKEDIT.jpg)

### DYLD Opcodes
DYLD使用一种特殊的编码--由各种`opcode`组成--来存储和加载符号绑定信息。这些操作码用于填充LC_DYLD_INFO命令所指向的rebase信息和binding表。有两种类型的操作码。Rebasing操作码和Binding操作码。
Binding操作码（用于lazy和non-lazy符号）被定义为BIND_xxx常数:

| opcode | val | desc |
| --- | --- | --- |
| DONE | 0x00 | 将当前记录push到导入栈, 并清零记录状态  |
| SET_DYLIB_ORDINAL_IMM | 0x10 | 设置dylib ordinal为immediate(低4-bits). <br/>用于0-15的ordinal number  |
| SET_DYLIB_ORDINAL_ULEB | 0x20 | 以 ULEB128 编码 dylib ordinal 。<br/> 用于16+的ordinal number |
| SET_DYLIB_SPECIAL_IMM | 0x30 | 设置dylib序数，以0或负数为immediate.<br/>该值为sign扩展。<br/> 目前已知的数值是：<br/> BIND_SPECIAL_DYLIB_SELF (0) <br/> BIND_SPECIAL_DYLIB_MAIN_EXECUTABLE(-1) <br/> BIND_SEPCIAL_DYLIB_FLAT_LOOKUP(-2) |
| SET_SYMBOL_TRAILING<br/>_FLAGS_IMM | 0x40 | 设置符号名（以NULL结尾的char[]） <br/>在immediate值中的标志可以是<br/>BIND_SYMBOL_FLAGS_WEAK_IMPORT(0) 或 <br/>BIND_SYMBOL_FLAGS_NON_WEAK_DEFINITION(8) |
| SET_TYPE_IMM | 0x50 | 设置符号类型. <br/>TYPE_POINTER<br/>TYPE_TEXT_ABSOLUTE32<br/>TYPE_TEXT_PCREL32 |
| SET_ADDEND_SLEG | 0x60 | 以SLEB128编码设置addend字段 |
| SET_SEGMENT_AND_OFFSET_ULEB | 0x70 | 将Segment设置为immediate value, 地址以SLEB128编码. |
| ADD_ADDR_ULEB | 0x80 | 将address字段以SLEB128编码. |
| DO_BIND | 0x90 | 对当前表行进行绑定 |
| DO_BIND_ADD_ADDR_ULEB | 0xA0 | 进行绑定, 并以ULEB128对地址进行编码 |
| DO_BIND_ADD_ADDR_IMM_SCALED | 0xB0 | 进行绑定, 并以使用scaling添加immediate (lower 4-bits)  |
| DO_BIND_ADD_ADDR_ULEB _TIMES_SKIPPING_ULEB | 0xC0 | 对多个符号进行绑定,并跳过几个字节.罕见 |

操作码填充了绑定表中行项的各个列，基本格式是:

> `opcode` + [`immediate|ULEB128整数|字符数组`]

每行以DO_BIND结束。每一行默认携带前一行的值，因此只有在两个符号之间列值发生变化时才会指定一个操作码。这样就可以对表进行压缩。
举个例子🌰:
```shell
0x0000 BIND_OPCODE_SET_DYLIB_ORDINAL_IMM(3)                                   # 设置 DYLIB to #3 (第三个LC_LOAD_DYLIB)
0x0001 BIND_OPCODE_SET_SYMBOL_TRAILING_FLAGS_IMM(0x00, __DefaultRuneLocale)   # 设置符号名为 __DefaultRuneLocale
0x0016 BIND_OPCODE_SET_TYPE_IMM(1)                                            # 设置类型为 pointer
0x0017 BIND_OPCODE_SET_SEGMENT_AND_OFFSET_ULEB(0x02, 0x00000000)              # 设置segement #2 (__DATA)
0x0019 BIND_OPCODE_DO_BIND()                                                  # 第一行结束

#
# 第二行会完全继承第一行的值, 除了覆盖symbol name:
#

0x001A BIND_OPCODE_SET_SYMBOL_TRAILING_FLAGS_IMM(0x00, ___stack_chk_guard)    # 设置符号名 __stack_chk_guard
0x002E BIND_OPCODE_DO_BIND()

#
# 同样的第三行也只用覆盖符号名:

0x002F BIND_OPCODE_SET_SYMBOL_TRAILING_FLAGS_IMM(0x00, ___stderrp)
0x003B BIND_OPCODE_DO_BIND()
```

这些操作码是由上面提到的`dyld_stub_binder`使用的，我们后面会讨论。但在此之前，我们必须再做一个转折来解释Mach-O中的两种类型的符号表。

### Symbol Tables
在Mach-O文件中的符号表由LC_SYMTAB命令描述的。链接器是通过 LC_SYMTAB 这个 load command 找到 symbol table, LC_SYMTAB 对应的 command 结构体如下:
```c
struct symtab_command {
    uint32_t cmd;     /* LC_SYMTAB */
    uint32_t cmdsize; /* sizeof(struct symtab_command) */
    uint32_t symoff;  /* symbol table offset */
    uint32_t nsyms;   /* number of symbol table entries */
    uint32_t stroff;  /* string table offset */
    uint32_t strsize; /* string table size in bytes */
};
```
`symoff`和`nsyms`指示了符号表的位置和条目，`stroff`和`strsize`指示了字符串表的位置和长度.
每个 symbol entry 长度是固定的，其结构由内核定义:

```c
struct nlist_64 {
    union {
        uint32_t  n_strx; /* index into the string table */
    } n_un;
    uint8_t n_type;        /* type flag, see below */
    uint8_t n_sect;        /* section number or NO_SECT */
    uint16_t n_desc;       /* see <mach-o/stab.h> */
    uint64_t n_value;      /* value of this symbol (or stab offset) */
};
```

- `n_un`: 符号的名字（在一个 Mach-O 文件里，具有唯一性）
- `n_sect`: 符号所在的 section index（内部符号有效值从 1 开始，最大为 255）
- `n_value`: 符号的地址值（在链接过程中，会随着其 section 发生变化）

### Indirect Symbol Table
indirect symbol table 由LC_DYSYMTAB定义，后者的参数类型是一个dysymtab_command结构体:
```c
struct dysymtab_command {
    uint32_t cmd;	/* LC_DYSYMTAB */
    uint32_t cmdsize;	/* sizeof(struct dysymtab_command) */

    uint32_t ilocalsym;	/* index to local symbols */
    uint32_t nlocalsym;	/* number of local symbols */

    uint32_t iextdefsym;/* index to externally defined symbols */
    uint32_t nextdefsym;/* number of externally defined symbols */

    uint32_t iundefsym;	/* index to undefined symbols */
    uint32_t nundefsym;	/* number of undefined symbols */

    uint32_t tocoff;	/* file offset to table of contents */
    uint32_t ntoc;	/* number of entries in table of contents */
    uint32_t modtaboff;	/* file offset to module table */
    uint32_t nmodtab;	/* number of module table entries */

    uint32_t extrefsymoff;	/* offset to referenced symbol table */
    uint32_t nextrefsyms;	/* number of referenced symbol table entries */

   uint32_t indirectsymoff; /* file offset to the indirect symbol table */
    uint32_t nindirectsyms;  /* number of indirect symbol table entries */

   
    uint32_t extreloff;	/* offset to external relocation entries */
    uint32_t nextrel;	/* number of external relocation entries */
    uint32_t locreloff;	/* offset to local relocation entries */
    uint32_t nlocrel;	/* number of local relocation entries */

};	

```
本质上，indirect符号表是 index 数组，即每个条目的内容是一个 index 值，`indirectsymoff`和`nindirectsyms`这两个字段定义了 indirect symbol table 的位置信息，每一个条目是一个 4 bytes 的 index 值.

### dyld_stub_binder
至此我们就可以通过indirect符号表找到符号表中的符号了. 但是 lazy binding 最终调用的是`dyld_stub_binder`, 这个函数具体做了什么呢?
```ruby
dyld_stub_binder:
	stp		fp, lr, [sp, #-16]!
	mov		fp, sp
	sub		sp, sp, #240
	stp		x0,x1, [fp, #-16]	; x0-x7 are int parameter registers
	stp		x2,x3, [fp, #-32]
	stp		x4,x5, [fp, #-48]
	stp		x6,x7, [fp, #-64]
	stp		x8,x9, [fp, #-80]	; x8 is used for struct returns
	stp		q0,q1, [fp, #-128]	; q0-q7 are vector/fp parameter registers
	stp		q2,q3, [fp, #-160]
	stp		q4,q5, [fp, #-192]
	stp		q6,q7, [fp, #-224]

	ldr		x0, [fp, #24]	; move address ImageLoader cache to 1st parameter
	ldr		x1, [fp, #16]	; move lazy info offset 2nd parameter
	; call dyld::fastBindLazySymbol(loadercache, lazyinfo)
	bl		__Z21_dyld_fast_stub_entryPvl
	mov		x16,x0			; save target function address in lr
	
	; restore parameter registers
	ldp		x0,x1, [fp, #-16]
	ldp		x2,x3, [fp, #-32]
	ldp		x4,x5, [fp, #-48]
	ldp		x6,x7, [fp, #-64]
	ldp		x8,x9, [fp, #-80]
	ldp		q0,q1, [fp, #-128]
	ldp		q2,q3, [fp, #-160]
	ldp		q4,q5, [fp, #-192]
	ldp		q6,q7, [fp, #-224]
	
	mov		sp, fp
	ldp		fp, lr, [sp], #16
	add		sp, sp, #16	; remove meta-parameters
```
可以看到最终通过调用dyld::fastBindLazySymbol, 该函数内部通过ImageLoaderMachO::getLazyBindingInfo根据opcode找到符号的真实地址, 并将该地址写入`__la_symbol_ptr`条目, 最后跳转符号地址.

## 总结
如图
![symbol Binding](https://github.com/lyn-euler/assets/raw/master/img/symbol%20Binding.png)


## 参考
- [Mach-O Binaries](http://www.m4b.io/reverse/engineering/mach/binaries/2015/03/29/mach-binaries.html)
- [Mach-O 与动态链接](https://zhangbuhuai.com/post/macho-dynamic-link.html)
- [链接、装载与库 --- 动态链接](https://markrepo.github.io/kernel/2018/08/19/dynamic-link/)
- [DYLD Detailed](http://www.newosxbook.com/articles/DYLD.html)
- 《程序员的自我修养》
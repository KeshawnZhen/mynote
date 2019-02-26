[toc]

# 目标文件


## 1.目标文件格式
目标文件和可执行文件格式几乎是一样的。此外，动态链接库、静态链接库等文件都按照可执行文件格式存储。它们在Windows下按照PE-COFF格式存储，在LInux下按照ELF格式存储。

## 2.目标文件里面有什么
ELF文件的开头是一个文件头，它描述了整个文件的文件属性，包括文件是否可执行、是静态链接还是动态链接及入口地址、目标硬件、目标操作系统等信息。文件头还包括一个段表，用来描述文件中各个段的数组。

总的来说，程序代码被编译以后主要分成两种段：程序指令和程序数据。代码段属于程序指令，而数据段和.bss段属于程序数据。
## 3.探究一个目标文件
``` SimpleSection.c
/*
SimpleSection.c

Linux:
        gcc -c SimpleSection.c

Windows:
        cl SimpleSection.c /c /Za
*/

int printf( const char* format, ... );

int global_init_var = 84;
int global_uninit_var;

void func1( int i )
{
        pirntf("%d\n", i );
}

int main(void)
{
        static int static_var = 85;
        static int static_var2;
        int a = 1;
        int b;

        func1( static_var + static_var2 + a + b );

        return a;
}
```
使用gcc编译这个文件。
> gcc -c SimpleSection.c
（参数c表示只编译不连接）

使用objdump来查看SimpleSection.o这个目标文件的内部结构

> objdump -h SimpleSection.o
-h是把ELF文件的各个段的基本信息打印出来
```
SimpleSection.o：     文件格式 elf64-x86-64

节：
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00000055  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000008  0000000000000000  0000000000000000  00000098  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000004  0000000000000000  0000000000000000  000000a0  2**2
                  ALLOC
  3 .rodata       00000004  0000000000000000  0000000000000000  000000a0  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .comment      00000035  0000000000000000  0000000000000000  000000a4  2**0
                  CONTENTS, READONLY
  5 .note.GNU-stack 00000000  0000000000000000  0000000000000000  000000d9  2**0
                  CONTENTS, READONLY
  6 .eh_frame     00000058  0000000000000000  0000000000000000  000000e0  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
```
文件SimpleSection.o中共有7个段。
`.text`:保存机器代码段
`.data`：保存已初始化的全局变量和局部变量 
`.bss`：未初始化的全局变量和局部静态变量 
`.rodata`：只读数据段 
`.comment`：注释信息段 
`.note.GNU-stack`：堆栈提示段 
`.eh_frame`：异常处理信息段
每个段的属性有，该段的大小，该段在文件中的偏移位置。第2行中的“CONTENTS”表示该段在文件中，没有“CONTENTS”标识表示该段在不在文件中，如.bss段。"ALLOC"表示是否字节对齐。

size命令可以用来查看ELF文件的代码段、数据段和BSS段的长度
> size SimpleSection.o

>   text	   data	    bss	    dec	    hex	filename
>    177	      8	      4	    189	     bd	SimpleSection.o


### 3.1 代码段
使用objdump提取出SimpleSection.o中代码段的内容
> objdump -s -d SimpleSection.o
"-s"参数：将所有段的内容以十六进制的方式打印出来
"-d"参数：将所有包含指令的段反汇编
```
SimpleSection.o：     文件格式 elf64-x86-64

Contents of section .text:
 0000 554889e5 4883ec10 897dfc8b 45fc89c6  UH..H....}..E...
 0010 bf000000 00b80000 0000e800 00000090  ................
 0020 c9c35548 89e54883 ec10c745 f8010000  ..UH..H....E....
 0030 008b1500 0000008b 05000000 0001c28b  ................
 0040 45f801c2 8b45fc01 d089c7e8 00000000  E....E..........
 0050 8b45f8c9 c3                          .E...           
Contents of section .data:
 0000 54000000 55000000                    T...U...        
Contents of section .rodata:
 0000 25640a00                             %d..            
Contents of section .comment:
 0000 00474343 3a202855 62756e74 7520352e  .GCC: (Ubuntu 5.
 0010 342e302d 36756275 6e747531 7e31362e  4.0-6ubuntu1~16.
 0020 30342e39 2920352e 342e3020 32303136  04.9) 5.4.0 2016
 0030 30363039 00                          0609.           
Contents of section .eh_frame:
 0000 14000000 00000000 017a5200 01781001  .........zR..x..
 0010 1b0c0708 90010000 1c000000 1c000000  ................
 0020 00000000 22000000 00410e10 8602430d  ...."....A....C.
 0030 065d0c07 08000000 1c000000 3c000000  .]..........<...
 0040 00000000 33000000 00410e10 8602430d  ....3....A....C.
 0050 066e0c07 08000000                    .n......        

Disassembly of section .text:

0000000000000000 <func1>:
   0:	55                   	push   %rbp
   1:	48 89 e5             	mov    %rsp,%rbp
   4:	48 83 ec 10          	sub    $0x10,%rsp
   8:	89 7d fc             	mov    %edi,-0x4(%rbp)
   b:	8b 45 fc             	mov    -0x4(%rbp),%eax
   e:	89 c6                	mov    %eax,%esi
  10:	bf 00 00 00 00       	mov    $0x0,%edi
  15:	b8 00 00 00 00       	mov    $0x0,%eax
  1a:	e8 00 00 00 00       	callq  1f <func1+0x1f>
  1f:	90                   	nop
  20:	c9                   	leaveq 
  21:	c3                   	retq   

0000000000000022 <main>:
  22:	55                   	push   %rbp
  23:	48 89 e5             	mov    %rsp,%rbp
  26:	48 83 ec 10          	sub    $0x10,%rsp
  2a:	c7 45 f8 01 00 00 00 	movl   $0x1,-0x8(%rbp)
  31:	8b 15 00 00 00 00    	mov    0x0(%rip),%edx        # 37 <main+0x15>
  37:	8b 05 00 00 00 00    	mov    0x0(%rip),%eax        # 3d <main+0x1b>
  3d:	01 c2                	add    %eax,%edx
  3f:	8b 45 f8             	mov    -0x8(%rbp),%eax
  42:	01 c2                	add    %eax,%edx
  44:	8b 45 fc             	mov    -0x4(%rbp),%eax
  47:	01 d0                	add    %edx,%eax
  49:	89 c7                	mov    %eax,%edi
  4b:	e8 00 00 00 00       	callq  50 <main+0x2e>
  50:	8b 45 f8             	mov    -0x8(%rbp),%eax
  53:	c9                   	leaveq 
  54:	c3                   	retq
```
"Contents of section .text:"就是.text的数据以十六进制方式打印出来的内容，共0X55个字节。最左边一列是偏移量，中间4列是十六进制内容，最右边一列是.text段的ASCII码形式。
对照反汇编表，.text段所包含的正是SimpleSection.c里面两个函数func1()和main()的指令。.text段的第一个字节“0X55”就是func1()函数的第一条指令“push   %rbp”，而最后一个字节0xc3是main()函数的最后一条指令“ret”。

### 3.2 数据段和只读数据段
`.data`：保存已初始化的全局变量和局部变量。SimpleSection.c中一共有两个这样的变量，global_init_varabal和static_var。这两个变量各4个字节，故.data段的大小为8个字节。

SimpleSection.c里面函数printf()用到字符型常量“%d\n”，它是一种只读数据，被放到了.rodata段。.rodata段内容“25640a00 ”刚好是这个字符串常量的ASCII字节序，最后以"\0"结尾。

.rodata段存放的是只读数据，一般是程序中的只读变量（const修饰的变量）或字符串常量。单独设立.rodata段，不光在语义上支持了C++的const关键字，而且操作系统在加载的时候可以将.rodata段的属性映射成只读（页表的作用？），这样对于该段的任何修改操作都会作为非法处理，保证了程序的安全性。

### 3.3 .bss段
`.bss`：存放未初始化的全局变量和局部静态变量，上述代码中的global_uinit_var和static_var2就是被存放在.bss段。更准确的说法是.bss段为它们预留了空间。
但是该段却只有4个字节，与两个变量共8个字节不符。

实际上，只有static_var2被存放在.bss段，而global_uinit_var却没有被放在任何段，只是一个未定义的COMMON符号。这与不同的语言和不同的编译器有关，有些编译器会将全局的未初始化变量存放在.bss段，有些则不，只是预留一个未定义的全局变量符合，等到最终链接成可执行文件的时候再在.bss段上分配空间。

一个小问题：
> static int x1 = 0;
static int x2 = 1;

x1会被放在.bss段中，x2会被放在.data段中。因为x1位0可以视为未初始化的，被优化掉放在.bss，这样可以节省磁盘空间，因为.bss不占磁盘空间。x2是被初始化的，放在.data段中。

## 4.ELF文件结构
### 4.1 段表
段表描述了ELF各个段的信息，比如每个段的段名、段的长度、在文件中的偏移、读写权限及段的其他属性。ELF文件的段结构就是由段表决定的，编译器、链接器和装载器都是依靠段表来定位和访问各个段的属性的。

使用readelf工具查看ELF文件的段：
>  readelf -S SimpleSection.o
```
共有 13 个节头，从偏移量 0x430 开始：

节头：
  [号] 名称              类型             地址              偏移量
       大小              全体大小          旗标   链接   信息   对齐
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       0000000000000055  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  00000320
       0000000000000078  0000000000000018   I      11     1     8
  [ 3] .data             PROGBITS         0000000000000000  00000098
       0000000000000008  0000000000000000  WA       0     0     4
  [ 4] .bss              NOBITS           0000000000000000  000000a0
       0000000000000004  0000000000000000  WA       0     0     4
  [ 5] .rodata           PROGBITS         0000000000000000  000000a0
       0000000000000004  0000000000000000   A       0     0     1
  [ 6] .comment          PROGBITS         0000000000000000  000000a4
       0000000000000035  0000000000000001  MS       0     0     1
  [ 7] .note.GNU-stack   PROGBITS         0000000000000000  000000d9
       0000000000000000  0000000000000000           0     0     1
  [ 8] .eh_frame         PROGBITS         0000000000000000  000000e0
       0000000000000058  0000000000000000   A       0     0     8
  [ 9] .rela.eh_frame    RELA             0000000000000000  00000398
       0000000000000030  0000000000000018   I      11     8     8
  [10] .shstrtab         STRTAB           0000000000000000  000003c8
       0000000000000061  0000000000000000           0     0     1
  [11] .symtab           SYMTAB           0000000000000000  00000138
       0000000000000180  0000000000000018          12    11     8
  [12] .strtab           STRTAB           0000000000000000  000002b8
       0000000000000066  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), l (large)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```
对于编译器和链接器来说，主要决定一个段的属性是段的类型(sh_type)和段的标致(sh_flags)

### 4.2 重定位表
SimpleSection.o中有一个叫做“.rel.text”的段，其类型为“SHT_REL”,即一个重定位表。链接器在处理目标文件时，需要对目标文件中的某些部位进行重定位，即代码段和数据段中那些绝对地址引用的位置。这些重定位的信息都记录在ELF文件的重定位表中，对于每个需要重定位的代码段或数据段，都会有一个相应的重定位表。比如SimpleSection.o中的“.rel.text”就是针对“.text”段的重定位表，因为“.text”段中至少有一个printf函数的绝对地址的引用；而“.data”段中没有绝对地址的引用，它只包含了几个常量，所以SimpleSection.o中没有针对“.data”段的重定位表“.rel.data”。

一个重定位表就是ELF中的一个段，段的类型时SHT_REL。 它的sh_link表示符号的下标，它的sh_info表示它作用于哪个段。如“.rel.text”作用于“.text”段，而“.text”段下标为1，那么“.rel.text”的sh_info为1。

## 5.符号
链接的过程本质就是要把多个不同的目标文件组合到一起。在链接中，目标文件之间相互拼合实际上就是目标文件之间对地址的引用，即对函数和变量的地址的引用。在连接中，将函数和变量统称为符号，函数名或变量名就是符号名。
链接过程中很关键的一部分就是符号的管理，每一个目标文件都会有一个相应的符号表，这个表记录了目标文件中所用到的所有符号。每个定义的符号都有一个对应的值，叫做符号值。对于变量和函数来说，符号值就是它们的地址。
可以使用redaelf、objdump、nm等工具来查看目标文件的符号，如：
>  nm SimpleSection.o
```
0000000000000000 T func1
0000000000000000 D global_init_var
0000000000000004 C global_uninit_var
0000000000000022 T main
                 U pirntf
0000000000000004 d static_var.1842
0000000000000000 b static_var2.1843
```

# 静态链接

当有两个目标文件，如何把它们链接起来形成一个可执行文件？

## 1.空间与地址分配
对于链接器来说，链接过程就是将几个输入目标文件加工后合并成一个输出文件。

链接器采用合并相似段方式。即，将所有输入文件的.text合并到输出文件的.text，接着是.data段、.bss段等。

链接器为目标文件分配地址和空间中地址和空间包含两种含义：一是指在输出的可执行文件中的空间，二是指装载后的虚拟地址中的虚拟地址空间。对于有实际数据的段，如.text和.data来说，它们在文件和虚拟地址中都要分配空间。对于.bss段来说，分配空间的意义只局限于虚拟地址空间的分配。

现在链接器一般采用两步链接：          
第一步：空间与地址分配        
第二步：符号解析与重定位           

使用ld链接器将a.o和b.o链接起来。
> ld a.o b.o -e main -o ab

> 这里链接有个undefined reference to `__stack_chk_fail'错误，我解决的办法是在编译a.c时，禁用堆栈保护。       
> gcc -fno-stack-protector -c a.c此处，具体对程序运行有什么影响，目前我尚未探究，因为我现在只关心链接部分内容。

-e main：表示main函数作为程序入口，ld链接器默认的程序入口为_start           
-o ab： 表示链接输出文件名为ab，默认为a.out          

链接前后各个段的属性：       
```
$ objdump -h a.o
a.o：     文件格式 elf64-x86-64
节：
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         0000002c  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000000  0000000000000000  0000000000000000  0000006c  2**0
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  0000006c  2**0
                  ALLOC
  3 .comment      00000035  0000000000000000  0000000000000000  0000006c  2**0
                  CONTENTS, READONLY
  4 .note.GNU-stack 00000000  0000000000000000  0000000000000000  000000a1  2**0
                  CONTENTS, READONLY
  5 .eh_frame     00000038  0000000000000000  0000000000000000  000000a8  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA

$ objdump -h b.o
b.o：     文件格式 elf64-x86-64
节：
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         0000004b  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .data         00000004  0000000000000000  0000000000000000  0000008c  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  00000090  2**0
                  ALLOC
  3 .comment      00000035  0000000000000000  0000000000000000  00000090  2**0
                  CONTENTS, READONLY
  4 .note.GNU-stack 00000000  0000000000000000  0000000000000000  000000c5  2**0
                  CONTENTS, READONLY
  5 .eh_frame     00000038  0000000000000000  0000000000000000  000000c8  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA

$ objdump -h ab
ab：     文件格式 elf64-x86-64
节：
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00000077  00000000004000e8  00000000004000e8  000000e8  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .eh_frame     00000058  0000000000400160  0000000000400160  00000160  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .data         00000004  00000000006001b8  00000000006001b8  000001b8  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  3 .comment      00000034  0000000000000000  0000000000000000  000001bc  2**0
                  CONTENTS, READONLY
```
可以看到，链接前所有段的虚拟地址都是0，因为虚拟空间还未分配，所以他默认都为0。等到链接之后，可执行文件ab中的各个段都被分配到了相应的虚拟地址。

## 2.符号解析与重定位
在完成空间和地址分配后，链接器进入符号解析与重定位步骤。           
文件a.o是如何使用外部符号的呢？通过代码的反汇编来结果来查看         
> objdump -d a.o
```
a.o：     文件格式 elf64-x86-64

Disassembly of section .text:

0000000000000000 <main>:
   0:	55                   	push   %rbp
   1:	48 89 e5             	mov    %rsp,%rbp
   4:	48 83 ec 10          	sub    $0x10,%rsp
   8:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
   f:	00 00 
  11:	48 89 45 f8          	mov    %rax,-0x8(%rbp)
  15:	31 c0                	xor    %eax,%eax
  17:	c7 45 f4 64 00 00 00 	movl   $0x64,-0xc(%rbp)
  1e:	48 8d 45 f4          	lea    -0xc(%rbp),%rax
  22:	be 00 00 00 00       	mov    $0x0,%esi
  27:	48 89 c7             	mov    %rax,%rdi
  2a:	b8 00 00 00 00       	mov    $0x0,%eax
  2f:	e8 00 00 00 00       	callq  34 <main+0x34>
  34:	b8 00 00 00 00       	mov    $0x0,%eax
  39:	48 8b 55 f8          	mov    -0x8(%rbp),%rdx
  3d:	64 48 33 14 25 28 00 	xor    %fs:0x28,%rdx
  44:	00 00 
  46:	74 05                	je     4d <main+0x4d>
  48:	e8 00 00 00 00       	callq  4d <main+0x4d>
  4d:	c9                   	leaveq 
  4e:	c3                   	retq
```
可以看到.text中有一个函数main。该函数占用0X4e个字节，共19个指令。在0x22位置，对shared变量的引用是一条mov指令。                
当编译器将a.c编译成目标文件时，编译器并不知道shared和swap的地址，因为它们定义在其他目标文件中。所以编译器就暂时把地址0看作是shared的地址。             
另外一个偏移为0x2f的指令是一条调用指令，其实就是对swap函数的调用。            
编译器把这两条指令的地址部分暂时用假地址来代替，把真正的地址计算留给了链接器。链接器可根据符号的地址对每个需要重定位的指令进行地址修正。           
使用objdump来反汇编输出程序ab的代码段。              
```
$ objdump -d ab
00000000004000e8 <main>:
  4000e8:	55                   	push   %rbp
  4000e9:	48 89 e5             	mov    %rsp,%rbp
  4000ec:	48 83 ec 10          	sub    $0x10,%rsp
  4000f0:	c7 45 fc 64 00 00 00 	movl   $0x64,-0x4(%rbp)
  4000f7:	48 8d 45 fc          	lea    -0x4(%rbp),%rax
  4000fb:	be b8 01 60 00       	mov    $0x6001b8,%esi
  400100:	48 89 c7             	mov    %rax,%rdi
  400103:	b8 00 00 00 00       	mov    $0x0,%eax
  400108:	e8 07 00 00 00       	callq  400114 <swap>
  40010d:	b8 00 00 00 00       	mov    $0x0,%eax
  400112:	c9                   	leaveq 
  400113:	c3                   	retq   

0000000000400114 <swap>:
  400114:	55                   	push   %rbp
  400115:	48 89 e5             	mov    %rsp,%rbp
  400118:	48 89 7d f8          	mov    %rdi,-0x8(%rbp)
  40011c:	48 89 75 f0          	mov    %rsi,-0x10(%rbp)
  400120:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  400124:	8b 10                	mov    (%rax),%edx
  400126:	48 8b 45 f0          	mov    -0x10(%rbp),%rax
  40012a:	8b 00                	mov    (%rax),%eax
  40012c:	31 c2                	xor    %eax,%edx
  40012e:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  400132:	89 10                	mov    %edx,(%rax)
  400134:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  400138:	8b 10                	mov    (%rax),%edx
  40013a:	48 8b 45 f0          	mov    -0x10(%rbp),%rax
  40013e:	8b 00                	mov    (%rax),%eax
  400140:	31 c2                	xor    %eax,%edx
  400142:	48 8b 45 f0          	mov    -0x10(%rbp),%rax
  400146:	89 10                	mov    %edx,(%rax)
  400148:	48 8b 45 f0          	mov    -0x10(%rbp),%rax
  40014c:	8b 10                	mov    (%rax),%edx
  40014e:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  400152:	8b 00                	mov    (%rax),%eax
  400154:	31 c2                	xor    %eax,%edx
  400156:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  40015a:	89 10                	mov    %edx,(%rax)
  40015c:	90                   	nop
  40015d:	5d                   	pop    %rbp
  40015e:	c3                   	retq   
```

其实重定位过程也伴随着符号的解析过程，每个目标文件都可能定义一些符号，也可能引用到定义在其他目标文件的符号。重定位过程中，每个重定位入口都是对一个符号的引用，那么当链接器需要对某个符号的引用进行重定位时，它就要确定这个符号的目标地址。此时，链接器就会去查找由所有输入目标文件的符号表组成的全局符号表，找到相应的符号后进行重定位。            
查看a.o的符号表
```
$ readelf -s a.o
Symbol table '.symtab' contains 11 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS a.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     8: 0000000000000000    44 FUNC    GLOBAL DEFAULT    1 main
     9: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND shared
    10: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND swap
```
"GLOBAL"类型的符号，除了main函数是定义在代码段之外，其他两个shared和swap都是“UND”，即undefined未定义类型，这种未定义类型的符号都是因为该目标文件中有关于它们的重定位项。所以在链接器扫描完所有的输入目标文件之后，所有这些未定义的符号能够在全局符号表中找到，否则链接器就报符号未定义错误。             

## COMMON块
链接器如何处理多个符号名字一样但其类型不一样的情况（符号均为弱符号的情况）。          
采用COMMON块机制。当不同的目标文件需要的COMMON块空间大小不一致时，以最大的那块为准。

# 4.C++相关问题
TODO

# 5.静态库链接
一个静态库可以看做一组目标文件的集合，即很多目标文件经过压缩打包后形成的一个文件。Linux中最常用的C语言静态库libc位于/usr/lib/x86_64-linux-gnu/libc.a。          
但是程序的目标文件如何与C语言运行库链接形成一个可执行文件呢？         
C语言运行库中，包含了许多跟系统功能相关的代码，比如输入输出、文件操作、时间日期、内存管理等。glibc本身是用C语言开发的，它由成百上千个C语言源代码文件组成，也就是，编译完成后由相同数量的目标文件，比如，输入输出printf.o，scanf.o；等把这些零散的目标文件提供给库的使用者，很大程度上会造成文件传输、管理和组织方面的不便，于是通常使用ar压缩程序命令将这些目标文件压缩到一起，并且对其进行编号和索引，以便查找和检索，就形成了libc.a这个静态库文件。          

假设hello.c中只调用函数printf()，编译链接如下
```
gcc -c -fno-builtin hello.c
ar -x libc.a
ld hello.o printf.o
```
按照道理，应该能链接成功，但实际上却链接失败。原因是printf.o中有其他外部符号的引用，也就是说printf.o依赖于其他的目标文件。              
理论上，把printf.o依赖的文件，及printf.o依赖文件的依赖文件全部收集齐，就可以链接成功，输出可执行文件。         

使用gcc命令编译hello.c，-verbose表示将整个编译链接过程的中间步骤打印出来：
```
gcc -static -verbose -fno-builtin hello.c
```
其关键步骤是:      
1.调用cc1程序，将hello.c编译成一个临时的汇编文件        
2.调用as程序，将汇编文件汇编成临时目标文件           
3.调用collect2来完成最后的链接。            

# 6.链接过程的控制
链接器一般提供多种控制整个链接过程的方法，以用来产生用户所需要的文件。用的较多的方法是，使用链接控制脚本。            
简单来讲，控制链接过程无非就是控制输入段如何变成输出段。比如，哪些输入段要合并成一个输出段，哪些输入段要丢弃；指定输出段的名字、装载地址、属性，等等。            
# 动态链接

## 1.为什么要动态链接
静态链接有浪费内存和磁盘空间、模块更新困难等缺点。

浪费内存和磁盘：Program1和Program2包含Program1.o和Program2.o两个模块，同时，它们还包含Lib.o模块。在静态链接的情况下，因为Program1.o和Program2.o都用到了Lib.o模块，所以它们在同时链接输出的可执行文件Program1和Program2有两个副本。当运行Program1和Program2时，Lib.o在磁盘和内存中都有两份副本。        
模块更新困难：如Lib.o进行了更新，那么Program1就要拿到最新的Lib.o，然后将其与Program1.o链接后，发布新的Program1。这样做的缺点，即一旦程序中有任何模块更新，整个程序就要重新链接、发布。

解决这两个问题最简单的办法就是把程序的模块相互分割开来，形成独立的文件，而不再将它们静态链接在一起。简单来说，就是不对那些组成程序的目标文件进行链接，等到程序要运行时才进行链接。这也是**动态链接**的基本思想。

这样的做法解决了共享的目标文件多个副本浪费磁盘和内存空间的问题。另外在内存中共享一个目标文件模块的好处不仅仅是减少内存，它还可以减少物理页面的换入换出，也可以增加CPU缓存的命中率。

## 2.简单的动态链接的例子


```
/*Program1.c*/
#include "Lib.h"

int main() 
{
	foobar(1);
	return 0;
}

/*Program2.c*/
#include "Lib.h"

int main()
{
	foobar(2);
	return 0;
}

/*Lib.c*/
#include <stdio.h>

void foobar(int i)
{
	printf("Printing from Lib.so %d\n", i);
}

/*Lib.h*/
#ifndef LIB_H
#define LIB_H

void foobar(int i);

#endif
```
使用gcc将Lib.c编译成一个共享对象文件
> $ gcc -fPIC -shared -o Lib.so Lib.c                    
> -shared 表示产生共享对象          
> -fPIC 表示位置无关代码        

此时得到Lib.so文件，这就是包含了Lib.c的foobar()函数的共享对象文件。然后分别编译链接Program1.c和Program2.c
> $ gcc -o Program1 Program1.c ./Lib.so      

> $ gcc -o Program2 Program2.c ./Lib.so
这样就得到了两个程序Program1和Program2，这两个程序都使用了Lib.so里面的foobar()函数。

当链接器将Program1.o链接成可执行文件是，这时候链接器必须确定Program1.o中所引用的foobar()函数的性质。如果foobar()是一个定义与其他静态目标模块中的函数，那么链接器将会按照静态链接这个规则，将Program1.o中的foobar地址引用重定位；如果foobar()是一个定义在某个动态共享对象中的函数，那么链接器就会将这个符号的引用标记为一个动态链接的符号，不对它进行地址重定位，把这个过程留到装载时再进行。            

链接器如何知道foobar的引用是一个静态符号还是动态符号？
把Lib.so作为链接输入之一，链接器在解析符号时就可以知道：foobar是一个定义在Lib.so的动态符号。这样链接器就可以对foobar的引用做特殊处理，使它成为一个对动态符号的引用。         

**动态链接程序运行时地址空间分布**
```
$./Program1 &
$ ./Program1 &
[1] 4378
Printing from Lib.so 1

$ cat /proc/4378/maps
00400000-00401000 r-xp 00000000 08:02 3145979                            /home/user/linker_loader/ch7/Program1
00600000-00601000 r--p 00000000 08:02 3145979                            /home/user/linker_loader/ch7/Program1
00601000-00602000 rw-p 00001000 08:02 3145979                            /home/user/linker_loader/ch7/Program1
006c4000-006e5000 rw-p 00000000 00:00 0                                  [heap]
7f2581d5a000-7f2581f1a000 r-xp 00000000 08:02 15339940                   /lib/x86_64-linux-gnu/libc-2.23.so
7f2581f1a000-7f258211a000 ---p 001c0000 08:02 15339940                   /lib/x86_64-linux-gnu/libc-2.23.so
7f258211a000-7f258211e000 r--p 001c0000 08:02 15339940                   /lib/x86_64-linux-gnu/libc-2.23.so
7f258211e000-7f2582120000 rw-p 001c4000 08:02 15339940                   /lib/x86_64-linux-gnu/libc-2.23.so
7f2582120000-7f2582124000 rw-p 00000000 00:00 0 
7f2582124000-7f2582125000 r-xp 00000000 08:02 3145976                    /home/user/linker_loader/ch7/Lib.so
7f2582125000-7f2582324000 ---p 00001000 08:02 3145976                    /home/user/linker_loader/ch7/Lib.so
7f2582324000-7f2582325000 r--p 00000000 08:02 3145976                    /home/user/linker_loader/ch7/Lib.so
7f2582325000-7f2582326000 rw-p 00001000 08:02 3145976                    /home/user/linker_loader/ch7/Lib.so
7f2582326000-7f258234c000 r-xp 00000000 08:02 15339912                   /lib/x86_64-linux-gnu/ld-2.23.so
7f258252f000-7f2582532000 rw-p 00000000 00:00 0 
7f258254a000-7f258254b000 rw-p 00000000 00:00 0 
7f258254b000-7f258254c000 r--p 00025000 08:02 15339912                   /lib/x86_64-linux-gnu/ld-2.23.so
7f258254c000-7f258254d000 rw-p 00026000 08:02 15339912                   /lib/x86_64-linux-gnu/ld-2.23.so
7f258254d000-7f258254e000 rw-p 00000000 00:00 0 
7ffc12c26000-7ffc12c47000 rw-p 00000000 00:00 0                          [stack]
7ffc12d8f000-7ffc12d92000 r--p 00000000 00:00 0                          [vvar]
7ffc12d92000-7ffc12d94000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```
可以看到整个进程虚拟地址空间中，多出了几个文件的映射。Lib.so与Program1一样，它们都被操作系统以同样的方法映射至进程的虚拟地址空间，只是它们占据的虚拟地址空间和长度不同。Program1除了使用Lib.so以外，它还用到了动态链接形式的C语言运行库libc-***.so。另外还有共享对象ld-***.so，它实际上是Linux下的动态链接器。动态链接器与普通共享对象一样被映射到了进程的地址空间，在系统开始运行Program1之前，首先把控制权交给动态链接器，由它完成所有的动态链接工作以后再把控制权交给Program1，然后开始执行。                 

通过readelf工具来查看Lib.so的装载属性。
```
$ readelf -l Lib.so 

Elf 文件类型为 DYN (共享目标文件)
入口点 0x5e0
共有 7 个程序头，开始于偏移量 64

程序头：
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x00000000000007bc 0x00000000000007bc  R E    200000
  LOAD           0x0000000000000e00 0x0000000000200e00 0x0000000000200e00
                 0x0000000000000230 0x0000000000000238  RW     200000
  DYNAMIC        0x0000000000000e18 0x0000000000200e18 0x0000000000200e18
                 0x00000000000001c0 0x00000000000001c0  RW     8
  NOTE           0x00000000000001c8 0x00000000000001c8 0x00000000000001c8
                 0x0000000000000024 0x0000000000000024  R      4
  GNU_EH_FRAME   0x0000000000000738 0x0000000000000738 0x0000000000000738
                 0x000000000000001c 0x000000000000001c  R      4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     10
  GNU_RELRO      0x0000000000000e00 0x0000000000200e00 0x0000000000200e00
                 0x0000000000000200 0x0000000000000200  R      1

 Section to Segment mapping:
  段节...
   00     .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .plt.got .text .fini .rodata .eh_frame_hdr .eh_frame 
   01     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss 
   02     .dynamic 
   03     .note.gnu.build-id 
   04     .eh_frame_hdr 
   05     
   06     .init_array .fini_array .jcr .dynamic .got 
```
除了文件类型与普通程序不同以外，其他几乎与普通程序一样。还有有一点比较不同，动态链接的装载地址是从0X00000000开始的。这个是个无效地址，从进程虚拟空间分布可以看到Lib.so最终装载地址。故，**共享对象的最终装载地址在编译时是不能确定的**，而是在装载时，装载器根据当前地址空间的空间情况，动态分配一块足够大小的虚拟地址空间给相应的共享对象。


## 3.地址无关代码
动态链接中，共享对象在装载时，怎么确定它在进程虚拟地址空间中的位置？         
装载时重定位能解决上面的问题。但是其无法解决指令部分在过个进程中共享。我们希望程序模块中共享的指令部分在装载时不因装载地址的改变而改变，故实现该目的的方法是将那些需要被修改的部分分离出来跟数据放在一起，这样指令部分就可以保持不变，而数据部分就可以在每个进程中拥有一个副本。这种方案就是地址无关代码。

把共享对象模块中的地址引用按照是否跨模块分为模块内引用和模块外部引用；按照不同的引用方式可以分为指令引用和数据访问。故有：
1. 模块内部的函数调用、跳转
2. 模块内部的数据访问（模块中定义的全局变量、静态变量）
3. 模块外部的函数调用、跳转
4. 模块外部的数据访问 （其他模块定义的全局变量）

对于模块内部的函数调用、跳转是不需要重定位的。对于模块内部的数据访问，其指令和要访问的模块内部数据直间的相对位置都是固定的，在指令前加上固定的偏移量就可以访问模块内部数据了。对于模块外部的数据访问，ELF通过在数据段中建立一个指向这些变量的指针数组（也称全局偏移表GOT）来做间接引用。对于模块外部的函数调用、跳转与模块外部的数据访问类似，也是通过GOT来实现。      

至此，这四种地址引用在理论上都实现了地址无关性。     
||指令跳转、调用|数据访问|
|---|---|---|
|模块内部|（1）相对跳转和调用|（2）相对地址访问
|模块外部|（3）间接跳转和调用（GOT）|（4）间接访问（GOT）


## 4.延迟绑定（PLT）
动态链接比静态链接的程序性能差，原因是：        
1、动态链接对于全局全局和静态的数据访问都有进行GOT定位，然后间接寻址。        
2、动态链接中链接的动作是在运行时完成的。

为优化性能采用延迟绑定做法。    
动态链接中，程序模块之间包含了大量的函数引用。在程序开始之前，动态链接要耗费时间解决模块之间的函数引用的符号查找以及重定位。但是很多函数在程序执行期间并不会被调用，所以一开始就把所有函数都链接好是一种浪费。在ELF中采用延迟绑定做法，即当函数第一次被调用时才进行绑定（非常向懒加载的概念）。

## 5.动态链接实现所需要的内容

### 5.1从可执行文件中找到动态链接器
动态链接器也是一个程序，它的位置从ELF文件中的`.interp`段可以找到
```
$ objdump -s Program1

Program1：     文件格式 elf64-x86-64

Contents of section .interp:
 400238 2f6c6962 36342f6c 642d6c69 6e75782d  /lib64/ld-linux-
 400248 7838362d 36342e73 6f2e3200           x86-64.so.2. 
```
`.interp`段内部就是一个字符串，该字符串就是动态链接器的路径，这里是/lib64/ld-linux-x86-64.so.2.         

还可以通过下面的命令来查看动态链接器的路径           
> readelf -l a.out | grep interpreter
```
$ readelf -l Program1 | grep interpreter
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

### 5.2 ".dynamic"段
`.dynamic`段保存了动态链接器所需要的基本信息。例如：共享对象、动态链接符号表的位置、动态链接重定位表的位置、共享对象初始化代码的地址等。         
可通过readelf工具来查看`.dynamic`段的内容。
```
$ readelf -d Lib.so 

Dynamic section at offset 0xe18 contains 24 entries:
  标记        类型                         名称/值
 0x0000000000000001 (NEEDED)             共享库：[libc.so.6]
 0x000000000000000c (INIT)               0x580
 0x000000000000000d (FINI)               0x714
 0x0000000000000019 (INIT_ARRAY)         0x200e00
 0x000000000000001b (INIT_ARRAYSZ)       8 (bytes)
 0x000000000000001a (FINI_ARRAY)         0x200e08
 0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x1f0
 0x0000000000000005 (STRTAB)             0x398
 0x0000000000000006 (SYMTAB)             0x230
 0x000000000000000a (STRSZ)              183 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000003 (PLTGOT)             0x201000
 0x0000000000000002 (PLTRELSZ)           48 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0x550
 0x0000000000000007 (RELA)               0x490
 0x0000000000000008 (RELASZ)             192 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000006ffffffe (VERNEED)            0x470
 0x000000006fffffff (VERNEEDNUM)         1
 0x000000006ffffff0 (VERSYM)             0x450
 0x000000006ffffff9 (RELACOUNT)          3
 0x0000000000000000 (NULL)               0x0
```
Linux中还提供了用来查看程序主模块或一个共享库依赖于哪些共享库的命令：
```
$ ldd Program1
	linux-vdso.so.1 =>  (0x00007ffe28d5e000)
	./Lib.so (0x00007fa9d0c19000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fa9d084f000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fa9d0e1b000)
```

### 5.3动态符号表  
符号表`.symtab`，里面保存了关于该目标文件的符号的定义和引用。ELF中用动态符号表`.dynsym`来保存模块之间的符号的导入导出关系。`.dynsym`段只保存了与动态链接相关的符号，不保存模块内部的符号。可以通过readelf工具来查看
```
$ readelf -s Lib.so 

Symbol table '.dynsym' contains 15 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000580     0 SECTION LOCAL  DEFAULT    9 
     2: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.2.5 (2)
     4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     5: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _Jv_RegisterClasses
     6: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
     7: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND sleep@GLIBC_2.2.5 (2)
     8: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@GLIBC_2.2.5 (2)
     9: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT   23 _edata
    10: 0000000000201038     0 NOTYPE  GLOBAL DEFAULT   24 _end
    11: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT   24 __bss_start
    12: 0000000000000580     0 FUNC    GLOBAL DEFAULT    9 _init
    13: 0000000000000714     0 FUNC    GLOBAL DEFAULT   13 _fini
    14: 00000000000006e0    51 FUNC    GLOBAL DEFAULT   12 foobar

$ readelf -sD Lib.so 

Symbol table of `.gnu.hash' for image:
  Num Buc:    Value          Size   Type   Bind Vis      Ndx Name
    9   0: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT  23 _edata
   10   0: 0000000000201038     0 NOTYPE  GLOBAL DEFAULT  24 _end
   11   1: 0000000000201030     0 NOTYPE  GLOBAL DEFAULT  24 __bss_start
   12   1: 0000000000000580     0 FUNC    GLOBAL DEFAULT   9 _init
   13   2: 0000000000000714     0 FUNC    GLOBAL DEFAULT  13 _fini
   14   2: 00000000000006e0    51 FUNC    GLOBAL DEFAULT  12 foobar
```

### 5.4动态链接重定位表
导入符号的存在，使得共享对象需要被重定位。       
动态链接文件中，使用“.rel.dyn”对数据引用的修正和“.rel.plt”对函数修正。     
使用readelf来查看一个动态链接文件的重定位表：
```
$ readelf -r Lib.so 

重定位节 '.rela.dyn' 位于偏移量 0x490 含有 8 个条目：
  偏移量          信息           类型           符号值        符号名称 + 加数
000000200e00  000000000008 R_X86_64_RELATIVE                    6b0
000000200e08  000000000008 R_X86_64_RELATIVE                    670
000000201028  000000000008 R_X86_64_RELATIVE                    201028
000000200fd8  000200000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTMClone + 0
000000200fe0  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000200fe8  000500000006 R_X86_64_GLOB_DAT 0000000000000000 _Jv_RegisterClasses + 0
000000200ff0  000600000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCloneTa + 0
000000200ff8  000800000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0

重定位节 '.rela.plt' 位于偏移量 0x550 含有 2 个条目：
  偏移量          信息           类型           符号值        符号名称 + 加数
000000201018  000300000007 R_X86_64_JUMP_SLO 0000000000000000 printf@GLIBC_2.2.5 + 0
000000201020  000700000007 R_X86_64_JUMP_SLO 0000000000000000 sleep@GLIBC_2.2.5 + 0



$ readelf -S Lib.so 
共有 29 个节头，从偏移量 0x18a8 开始：

节头：
  [号] 名称              类型             地址              偏移量
       大小              全体大小          旗标   链接   信息   对齐
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .note.gnu.build-i NOTE             00000000000001c8  000001c8
       0000000000000024  0000000000000000   A       0     0     4
  [ 2] .gnu.hash         GNU_HASH         00000000000001f0  000001f0
       000000000000003c  0000000000000000   A       3     0     8
  [ 3] .dynsym           DYNSYM           0000000000000230  00000230
       0000000000000168  0000000000000018   A       4     2     8
  [ 4] .dynstr           STRTAB           0000000000000398  00000398
       00000000000000b7  0000000000000000   A       0     0     1
  [ 5] .gnu.version      VERSYM           0000000000000450  00000450
       000000000000001e  0000000000000002   A       3     0     2
  [ 6] .gnu.version_r    VERNEED          0000000000000470  00000470
       0000000000000020  0000000000000000   A       4     1     8
  [ 7] .rela.dyn         RELA             0000000000000490  00000490
       00000000000000c0  0000000000000018   A       3     0     8
  [ 8] .rela.plt         RELA             0000000000000550  00000550
       0000000000000030  0000000000000018  AI       3    22     8
  [ 9] .init             PROGBITS         0000000000000580  00000580
       000000000000001a  0000000000000000  AX       0     0     4
  [10] .plt              PROGBITS         00000000000005a0  000005a0
       0000000000000030  0000000000000010  AX       0     0     16
  [11] .plt.got          PROGBITS         00000000000005d0  000005d0
       0000000000000010  0000000000000000  AX       0     0     8
  [12] .text             PROGBITS         00000000000005e0  000005e0
       0000000000000133  0000000000000000  AX       0     0     16
  [13] .fini             PROGBITS         0000000000000714  00000714
       0000000000000009  0000000000000000  AX       0     0     4
  [14] .rodata           PROGBITS         000000000000071d  0000071d
       0000000000000019  0000000000000000   A       0     0     1
  [15] .eh_frame_hdr     PROGBITS         0000000000000738  00000738
       000000000000001c  0000000000000000   A       0     0     4
  [16] .eh_frame         PROGBITS         0000000000000758  00000758
       0000000000000064  0000000000000000   A       0     0     8
  [17] .init_array       INIT_ARRAY       0000000000200e00  00000e00
       0000000000000008  0000000000000000  WA       0     0     8
  [18] .fini_array       FINI_ARRAY       0000000000200e08  00000e08
       0000000000000008  0000000000000000  WA       0     0     8
  [19] .jcr              PROGBITS         0000000000200e10  00000e10
       0000000000000008  0000000000000000  WA       0     0     8
  [20] .dynamic          DYNAMIC          0000000000200e18  00000e18
       00000000000001c0  0000000000000010  WA       4     0     8
  [21] .got              PROGBITS         0000000000200fd8  00000fd8
       0000000000000028  0000000000000008  WA       0     0     8
  [22] .got.plt          PROGBITS         0000000000201000  00001000
       0000000000000028  0000000000000008  WA       0     0     8
  [23] .data             PROGBITS         0000000000201028  00001028
       0000000000000008  0000000000000000  WA       0     0     8
  [24] .bss              NOBITS           0000000000201030  00001030
       0000000000000008  0000000000000000  WA       0     0     1
  [25] .comment          PROGBITS         0000000000000000  00001030
       0000000000000034  0000000000000001  MS       0     0     1
  [26] .shstrtab         STRTAB           0000000000000000  000017af
       00000000000000f6  0000000000000000           0     0     1
  [27] .symtab           SYMTAB           0000000000000000  00001068
       0000000000000570  0000000000000018          28    45     8
  [28] .strtab           STRTAB           0000000000000000  000015d8
       00000000000001d7  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), l (large)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```
## 6.动态链接的步骤和实现
动态链接分为3步：1.启动动态链接器本身；2.装载所有需要的共享对象；3.重定位和初始化

## 7.显式运行时链接
支持动态链接的系统一般都支持一种更灵活的模块加载方式**显式运行时链接**或称为**运行时加载**。就是让程序在运行时控制加载指定的模块，并在不需要改模块时将其卸载。

使用运行时加载可是程序的模块组织变得很灵活。程序用到某个插件或驱动时，才将相应的模块这种进来，而不需要在一开始就把他们全部装载进来，减少了程序启动时间和内存使用。此外，程序必须重启实现对模型的更替，对于需要长期运行的程序来说是很大的优势。

动态库的装载时通过一系列由动态链接器提供的API负责装载和链接的。

**dlopen()** 用来打开一个动态库，并将其装载到进程的地址空间，完成初始化过程。
```
void * dlopen(const char *filename, int flag);
```
**dlsym()** 可通过这个函数找到所需要的符号
```
void * dlsym(void *handle, char *symbol)
```
**dlerror()** 可以通过调用该函数来判断上一次的调用是否成功。             
**dlclose()** 卸载一个已被加载的模块。卸载过程写执行“.finti”段的代码，然后将相应的符号从符号表中去除，取消进程空间跟模块的映射关系，然后关闭模块文件。

一个简单的实例，将数学库模块用运行时加载的方法加载到进程中，然后获取sin()函数符号的地址，调用sin()并且返回结果。
```
#include <stdio.h>
#include <dlfcn.h>

int main(int argc, char* argv[])
{
     void* handle;
     double (*func)(double);
     char* error;

     handle = dlopen(argv[1], RTLD_NOW);
     if(handle == NULL){
          printf("Open library %s error：%s\n", argv[1], dlerror());
          return -1;
     }

     func = dlsym(handle, "sin");
     if( (error = dlerror()) != NULL){
          printf("Symbol sin not found: %s\n", error);
          goto exit_runso;
     }

     printf("%f\n", func(3.1415926 / 2));

     exit_runso:
     dlclose(handle);
}


$ gcc -o RunSoSimple RunSoSimple.c -ldl
$ ./RunSoSimple /lib/x86_64-linux-gnu/libm-2.23.so 
1.000000
```
> -ldl表示使用DL库



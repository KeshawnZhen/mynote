# 内存

## 1.程序内存布局 
应用程序中的内存空间分为内核空间和用户空间。32位系统中，Linux中将高地址的1GB分配给内核，windows将高地址中的2GB分配给内核。用户空间中也分为若干个区间。如图所示。         

![Linux进程地址空间布局](https://github.com/KeshawnZhen/mynote/blob/master/compile-linker/pic/linux_process_address_space_layout.jpg)

- 栈：栈用于维护函数调用的上下文。栈通常在用户空间的高处分配，有数兆字节大小。
- 堆：堆是用来容纳应用程序动态分配的内存区域，当程序使用malloc或new分配内存时，得到的内存来自堆里。堆通常位于栈的下方。堆一般比栈大很多，可以有几十到数百兆字节的容量。
- 可执行文件镜像：存储着可执行文件在内存里的映像。由装载器在装载时将可执行文件的内存读取或映射到这里。
- 保留区：是对内存中受到保护而禁止访问的内存区域的总称。
- 动态链接库映射区：用于映射装载的动态链接库。在Linux下，如果可执行文件依赖其他共享库，那么系统会为它从0X40000000开始的地址分配相应的空间，并将共享库载入到该空间。

## 2.栈
栈保存了一个函数调用所需要的维护信息。常常其被称为堆栈帧或活动记录。其一般包含以下内容：
- 函数的返回地址和参数
- 临时变量：包括函数的非静态局部变量已经编译器自动生成的其他临时变量
- 保存的上下文：包括在函数调用前后需要保持不变的寄存器

**调用惯例**         
函数的调用方和被调用方对于函数如何调用需要有一个明确的约定。只有双方都遵守同样的约定，函数才能被正确调用。这样的约定就称为调用惯例。

调用惯例一般规定如下几个方面的内容。       

1. 函数参数传递顺序和方式

    函数的参数常见地是通过栈传递。函数调用方将栈压入栈中，函数自己再从栈中取出。对于多个参数的函数，调用惯例要规定函数调用方将参数压栈的顺序。有些调用惯例还允许通过寄存器传递参数，以提高性能。
2. 栈的维护方式
    函数将参数压栈之后，函数体被调用。此后需要将压入栈的参数全部弹出，以使得栈在函数调用前后保持一致。弹出的工作，可有函数调用方或者函数本身来完成。
3. 名字修饰的策略
    为了链接的时候对调用惯例进行区分，调用管理要对函数本身的名字进行修饰。

**函数返回值**           
对于返回值本身是低于4个字节的，通过eax寄存器传递。如果返回值在4-8个字节的，通过eax和edx联合返回。对于超过8个字节的，会使用一个临时的栈上内存区域作为中转，结果返回值对象会被拷贝两次。

## 3.堆和内存管理
栈上的数据在函数返回时就会被释放掉，所以无法将数据传递至函数外部。而全局变量没有办法动态地产生，只能在编译的时候被定义，所以无法将数据传递至函数外部。使用堆可以改善这类情况。       
### Linux进程堆管理
Linux提供两种堆空间分配方式：一个brk()系统调用，而是mmap()系统调用。  
```
int brk(void* end_dat_segment)
```       
brk()实际是设置进程数据段的结束地址，即它可以扩大或缩小数据段。mmap()是向操作系统申请一段虚拟地址空间，这块虚拟地址可以映射到某个文件，当它不将地址空间映射到某个文件时，称这块空间为匿名空间，匿名空间就可以拿来作为堆空间。
```
void *mmap(
    void *start,        //申请空间的起始地址
    size_t length,      //申请空间的长度
    int prot,           //申请空间的权限（rwx）
    int flags,          //映射类型（文件，匿名空间）
    int fd,             //映射为文件时使用的文件描述符
    off_t offset        //映射为文件时使用的文件偏移
);
```
glibc中的mmap函数对于小于128KB的请求来说，会在现有的堆空间里面安装堆分配算法为它分配一块空间并返回。对于大于128KB的请求，mmap会为它分配一块匿名空间，然后在这个匿名空间中为用户分配空间。可用mmap轻易地实现malloc函数
```
void *malloc(size_t type)
{
    void *ret = mmap(0, nbytes, PORT_READ |PORT_WRITE,
        MAP_PRIVATE | MAP_ANONYMOUS, 0, 0);
    if(ret == MAP_FAILED)
        return 0;
    return ret;
}
```
mmap申请的空间的起始地址和大小必须是系统页的大小的整数倍。

### windows进程堆管理
TODO

## 堆分配算法
堆分配算法用来管理一大块连续的内存空间，做到按需分配、释放空间。

介绍两种较为简单的堆分配算法

**1.空闲链表**               
把堆中各个空闲的块按照链表的方式连接起来，当用户请求一块空间时，可以遍历整个链表，直到找到合适大小的块并且将它拆分。当用户释放空间时，将它合并到空闲链表中             
![空闲链表分配](https://github.com/KeshawnZhen/mynote/blob/master/compile-linker/pic/free_list_allocation.jpg)            
在空闲链表中分配空间过程如下：      
在空闲链表里查找足够能够容纳请求大小的一个空闲块，然后将这个块分为两个部分，一部分为程序请求的空间，另一部分为剩余下来的空闲空间。将链表里面对应原来空闲块的结构更新为新的剩下的空闲块，如果剩下的空闲块大小为0，则直接将这个结构从链表中删除。            
![空闲链表分配](https://github.com/KeshawnZhen/mynote/blob/master/compile-linker/pic/free_list_allocation_2.jpg)             
 
 这样分配带来的问题是，释放空间时，不知道对应块的大小。一种解决办法是分配n+4个字节，额外的4个字节用来存储块的大小。但是一旦链表结构被破坏或者记录长度的4个字节被破坏，整个堆就无法工作。

**2.位图**             
位图的核心思想是将整个堆划分为大量的块，每个块的大小相同。当用户请求内存的时候，总是分配整数个块的空间给用户。第一个块称为已分配区的头，其余的称为已分配区的主体。使用一个整数数组来记录块的使用情况。每个块有头/主体/空闲三种状态，使用2位就可表示一个块的状态。因此称为位图。         

这样实现方式的优点有：             
- 速度快：堆的空闲信息存储在一个数组内，访问该数组时cache容易命中
- 稳定性好：即使部分数据被破坏，堆仍可继续工作
- 快不需要额外信息，易于管理

这样实现方式的缺点有：           
- 分配内存时易产生碎片
- 若堆很大，或者一个块很小，那么位图就很大。可是失去cache命中率高的优势（可以使用多级位图类解决）


**3.对象池**                  
对象池的思路是，如果每一次分配的空间大小都一样，那么就可以按照这个每次请求分配的大小作为一个单位，把整个堆空间划分为大量的小块，每次请求时只需要找到一个小块就可以了。    

对象池的管理方法可以采用空闲链表，也可以采用位图。    

实际上，堆的分配算法是采取多种算法复合而成的。对于glibc，其小于64字节的空间申请是采用类似于对象池的方法；而对于大于512字节的空间申请采用的是最佳适配算法；对于大于128Kb的申请，使用mmap机制直接向操作系统申请空间。








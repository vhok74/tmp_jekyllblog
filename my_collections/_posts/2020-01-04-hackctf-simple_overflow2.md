---
layout: post
title:  "HackCTF Simple Overflow ver2 write-up"
date:   2020-01-04 19:45:55
image:  hackctf_simple2.PNG
tags:   [HackCTF]
categories: [Write-up]
---

### 1.  문제

7번 Simple_overflow_ver_2 문제에 걸린 보호기법 및 프로그램을 실행시키면 다음과 같다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Simple_Overflow_ver_2/Untitled.png)

이번에도 보호기법은 딱히 걸린게 없다
<br>  
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Simple_Overflow_ver_2/Untitled%201.png)

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Simple_Overflow_ver_2/Untitled%202.png)
<br><br>  
**프로그램을 실행시키면 다음과 같다.**

1. Data 입력을 받는다
2. 해당 버퍼의 주소로 보이는 주소가 나온다
3. 다시 실행을 할 것인지 묻는다
4. y를 누르면 다시 입력을 받게되고, 동일한 버퍼의 주소가 나온다
5. 프로그램을 종료했다가 다시 실행시키면 버퍼의 주소가 다르게 나온다.

6번 문제와 동일한 방법으로 풀면 될 것같다

<br><br>
### 2. 접근방법

아이다로 코드를 확인해보자

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Simple_Overflow_ver_2/Untitled%203.png)

사실 15-24 라인까지는 버퍼의 주소를 출력해주는 부분이고, 29라인은 Again 문자열 출력시 y를 눌르냐 마느냐 에 해당하는 부분이고, scanf를 이용하여 bof를 일으키고, ret 주소를 buf의 시작주소로 변경하는 방식으로 접근하는 6번 문제와 동일하다  
<br><br>


### 3. 풀이

버퍼의 시작주소를 바로 얻기 위해서 코드를 다음과 같이 작성하였다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Simple_Overflow_ver_2/Untitled%204.png)

1. 5라인 : recvuntil 을 이용하여 "Data : " 문자열까지 받는다
2. 6라인 : 버퍼의 주소를 얻기 위해 아무 문자열이나 보낸다
3. 7라인 : 버퍼의 주소를 얻은 뒤 buf_addr에 16진수로 형변환을 해서 저장한다
4. 8라인 : Again 문자열 까지 받는다
5. 9라인 : y를 누른다 (프로그램을 종료하지 않고 y를 누르면 버퍼의 주소는 고정됨)
6. 10라인 : 다시 recvuntil을 이용하여 Data : 문자열을 받는다
7. 12~14 : 26바이트 쉘코드 + 쓰레기값114바이트 + 3번에서 얻은 버퍼의 시작주소를 payload에 담는다  
<br><br>
### 4. 몰랐던 개념

6번과 동일하게 쓰레기값과 쉘코드의 순서를 변경하면 왜 안되는지..
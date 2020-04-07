---
layout: post
title:  "HacCTF x64 Simple size BOF write-up"
date:   2020-01-05 19:45:55
image:  hackctf_x64simple.PNG
tags:   [Hackctf]
categories: [Write-up]
---


### 1.  문제

6번 Simple_size_bof 문제에 걸린 보호기법 및 프로그램을 실행시키면 다음과 같다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20x64%20Simple_size_BOF/Untitled.png)

보호기법은 딱히 걸린 것이 없다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20x64%20Simple_size_BOF/Untitled%201.png)

프로그램을 실행시키면 다음과 같이 나오는데 자세히 보면 buf의 주소로 보이는 것이 출력된다.

하지만, 이 버퍼의 주소는 매번 달라지는 것으로 보아 ASLR 같은것이 걸려져 있는 것으로 보인다.

### 2. 접근방법

우선 아이다로 코드를 확인해 보자.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20x64%20Simple_size_BOF/Untitled%202.png)

v4 변수가 rbp 기준으로 0x6d30h 만큼 떨어진 곳에 선언이 되어 있다.

그렇다면 8번째 라인에서 gets() 함수를 이용하여 0x6d30h + 8 만큼 값을 채워 ret 값을 변경할 수 있을 것으로 보인다.

그렇다면 쓰레기값 + ret(버퍼 시작주소) 로 채우면 되는데 버퍼의 시작 주소가 매번 바뀌기 때문에 이를 고려해야한다.

### 3. 풀이

버퍼의 시작주소를 바로 얻기 위해서 코드를 다음과 같이 작성하였다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20x64%20Simple_size_BOF/Untitled%203.png)

1. recvuntil 을 이용하여 "buf: " 문자열까지 받는다
2. 그다음 나오는 주소의 길이가 14바이트이므로, recv를 이용하여 14바이트 만큼 받은 뒤 16진수 형태로 형변환을 해준다
3. 23바이트 쉘코드 + "\x90"*27937 + 2번에서 받은 버퍼 주소를 넣는다

### 4. 몰랐던 개념

어려운 개념은 없었는데, "\x90"*27937을 먼져 넣고, 쉘코드를 집어넣으면 안된다.. 왜그런지 알아봐야겠다..
---
layout: post
title:  "HacCTF 내 버퍼가 흘러넘친다!! write-up"
date:   2020-01-04 19:45:55
image:  hackctf_mybuf.PNG
tags:   [HackCTF]
categories: [Write-up]
---

### 1.  문제

4번 문제인 prob을 먼저 실행 시켜보면 다음과 같다

!![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled.png)

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%201.png)

이것만 봐서는 어떠한 동작을 하는 코드인지 모르기 때문에 아이다를 이용하여 바이너리를 뜯어보았다.
<br>  
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%202.png)

9번쨰 라인 gets 함수에서 bof가 일어날 수 있을 것같은데, 7번째 read 함수를 보면 name 변수에 값을 넣는 것을 알 수 있다. 

헌데 name 변수는 지역변수에 선언이 되어 있지 않다. 그렇다면 전역변수이지 않을 까 싶어서 확인을 해보았다.
<br>  
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%203.png)

bss 영역에 name이 public으로 선언되어 있는 것을 확인 할 수 있다. 따라서 전역변수로 해당 변수가 선언 되어 있는 것을 확인 할 수 있고, aslr이나 nx 비트가 걸려있지 않으므로 이부분을 이용하면 될 것으로 보인다.


<br><br>
### 2. 접근방법

name 변수에 0x32(50d) 크기만큼 삽입이 가능하므로 50바이트 이하 크기의 쉘코드를 삽입하고, gets의 bof를 이용하여 ret 주소를 name 변수 시작주소로 변경하면 되는 간단한 문제이다
<br><br>  
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%204.png)

메모리 구조가 다음과 같이 되어있으므로, gets를 이요하여 20(0x14)+4 = 24바이트 만큼 아무 값이나 넣고, 그 이후 name 변수의 주소 4바이트를 박으면 된다


<br><br><br> 
### 3. 풀이

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%205.png)

아까 사진을 다시 보자. "AAAA %x %x %x %x %x %x %x" 이렇게 입력하면, AAAA 0 41414141 ...~~ 이렇게 나오는 것을 확인 할 수 있다. 

2번째 포맷스트링부터 입력한 AAAA가 들어간 것을 알 수 있고 따라서 2번째 부분에서 %n을 이용하면 된다.

정리를 하자면 printf got 주소 + %"flag 함수 주소크기"x%n

이런식으로 입력을 하면 되고, 이 입력의 의미는 flag 함수 주소 크기+4 만큼 해당하는 값을 printf got 주소에 넣어라! 라는 의미가 된다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF/Untitled%206.png)


<br><br>
### 4. 몰랐던 개념

코드가 메모리 상에 올라가는 구조가 순간 헷갈렸다. 전에 포스팅 했던 자료를 이용하여 복습이 필요하다.  
<br>  
- **포스팅 자료**

[인텔 8086 메모리 구조](https://wogh8732.tistory.com/87?category=711515)
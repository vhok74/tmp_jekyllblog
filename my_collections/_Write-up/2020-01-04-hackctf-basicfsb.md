---
layout: post
title:  "HackCTF basic_fsb write-up"
date:   2020-01-04 19:45:55
image:  hackctf_basic_fsb.PNG
tags:   [HackCTF]
categories: [Write-up]
---

### 1.  문제

3번 문제인 basic_fsb를 먼저 실행 시켜보면 다음과 같다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Basic_FSB/Untitled.png)

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Basic_FSB/Untitled%201.png)

문제가 포맷 스트링 인 만큼 위와 같이 입력을 해보았다. fsb 확실.. 

이것만 봐서는 어떠한 동작을 하는 코드인지 모르기 때문에 아이다를 이용하여 바이너리를 뜯어보았다.
<br><br>
- main 문  
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Basic_FSB/Untitled%202.png)
<br>
- vuln() 함수  
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Basic_FSB/Untitled%203.png)

<br>
메인에는 별 다른 내용이 없다. 하지만 vuln() 함수가 있어서 해당 부분을 확인해 보았다. snprintf와 printf 함수에서 서식문자를 지정해주지 않고 format 변수에 바로 때려박는 것을 확인 할 수 있다.  

<br><br>  
### 2. 접근방법

fgets 함수에서 s 변수에 값을 넣고, snprintf 함수를 이용하여 s에 저장된 값을 format에 집어 넣는다. 9번째 라인의 printf 함수에서 포맷 스트링 공격이 가능 하다는 것이 확인된다. 또한 got 오버라이트를 통해 printf가 아닌 다른 함수의 실행이 가능하다.

그렇다면 시스템 함수를 실행시킬 수 있는 다른 함수가 있는지 찾아보았다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Basic_FSB/Untitled%204.png)

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Basic_FSB/Untitled%205.png)

flag라는 함수를 확인 할 수 있었고, 해당 함수에서 시스템 함수를 실행 시킬수 있다는 것을 확인 할 수 있다.

<br>

### 3. 풀이

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Basic_FSB/Untitled%201.png)

아까 사진을 다시 보자. "AAAA %x %x %x %x %x %x %x" 이렇게 입력하면, AAAA 0 41414141 ...~~ 이렇게 나오는 것을 확인 할 수 있다. 

2번째 포맷스트링부터 입력한 AAAA가 들어간 것을 알 수 있고 따라서 2번째 부분에서 %n을 이용하면 된다.

정리를 하자면 printf got 주소 + %"flag 함수 주소크기"x%n

이런식으로 입력을 하면 되고, 이 입력의 의미는 flag 함수 주소 크기+4 만큼 해당하는 값을 printf got 주소에 넣어라! 라는 의미가 된다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Basic_FSB/Untitled%206.png)

<br><br>
### 4. 몰랐던 개념

포맷 스트링은 아직 완벽한 이해가 되지 않은 것 같다. 현우나 영은이형한테 물어봐서 완벽하게 이해하는 것이 필요하다
---
layout: post
title:  "HackCTF x64 Buffer Overflow write-up"
date:   2020-01-04 19:45:55
image:  hackctf_x64_buf.PNG
tags:   [HackCTF]
categories: [Write-up]
---

### 1.  문제

5번 문제인 64bof_basic을 먼저 실행 시켜보면 다음과 같다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20x64%20Buffer%20Overflow/Untitled.png)

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20x64%20Buffer%20Overflow/Untitled%201.png)

해당 문제는 RELRO, NX 비트가 걸려있기 때문에 쉘코드의 실행이 불가하고, got 오버라이트도 불가하다. 그리고 64비트 elf 파일인 것을 알 수 있다.

이것만 봐서는 어떠한 동작을 하는 코드인지 모르기 때문에 아이다를 이용하여 바이너리를 뜯어보았다.
<br>  
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20x64%20Buffer%20Overflow/Untitled%202.png)

scanf를 이용하여 bof가 가능 할 것으로 보이는데, 보호기법때문에 got overwrite와 쉘코드 삽입은 불가하다. 

<br><br>
### 2. 접근방법

이 역시 주어진 함수가 있는지 확인해 보았다.  
<br> 
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20x64%20Buffer%20Overflow/Untitled%203.png)

해당 함수를 실행만 시키면, 쉘이 떨어질 것이다. 따라서 scanf를 이용하여 ret 주소를 callMeMaybe 함수의 주소로 바꿔주면 된다.

<br><br>
### 3. 풀이

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20x64%20Buffer%20Overflow/Untitled%202.png)  
<br> 
메모리 구조는 다음과 같다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20x64%20Buffer%20Overflow/Untitled%204.png)

sfp까지의 거리가 272(0x110)이므로 ret 까지의 거리는 +8 인 280바이트이다

<br>
- **callmeMaybe 함수 주소**  
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20x64%20Buffer%20Overflow/Untitled%205.png)

<br>

- **익스 코드**  
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20x64%20Buffer%20Overflow/Untitled%206.png)

<br><br>
### 4. 몰랐던 개념

이번 문제는 딱히 없었다.
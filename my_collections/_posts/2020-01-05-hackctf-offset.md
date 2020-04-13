---
layout: post
title:  "HackCTF offset write-up"
date:   2020-01-05 19:45:55
image:  hackctf_offset.PNG
tags:   [HackCTF]
categories: [Write-up]
---


### 1.  문제

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Offset/Untitled.png)

이번 문제는 스택 카나리 빼고 전부다 보호기법이 걸려있다

- **RELRO**

    Got Overwrite 불가

- **PIE, NX**

    Code 영역을 포함한 모든 영역이 랜덤하게 매핑되며 data, stack, heap에 실행권한이 없다  


<br>
코드는 다음과 같다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Offset/Untitled%201.png)

- 8번째 라인에서 어느함수를 실행시킬것인지 출력을한다
- 9번째 라인에서 gets를 이용하여 입력을 받고
- 10번째 라인에서 select_func함수를 call한다. 그때 입력한 문자열을 인자로 가지고 간다  

<br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Offset/Untitled%202.png)

select_func 함수는 다음과 같이 구성되어 있다

- char 변수 dest와 함수 포인터 v3가 선언 되어 있다
- two 라는 함수를 함수포인터 v3에 담는다
- 인자로 전달받은 src를 31바이트 만큼 dest 변수에 복사한다
- dest가 one과 같으면 v3에 one 함수를 집어넣는다
- return으로 v3() 함수를 호출한다


<br>
문제 흐름상 v3 함수 포인터에 정상 함수가 아닌 원하는 함수를 집어 넣어 변경시키면 될것으로 보인다

<br><br>

### 2. 접근방법

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Offset/Untitled%203.png)

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Offset/Untitled%204.png)

two와 one 함수는 다음과 같이 단순 문자열을 출력하는 함수이다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Offset/Untitled%205.png)

존재하는 함수를 확인해보면 수상한 함수가 하나 보인다.

print_flag 라는 함수인데, 누가봐도 이 함수를 호출하게 하면 될 것으로 보인다.  <br>
<br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Offset/Untitled%202.png)

다시 코드를 봐보쟈. 

strncpy 함수를 통해 src에 있는 값을 dest로 31바이트(0x1F)만큼 복사가 진행되는데   
<br><br>
현재 메모리 구조는 다음처럼 구성되어있다.

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Offset/Untitled%206.png)

dest 변수가 ebp-42 위치에 있고, v3 변수는 ebp-12 위치에 있다.

현재 두 변수 사이의 공간이 30바이트이므로, strncpy를 이용하여 31바이트 복사가 이루어 지기 때문에 1바이트 만큼 변조가 가능하다. 즉 다음과 같은 것이 가능하다  
<br><br>
![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Offset/Untitled%207.png)

헌데 자세히 보면 Two() 함수의 오프셋은 0x6AD이고, 우리가 변경하고자 하는 함수의 오프셋은 0x6D8이다

따라서  뒤에 \xAD를 \xD8로 변경해주면 된다

<br><br>

### 3. 풀이

우선 gdb로 내가 생각하는 위치에 올바르게 값이 들어가있는지 확인해보았다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Offset/Untitled%208.png)

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Offset/Untitled%209.png)  
<br>
입력한 명령어는 다음과 같다

1. gdb -q offset
2. b* main+92 → select_func 함수에 브레이크 포인트 걸기
3. r → 실행
4. aaaaaaaa 입력 → a8개
5. si로 select_func 함수로 들어가기
6. ni로 넘어가면서 아이다 코드에서 보면 select_func 마지막 라인으로 가기

    !![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Offset/Untitled%202.png)

    → 10번째 라인

call eax (0x56555600) 이 부분이 v3() 함수가 호출되는 부분이다. 

이 말은, v3 주소가 0x56555600 이므로 저 위치를 찾아서 원하는 값으로 덮으면 된다.

x/32x $esp를 입력하여 0x5655560 직전까지 30개의 더미 값이 필요한 것을 다시 한번 확인 하였다.

위 내용을 토대로 다음과 같은 코드를 작성 하였다

![]({{ site.baseurl }}/images/write-up/HackCTF/HackCTF%20Offset/Untitled%2010.png)


<br><br>
### 4. 몰랐던 개념

ez
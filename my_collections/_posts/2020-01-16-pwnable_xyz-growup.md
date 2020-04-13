---
layout: post
title:  "Pwnable.xyz Grow up write-up"
date:   2020-01-16 19:45:55
image:  pwnable_xyz_growup.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---

### 1.  문제

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20GrowUp/Untitled.png)

해당 문제는 PIE가 안걸려있다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20GrowUp/Untitled%201.png)

18상 이상이냐라는 문구가 나오고 이름을 입력할수 있는 공간이 나온다

코드를 한번 봐보쟈

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20GrowUp/Untitled%202.png)

- 13번째 라인에서 read를 통해 buf에 입력을 하고
- 입력한 값이 y 또는 Y 인지 한바이트 확인을 한다

- 그다음 0x84만큼 동적할당은 src에 하고 0x80 만큼 쳐 넣는다

- src에 0x80만큼 입력을 받고
- strcpy를 이용해서 usr에 복사한다 —>  (**굳이..? 수상함..)**
- 그다음 usr를 출력해주는 것 같다

    여기서 보면 첫번째 인자 부분에 서식문자들이 들어가야하는데 이상한 변수가 들어가 있다. 저건 setup 함수를 통해 지정되는 부분이다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20GrowUp/Untitled%203.png)

- setup 함수를 보면 다음과 같다
- 아까 printf 첫번째 인자가 보인다. 우선 여기에 &byte_601168의 주소를 qword_601160에 넣는다
- 그다음 차례로 %s\n를 넣는다
- 최종적으로 qword_601160 = (&byte_601168)%s\n 이렇게 printf 첫번째 인자로 들어갈 것이다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20GrowUp/Untitled%204.png)

- 전역변수, 즉 .data 영역을 보면 다음과 같은 수상한 값이 보인다
- 물론 이 flag는 가짜이다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20GrowUp/Untitled%205.png)

- usr는 0x6010E0 위치에 선언되어 있다
- 나머지 값들도 이렇게 들어가 있다
- 생각해보면 usr의 시작주소와 byte_601168 위치가 가깝기 때문에, 덮을수 있을것이다. 이거를 strcpy 함수와 엮어서 생각해보면 될것같다

<br><br>

### 2. 접근방법

위에서 말한대로 한번 생각을 해보자

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20GrowUp/Untitled%206.png)

이 부분은 사실 출력하는데 의미가 없는것이다. 서식문자는 %s 하나인데 저부분을 이용해야한다. 우선 동적할당 src를 봐보자. 사이즈는 0x84만큼 할당하는데 입력은 0x80만 받는다. gdb로 직접 0x80 즉 128바이트 만큼 a를 입력해보자

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20GrowUp/Untitled%207.png)

src는 총 132바이트 만큼 할당 했으므로 해당 구역은 zero filled 일 것이다. 그리고 128바이트 만큼 a를 입력했으므로 다음과 같이 들어갈 것이다

그다음 strcpy(usr,src)를 하는데 read 함수는 문자열의 끝을 나타내는 널문자를 삽입하지 않는다

그렇다면 src가 가리키는 문자열의 끝이 어딘지를 모르는데 위에 사진을 보면 132바이트 위치에 0, 즉 널포인트가 들어가 있다. 

따라서 저기를 문자열의 끝이라 인식하고 0x81, 즉 132바이트가 usr로 복사되어 처음 setup() 함수를 통해 다음과 같이 값이 들어가 있던것이

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20GrowUp/Untitled%208.png)

<br>

이렇게 변경된다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20GrowUp/Untitled%209.png)

<br>

따라서 아까 

- qword_601160 = (&byte_601168)%s\n 이 부분이
- qword_601160 = (&byte_601100)%s\n 이렇게 변경되는데

0x601100 위치는 usr 변수의 +32 위치이므로 변경이 가능하다

그렇다면 다음과 같이 128바이트를 구성해주면 

- 아무값*32 + "%p %p %p %p %p %p ..."  + 아무값*x = 128 바이트가 되도록 입력해주면

- qword_601160 = (&byte_601100)%s\n     → 이 의미가
- qword_601160 = "%p %p %p %p ...%s\n"  → 이렇게 된다

이 서식문자의 개수를 조정하면서 fsb를 이용한 특정주소의 leak이 가능하다

leak은 처음 read함수가 진행될때 0x16바이트 만큼 입력받고 앞의 1바이트만 y인지 아닌지 검사하므로

1바이트 y + 7바이트 아무값 + 아까 flag 주소 = 16바이트 이렇게 채우고 

pie가 꺼져있으므로 %p를 통해 아까 flag 주소가 나온 서식문자 개수를 확인하면 될 것이다

<br>

### 정리해보자

1. 처음 buf에 입력할 때 16바이트 입력이 가능하니 8바이트 얼라인에 따라서 y*8 입력 후 flag 주소를 넣자. fsb를 이용하여 이 주소에 들어있는 값을 확인해야 한다
2. 0x80 꽉채워서 동적할당 변수에 값을 넣자
3. strcpy가 진행되고
4. 마지막 printf 시 출력되는 플래그 주소를 확인하여 서식문자 개수를 알맞게 지정하여 %s로 flag에 담긴 내용을 확인해본다

<br><br>

### 3. 풀이

최종 익스 코드는 다음과 같다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20GrowUp/Untitled%2010.png)

<br><br>

### 4. 몰랐던 개념

1. **read 함수를 이용한 표준입력에는 문자열의 뒤에 널을 붙여주지 않는다**

2.  **strcpy는 문자열의 끝까지 복사를 진행한다**

3.  **pwntools에서 sendline 과 send는 다르다.** 

    - 전자는 개행을 포함해서 보내기때문에 입력한 문자열 +1만큼 크기가 전달된다

4.  **malloc을 하면 할당항 공간들이 zero filled 가 된다**

5.  **해당 익스 코드를 sendline으로 보내면 이유**

    - 3번에서 말했듯이, 개행이 포함되므로 16바이트만큼 입력하고 보내도 +1 개행이여서 17바이트가 보내진다
    - 그렇게 되면 입력버퍼에 개행 하나가 남게되고
    - 두번째 read시 개행을 읽어 씹히게 된다
    
        ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20GrowUp/Untitled%2011.png)<br><br>
    
        attach로 붙어서 확인해본 결과이다, 두번째 read함수가 끝난뒤 src 주소에 들어간 값을 확인했다.
    
        - 코드상에서는 128바이트 만큼 send를 했지만
        - 실제 들어간 값은 이전 read해서 보낸 개행하나이다
        - 따라서 sendline을 사용하지말고 send를 사용해서 코드를 짜야하는게 이러한 이유이다
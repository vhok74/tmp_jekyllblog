---
layout: post
title:  "Pwnable.xyz Note write-up"
date:   2020-01-17 19:45:55
image:  pwnable_xyz_note.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---

### 1.  문제

해당 문제는 RELRO와 PIE가 안걸려 있다. 따라서 GOT overwrite가 가능하다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20note/Untitled.png)

<br>

문제를 실행시키면 다음과 같은 메뉴가 나온다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20note/Untitled%201.png)

코드를 한번 봐보자

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20note/Untitled%202.png)

while 문으로 입력을 반복적으로 받을 수 있다. 

1을 입력하면 edit_note 함수가 실행될 것이고

2를 입력하면 edit_desc 함수가 실행될 것이다

read_int32함수는 입력을 받는 함수이다. 하나씩 확인해보자

<br>

1. **read_int32()**

    ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20note/Untitled%203.png)

    char형 변수에 0x20만큼 입력을 받고 해당 문자열을 atoi 함수를 통해 정수로 바꾼 값을 return 한다.<br><br>

2. **edit_note()**

    ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20note/Untitled%204.png)

    - read_int32 함수를 이용하여 입력을 받고 그 값만큼 동적할당을 한다
    - 그다음 해당 위치에 입력을, 설정한 크기만큼 받는다
    - 그거를 전역 변수 s에 입력 사이즈 만큼 복사한다
    - 해당 동적할당 주소를 free 시킨다
<br><br>
3. **edit_desc()**

    ![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20note/Untitled%205.png)

    - 전역변수 buf가 0이면 0x20만큼 동적할당을 한다
    - 해당 전역변수 buf에 0x20만큼 입력을 받는다

<br><br>

### 2. 접근방법

우선 note_edit 함수에서 strncpy를 이용하는 부분을 유심히 보자. 느낌적으로 저 부분일 듯싶다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20note/Untitled%206.png)

.bss 영역을 확인해보니 s와 buf 변수의 위치가 가깝다

s는 0x601480 주소를 시작으로 32바이트 만큼 배열에 입력을 받을 수 있는데, 그 다음 바로 buf 주소가 나온다

그렇다면 edit_note 함수에서 32+알파 만큼 값을 입력해준다면 s영역을 벗어나 buf 영역에 값이 덮어질 것이고,

edit_desc 함수에서 buf에 값이 들어가 있으니 동적할당을 하지 않고 바로 read함수가 호출될 것이다

<br>

### 그럼 이걸 어떻게??

생각해보자. 이전문제에서는 대부분 RELRO 가 다 걸려있었는데, 이번에는 안걸려 있다. 그말인 즉슨 GOT 섹션에 쓰기 권한이 있다는 소리이고 , 이걸 이용하여 GOT overwrite가 가능할 것이다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20note/Untitled%207.png)

실제 got 영역이 write가 가능하다

그렇다면 특정 함수의 got 값을 변경시켜 해당 함수가 아닌 win 함수로 변형시키면 될 것이다

정리하면 다음과 같다

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20note/Untitled%208.png)

- edit_note를 이용하여 s에 값을 넣을때 buf까지 침범하게 만든다
- 해당 buf에는 read 함수의 실제 got 주소를 넣어준다
- 그리고 2번을 입력하여 edit_desc가 실행되게 한다

- 현재 buf got 주소(0x1234)가 들어가 있기 때문에 read 함수가 실행되면 0x1234 주소에 값을 쓰게 된다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20note/Untitled%209.png)

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20note/Untitled%2010.png)

- 즉, buf에 win함수의 주소를 넣게 되면 read의 got 주소는 win 함수로 변경되어 다음번 read가 실행될시 win함수를 호출할 것이다

<br><br>

### 3. 풀이

최종 익스 코드는 다음과 같다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20note/Untitled%2011.png)

<br><br>

### 4. 몰랐던 개념

- plt, got 의 호출과정을 따로 정리할 필요가 있
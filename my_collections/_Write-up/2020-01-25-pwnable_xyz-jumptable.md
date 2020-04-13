---
layout: post
title:  "Pwnable.xyz Jump table write-up"
date:   2020-01-25 19:45:55
image:  pwnable_xyz_jumptable.PNG
tags:   [Pwnable_xyz]
categories: [Write-up]
---

### 1.  문제

 

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Jump%20table/Untitled.png)

PIE가 안걸려있다. got overwrite가 가능할 것같다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Jump%20table/Untitled%201.png)

프로그램을 실행시키면 다음과 같이 나온다. 딱봐도 동적할당에 대한 문제인듯 싶다. 할당, read,write, 해제에 대한 메뉴를 입력할 수 있다. 빠르게 코드를 봐보자

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Jump%20table/Untitled%202.png)

반복문을 돌면서 read_long() 함수를 통해 입력을 받는다. 입력받은 값은 v3 변수에 저장되어 5보다 작은 값인지를 확인하고, 조건에 맞는 사이즈면 vtable[] 의 인자에 담긴 값이 실행된다. 함수포인터 형식으로 저장되있는 것같은데 한번 확인해보자.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Jump%20table/Untitled%203.png)

우선 아이다로 확인해보았는데, vtable 기준으로 각 배열의 인덱스에 malloc, free, read, write 함수가 들어있다. 실제 그런지 확인해보자.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Jump%20table/Untitled%204.png)

예상한대로 vtable기준으로 8바이트 마다 해당 함수들의 주소가 들어가있다

나머지 malloc, free, read, write 함수들은 각각의 역할을 수행하는 함수들이다. 

<br><br>

### 2. 접근방법

while 문으로 malloc, free 등을 계속 진행 할 수 있기때문에 uaf나 double free 문제인듯 싶었다. 하지만 fd,bk 등을 조작하기 위해서는 힙 오버플로우 같은 것을 이용하여 덮어야 하는데, 사용자가 입력한 사이즈 만큼 데이터입력을 할 수 있으므로 불가능했다.

그렇게 고민하다가 다음부분을 발견했다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Jump%20table/Untitled%205.png)

아까 vtable의 영역 바로 위에 heap_buffer 변수와, 또 그 바로 위에 size 변수가 위치해 있다. heap_buffer 변수는 동적할당 한 힙 영역의 주소가 들어가있는 변수고, size는 사용자가 입력한 크기가 저장되는 공간이다. vtable 배열에서 만약 음수의 입력이 가능하다면, OOB가 가능하여 위의 위치에 값을 덮을 수 있을 것이다.

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Jump%20table/Untitled%206.png)

입력을 받는 read_long() 함수를 보면 다음과 같이 생겼다. [pwnable.xyz](http://pwnable.xyz) 문제들 대부분 사용자 입력을 이런식을 받기때문에 비슷하다. 8번라인은 결국 s 배열에 char 형태로 문자열을 저장시키고, strtoul 함수를 이용하여 unsigned long 형태로 반환해준다. 결국 음수를 입력하더라도 부호없는 형태로 반환해준다. 하지만

<br>

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Jump%20table/Untitled%207.png)

메인문에서 read_long 함수의 반환값을 v3에 받는데, 이 v3 변수는 int 형 변수이기 때문에 singed 형태로 인식될 것이다. 따라서 v3에 0xffff.... 이런식의 값이 들어가 있는데, 어셈블리어는 해당 최상위바이트를 부호비트로 판단하여 정상적인 음수값으로 비교를 진할 것이다.

따라서 결국 vtable의 - 0x10 위치의 size에 flag 함수의 주소를 (10진수로)입력해주면 끝이다. 배열 하나의 크기는 8바이트이므로, -2를 인덱스로 주면 vtable-0x10 의 값을 실행할 것이다.

<br><br>

### 3. 풀이

익스코드를 짤 필요도 없었다

![]({{ site.baseurl }}/images/write-up/pwnable_xyz/pwnable%20xyz%20Jump%20table/Untitled%208.png)

<br><br>

### 4. 몰랐던 개념

- 난이도에 비해 시간이 오래걸린 문제이다. 힙이 나와서 힙 취약점을 이용하는 건줄알아서 이상한 곳에 시간을 낭비했다.
- 앞으로 자신감을 가지고 침착하게 문제를 먼져 살펴봐야겠다